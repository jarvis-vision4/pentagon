# Architecture Document - Pentagon Platform

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Browser                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP/WebSocket
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
┌──────────────────┐ ┌─────────────┐ ┌──────────────┐
│   Nginx/Proxy    │ │   Daphne    │ │  Static/Media│
│   (Port 80/443)  │ │  (Port 8001)│ │   Files      │
└────────┬─────────┘ └──────┬──────┘ └──────┬───────┘
         │                  │               │
         ▼                  ▼               ▼
┌────────────────────────────────────────────────────────────┐
│                    Django Application                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐ │
│  │ Account  │ │  Writer  │ │  Reader  │ │  Channels      │ │
│  │  App     │ │  App     │ │  App     │ │  Consumers     │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └───────┬────────┘ │
│       │            │            │                │          │
│       └────────────┴─────┬───────┴────────────────┘          │
│                          ▼                                    │
│              ┌─────────────────────┐                          │
│              │   Django ORM        │                          │
│              │   (Models)          │                          │
│              └──────────┬──────────┘                          │
│                         │                                      │
│         ┌───────────────┼───────────────┐                     │
│         ▼               ▼               ▼                     │
│  ┌────────────┐  ┌─────────────┐ ┌────────────┐              │
│  │  SQLite/   │  │   Redis     │ │   Celery   │              │
│  │  PostgreSQL│  │  (Cache/    │ │  (Worker/  │              │
│  │            │  │   Channel   │ │   Beat)    │              │
│  │            │  │   Layer)    │ │            │              │
│  └────────────┘  └─────────────┘ └────────────┘              │
└──────────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    ┌────────────┐ ┌──────────────┐ ┌──────────────┐
    │  AWS SES   │ │   PayPal     │ │  File Storage│
    │  (Email)   │ │  Subscriptions│ │  (Local/S3)  │
    └────────────┘ └──────────────┘ └──────────────┘
```

## Data Models & Relationships

```
┌─────────────────┐       ┌─────────────────┐
│   User (Django) │◄──────│  AccountStatus  │
│  (auth.User)    │  1:1  │  (profile, rank)│
└────────┬────────┘       └────────┬────────┘
         │                         │
    ┌────┴────┐             ┌──────┴──────┐
    ▼         ▼             ▼             ▼
┌────────┐ ┌────────┐  ┌───────────┐ ┌────────────┐
│Article │ │Favorite│  │Subscription│ │ArticleReview│
│        │ │        │  │            │ │             │
│author──┼─┤user    │  │user  1:1   │ │user         │
│        │ │article │  │            │ │article      │
└────┬───┘ └────────┘  └────────────┘ └──────┬─────┘
     │                                        │
     ▼                                        ▼
┌─────────┐                              ┌──────────┐
│Category │                              │Author    │
│(choices)│                              │Reply     │
└─────────┘                              └──────────┘
```

### Model Details

**AccountStatus** (account.models)
- `user` → User (FK, related_name=account_status)
- `is_verified` (Boolean)
- `profile` (ImageField)
- `rank` (CharField: Silver/Gold/Platinum)
- `created_at` (DateTime)

**Article** (writer.models)
- `author` → User (FK)
- `title`, `content` (min 40 chars)
- `category` (choices: 10 tech categories)
- `is_premium`, `is_standard` (Boolean, mutually exclusive)
- `view_count` (PositiveIntegerField)
- `photo` (ImageField)
- `posted_at` (DateTime)
- Methods: `reading_time()`, `get_rating()`, `get_author_name()`

**ArticleReview** (writer.models)
- `article` → Article (FK, related_name=article_review)
- `user` → User (FK, related_name=article_reviewer)
- `rating` (Float 0-5)
- `comment`, `author_reply` (TextField)
- `posted_at` (DateTime)

**Subscription** (reader.models)
- `user` → User (OneToOne, related_name=sub_user)
- `paypal_subscription_id` (unique)
- `subscription_plan` (Standard/Premium)
- `subscription_cost` (CharField)
- `is_active` (Boolean)
- `expires_at` (DateField)
- Methods: `remaining()` days left

**Favorite** (reader.models)
- `user` → User (FK)
- `article` → Article (FK)
- `created_at` (DateTime)

## Application Flow

### Article Access Control Flow

```
User requests article detail
         │
         ▼
┌────────────────────────┐
│  article.is_premium?   │──Yes──► User is author OR rank=Platinum? ──Yes──► Allow
└────────────────────────┘                │
         │ No                              No
         ▼                                 ▼
┌────────────────────────┐      ┌────────────────────────┐
│ article.is_standard?   │──Yes──► Active sub (Standard/  │──Yes──► Allow
└────────────────────────┘        Premium)?              │
         │ No                       │ No                  │
         ▼                          ▼                     ▼
    Allow (free)            Redirect to              Redirect to
                            subscription-locked      subscription-locked
```

### Writer Rank Calculation (Celery Task - every 30s)

```
For each user with AccountStatus:
    articles = Article.objects.filter(author=user)
    if articles.count() == 0: rank = Silver
    else:
        avg_views = Sum(view_count) / Count(articles)
        if articles.count() >= 2 and avg_views >= 2: rank = Platinum
        elif articles.count() >= 1 and avg_views >= 1: rank = Gold
        else: rank = Silver
    Update AccountStatus.rank
```

### Subscription Lifecycle

```
PayPal Checkout
      │
      ▼
User redirected to create_subscription view
      │
      ▼
Validate PayPal subscription details via API
      │
      ▼
Create Subscription record (is_active=True, expires_at=now+30d)
      │
      ▼
Celery Beat (every 60s): delete_expired_subscriptions
      │
      ├─ expires_at < today ──► is_active=False
      │
      └─ PayPal webhook: subscription cancelled ──► is_active=False
```

### Real-time Notification Flow

```
User posts review on Article
      │
      ▼
ArticleReview saved (writer/views.py)
      │
      ▼
Channel Layer: group_send("author_{author_id}", {...})
      │
      ▼
WebSocket Consumer (reader/consumers.py) receives event
      │
      ▼
Send JSON to connected WebSocket clients
      │
      ▼
Client JS updates notification badge/UI
```

## Key Components

### Celery Tasks

| Task | Schedule | Location |
|------|----------|----------|
| `delete_expired_subscriptions` | Every 60s | `reader.tasks` |
| `update_user_rank` | Every 30s | `writer.tasks` |

### WebSocket Consumer (reader/consumers.py)

```
WebSocketConsumer
  - connect(): join group "author_{user_id}"
  - disconnect(): leave group
  - receive(): handle incoming messages
  - send_notification(): send JSON to client
```

Group naming: `author_{author_id}` - allows targeting specific authors

### Cache Usage (Redis)

- `recent_articles:{user_id}` - LRU cache of 5 recent article IDs (TTL 24h)
- Used in `reader.views.log_recent_article()` / `get_recent_articles()`

### Email Flow

```
Registration → Generate token + uidb64
      │
      ▼
Send verification email (AWS SES) with link: /verify-email/{uidb64}/{token}/
      │
      ▼
User clicks link → verify_email view validates token
      │
      ▼
AccountStatus.is_verified = True → Redirect to success page
```

### PayPal Integration (reader/paypal.py)

Functions:
- `get_access_token()` - OAuth2 client credentials
- `get_subscription_details(access_token, sub_id)` - fetch plan details
- `cancel_subscription(access_token, sub_id)` - cancel via API
- `update_subscription_plan(...)` - change plan
- `update_current_subscription_plan(...)` - revise billing cycle

## Security Considerations

| Layer | Implementation |
|-------|----------------|
| Auth | Django auth + email verification required |
| Session | Auto-logout after 30min idle (django-auto-logout) |
| CSRF | Django CSRF middleware |
| reCAPTCHA | django-recaptcha on forms |
| CSP | django-csp headers |
| PayPal | Webhook signature verification |
| File Upload | ImageField with upload_to, default images |
| SQL Injection | Django ORM parameterized queries |

## Scalability Considerations

1. **Channel Layer**: InMemoryChannelLayer (dev) → RedisChannelLayer (prod)
2. **Database**: SQLite (dev) → PostgreSQL (prod)
3. **Cache**: Redis for both Django cache + Celery broker
4. **Static/Media**: Local → S3/CloudFront via django-storages
5. **Celery**: Single worker → Multiple workers with prefork/gevent
6. **ASGI**: Daphne → Daphne + Gunicorn workers or uvicorn

## API Endpoints Summary

### Account
- `GET/POST /account/sign-up/` - Registration
- `GET/POST /account/sign-in/` - Login
- `GET /account/sign-out/` - Logout
- `GET /account/verify-email/<uidb64>/<token>/` - Email verification
- `GET /account/create-subscription/?plan=X&token=Y` - PayPal callback

### Writer
- `GET /writer/dashboard/` - Author's articles
- `GET/POST /writer/create-article/` - Free article
- `GET/POST /writer/create-standard-article/` - Silver+ only
- `GET/POST /writer/create-premium-article/` - Platinum only
- `GET/POST /writer/update-article/<id>/` - Edit own article
- `GET /writer/delete-article/<id>/` - Delete own article
- `GET /writer/admin-dashboard/` - Admin stats (staff only)
- `GET /writer/statistics/` - Subscription revenue stats
- `GET /writer/profile/` - Writer profile + subscription
- `GET /writer/check-comments/` - Reviews on author's articles

### Reader
- `GET /reader/` - Home (paginated articles)
- `GET /reader/post-detail/<id>/` - Article detail + access control
- `GET /reader/profile/` - User profile, favorites, recent
- `GET /reader/standard-posts/` - Standard articles (sub required)
- `GET /reader/premium-posts/` - Premium articles (Premium sub required)
- `GET /reader/subscription-posts-filter/` - Filtered standard
- `GET /reader/premium-subscription-posts/` - Filtered premium
- `GET /reader/search/?search=X` - Search articles

## Deployment Checklist

- [ ] `DEBUG=False`
- [ ] `SECRET_KEY` from env
- [ ] `ALLOWED_HOSTS` configured
- [ ] PostgreSQL database
- [ ] Redis for cache + Celery + Channels
- [ ] AWS SES credentials
- [ ] PayPal live credentials + webhook URL
- [ ] reCAPTCHA v3 keys
- [ ] CSP headers tuned for PayPal popups
- [ ] Static files: `collectstatic` + CDN
- [ ] Media files: S3 or volume mount
- [ ] Celery worker + beat as services
- [ ] Daphne + Gunicorn via systemd/supervisor
- [ ] Nginx reverse proxy with WebSocket upgrade
- [ ] HTTPS + HSTS
- [ ] Backup strategy for DB + media