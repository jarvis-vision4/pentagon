# Pentagon - Technical Writing Platform (Production Soon)

A Django-based platform for technical writers to publish articles with tiered access (Free, Standard, Premium) and subscription monetization via PayPal.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Django 5.1.5 |
| Async/WSGI | Daphne (ASGI), Channels 4.x |
| Task Queue | Celery 5.4 + Redis |
| Database | SQLite (dev) / PostgreSQL (prod) |
| Cache | Redis (django-redis) |
| Frontend | Django Templates, Crispy Forms + DaisyUI, FontAwesome |
| Email | AWS SES (django-ses) |
| Payments | PayPal Subscriptions API |
| Auth | Django Auth + Email Verification + reCAPTCHA v3 |
| Security | django-csp, django-auto-logout, CSRF, XFrameOptions |

## Project Structure

```
pentagon/
├── account/          # User authentication, profiles, email verification
├── reader/           # Article reading, subscriptions, favorites, reviews
├── writer/           # Article creation, author dashboard, statistics
├── pentagon/         # Project settings, URLs, Celery, ASGI/WSGI
├── media/            # Uploaded images (articles, profiles)
├── templates/        # Base templates (if any)
└── manage.py
```

## Key Features

### User Roles & Ranks
- **Reader**: Browse free articles, subscribe to Standard/Premium
- **Writer**: Create free articles (all), Standard articles (Silver+), Premium articles (Platinum only)
- **Ranks**: Silver (default) → Gold (≥1 article, ≥1 avg views) → Platinum (≥2 articles, ≥2 avg views)
- **Admin**: Dashboard with platform statistics

### Article Tiers
| Tier | Access | Writer Requirement |
|------|--------|-------------------|
| Free | All users | Any authenticated user |
| Standard | Active Standard/Premium sub or Gold+ rank | Silver+ rank |
| Premium | Active Premium sub or Platinum rank | Platinum rank |

### Subscription System
- PayPal Subscriptions integration (Standard $4.99/mo, Premium $9.99/mo)
- Automatic expiration handling via Celery beat (runs every 60s)
- Plan upgrade/downgrade via PayPal API
- Webhook-ready for payment events

### Real-time Features
- WebSocket notifications when readers comment on author's articles
- Redis-backed channel layer (InMemory in dev)
- Per-author groups: `author_{user_id}`

### Content Features
- Categories: 10 tech categories (Django, React Native, AWS, etc.)
- Reading time estimation
- Article ratings & reviews with author replies
- Favorites/bookmarks
- View counting (session-based deduplication)
- Search with author/category filters
- Recent articles cache (Redis, 5 items, 24h TTL)

## Getting Started

### Prerequisites
- Python 3.11+
- Redis server (localhost:6379)
- PayPal Developer account (for subscriptions)
- AWS SES credentials (for email)

### Installation

```bash
# Clone and setup
cd pentagon
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Environment variables (create .env or export)
export SECRET_KEY='your-secret-key'
export DEBUG=True
export ALLOWED_HOSTS='localhost,127.0.0.1'
export RECAPTCHA_PUBLIC_KEY='your-recaptcha-public'
export RECAPTCHA_PRIVATE_KEY='your-recaptcha-private'
export AWS_SES_REGION_NAME='your-region'
export AWS_SES_REGION_ENDPOINT='email.your-region.amazonaws.com'
export DEFAULT_FROM_EMAIL='your-verified-ses-email'
export PAYPAL_CLIENT_ID='your-paypal-client-id'
export PAYPAL_CLIENT_SECRET='your-paypal-secret'
export PAYPAL_WEBHOOK_ID='your-webhook-id'  # for production

# Database
python manage.py migrate
python manage.py createsuperuser

# Collect static files
python manage.py collectstatic

# Run development servers (need 3 terminals)
# Terminal 1: Django + Daphne
python manage.py runserver

# Terminal 2: Celery worker
celery -A pentagon worker -l info

# Terminal 3: Celery beat (scheduled tasks)
celery -A pentagon beat -l info
```

### Access
- Home: http://127.0.0.1:8000/
- Admin: http://127.0.0.1:8000/admin/
- Writer Dashboard: http://127.0.0.1:8000/writer/dashboard/

## Configuration

### Key Settings (`pentagon/settings.py`)

```python
# Security
DEBUG = False  # Production
ALLOWED_HOSTS = ['yourdomain.com']
SECRET_KEY = env('SECRET_KEY')

# Database (Production)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': '5432',
    }
}

# Redis
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.RedisChannelLayer',
        'CONFIG': {'hosts': [('127.0.0.1', 6379)]},
    }
}
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'

# Email (AWS SES)
EMAIL_BACKEND = 'django_ses.SESBackend'
AWS_SES_REGION_NAME = 'ap-southeast-2'
DEFAULT_FROM_EMAIL = 'noreply@yourdomain.com'

# PayPal
PAYPAL_MODE = 'live'  # or 'sandbox'
PAYPAL_CLIENT_ID = env('PAYPAL_CLIENT_ID')
PAYPAL_CLIENT_SECRET = env('PAYPAL_CLIENT_SECRET')
```

### Celery Tasks

| Task | Schedule | Description |
|------|----------|-------------|
| `reader.tasks.delete_expired_subscriptions` | Every 60s | Removes expired subscriptions |
| `writer.tasks.update_user_rank` | Every 30s | Recalculates writer ranks based on views/articles |

## Deployment

### Production Checklist
- [ ] `DEBUG=False`
- [ ] `SECRET_KEY` from secure env
- [ ] PostgreSQL database configured
- [ ] Redis for cache + Celery + Channels
- [ ] AWS SES verified sender + production access
- [ ] PayPal Live credentials + webhook configured
- [ ] reCAPTCHA v3 keys for domain
- [ ] `collectstatic` run, static files served via CDN/Nginx
- [ ] Media files on S3 or persistent volume
- [ ] HTTPS + HSTS + CSP headers tuned
- [ ] Celery worker + beat as systemd services
- [ ] Daphne + Gunicorn workers behind Nginx
- [ ] Nginx WebSocket proxy config for `/ws/`
- [ ] Database backups scheduled

### Nginx WebSocket Config (example)
```nginx
location /ws/ {
    proxy_pass http://daphne_workers;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;
}
```

### Systemd Service (Celery Worker)
```ini
[Unit]
Description=Celery Worker for Pentagon
After=network.target redis.service

[Service]
Type=forking
User=www-data
WorkingDirectory=/opt/pentagon
EnvironmentFile=/opt/pentagon/.env
ExecStart=/opt/pentagon/venv/bin/celery -A pentagon worker -l info --detach
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
Restart=always

[Install]
WantedBy=multi-user.target
```

## API Endpoints Summary

### Account
| Method | Path | Description |
|--------|------|-------------|
| GET/POST | `/account/sign-up/` | Register + email verification |
| GET/POST | `/account/sign-in/` | Login (verified email required) |
| GET | `/account/sign-out/` | Logout |
| GET | `/account/verify-email/<uidb64>/<token>/` | Email verification link |
| GET | `/account/create-subscription/` | PayPal subscription callback |

### Writer
| Method | Path | Description |
|--------|------|-------------|
| GET | `/writer/dashboard/` | Author's articles list |
| GET/POST | `/writer/create-article/` | Create free article |
| GET/POST | `/writer/create-standard-article/` | Create Standard (Silver+) |
| GET/POST | `/writer/create-premium-article/` | Create Premium (Platinum) |
| GET/POST | `/writer/update-article/<id>/` | Edit own article |
| GET | `/writer/delete-article/<id>/` | Delete own article |
| GET | `/writer/admin-dashboard/` | Platform stats (staff) |
| GET | `/writer/statistics/` | Subscription revenue |
| GET | `/writer/profile/` | Writer profile + sub |
| GET | `/writer/check-comments/` | Reviews on author's articles |

### Reader
| Method | Path | Description |
|--------|------|-------------|
| GET | `/reader/` | Home - paginated articles |
| GET | `/reader/post-detail/<id>/` | Article detail + access control |
| GET | `/reader/profile/` | User profile, favorites, recent |
| GET | `/reader/standard-posts/` | Standard articles (sub required) |
| GET | `/reader/premium-posts/` | Premium articles (Premium sub) |
| GET | `/reader/subscription-posts-filter/` | Filtered standard |
| GET | `/reader/premium-subscription-posts/` | Filtered premium |
| GET | `/reader/search/?search=X` | Search articles |
| POST | `/reader/article-favorite/<id>/` | Add favorite |
| POST | `/reader/remove-favorite/<id>/` | Remove favorite |
| POST | `/reader/article-review/<id>/` | Submit review/rating |

## Extending the Platform

### Adding New Article Categories
Edit `writer/models.py` `CATEGORY_CHOICES` and run migrations.

### Custom Rank Thresholds
Modify `writer/tasks.py` `update_user_rank()` logic.

### Additional Payment Providers
Create new payment module in `reader/` following `paypal.py` pattern.

### WebSocket Event Types
Extend `reader/consumers.py` `send_notification` for new event types.

## License

MIT License - feel free to use for learning or commercial projects.

## Support

For issues or questions, check the [architecture documentation](ARCHITECTURE.md) or create an issue.
