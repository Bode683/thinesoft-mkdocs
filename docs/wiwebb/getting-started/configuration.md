# Configuration Guide

This guide covers all configuration options for WiWebb, from basic environment variables to advanced production settings.

## Configuration File

WiWebb uses environment files (`.env`) to manage configuration. The main configuration file should be named `.env.local` for development or `.env` for production.

!!! danger "Security Warning"
    **NEVER** commit `.env` files to version control!

    The `.env.example` file is provided as a template. Always create your own `.env.local` or `.env` file with secure values.

## Environment Variables Reference

### Django Core Settings

#### SECRET_KEY

**Required** | **Security Critical**

Django's secret key used for cryptographic signing.

```bash
SECRET_KEY=your-super-secret-key-here
```

!!! danger "Generate a Secure Key"
    **NEVER** use the example key in production!

    Generate a secure key:

    === "Python"
        ```python
        python3 -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
        ```

    === "OpenSSL"
        ```bash
        openssl rand -base64 50
        ```

    === "Online Generator"
        Use [Djecrety](https://djecrety.ir/) for a web-based generator

#### DEBUG

**Required** | Default: `False`

Enable Django debug mode.

```bash
DEBUG=True   # Development only!
DEBUG=False  # Production
```

!!! warning "Production Settings"
    **ALWAYS** set `DEBUG=False` in production!

    Debug mode exposes sensitive information and should never be enabled in production environments.

#### DOMAIN

**Required** | Default: `localhost`

The domain where WiWebb is hosted.

```bash
# Development
DOMAIN=localhost

# Production
DOMAIN=wiwebb.example.com
```

#### ALLOWED_HOSTS

**Optional** | Default: Derived from `DOMAIN`

Comma-separated list of allowed hosts.

```bash
# Development
ALLOWED_HOSTS=localhost,127.0.0.1

# Production
ALLOWED_HOSTS=wiwebb.example.com,www.wiwebb.example.com
```

#### SECURE_SSL_REDIRECT

**Optional** | Default: `False`

Redirect all HTTP requests to HTTPS.

```bash
SECURE_SSL_REDIRECT=False  # Development
SECURE_SSL_REDIRECT=True   # Production
```

!!! tip "HTTPS in Production"
    Always enable SSL redirect in production:

    ```bash
    SECURE_SSL_REDIRECT=True
    SECURE_HSTS_SECONDS=31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS=True
    SESSION_COOKIE_SECURE=True
    CSRF_COOKIE_SECURE=True
    ```

### Database Configuration

WiWebb uses PostgreSQL as its database backend.

#### POSTGRES_DB

**Required** | Default: `django_db`

The name of the PostgreSQL database.

```bash
POSTGRES_DB=django_db
```

#### POSTGRES_USER

**Required** | Default: `admin`

PostgreSQL username.

```bash
POSTGRES_USER=django_user
```

#### POSTGRES_PASSWORD

**Required** | **Security Critical**

PostgreSQL password.

```bash
POSTGRES_PASSWORD=your-secure-database-password
```

!!! danger "Use a Strong Password"
    Generate a strong password:

    ```bash
    openssl rand -base64 32
    ```

#### POSTGRES_HOST

**Required** | Default: `django_db`

PostgreSQL server hostname.

```bash
# Docker Compose (service name)
POSTGRES_HOST=django_db

# External database
POSTGRES_HOST=db.example.com
```

#### POSTGRES_PORT

**Optional** | Default: `5432`

PostgreSQL server port.

```bash
POSTGRES_PORT=5432
```

#### Connection String Format

Alternatively, use a connection string:

```bash
DATABASE_URL=postgresql://user:password@host:5432/database
```

### Superuser Configuration

These variables create an initial superuser account automatically.

#### DJANGO_SUPERUSER_USERNAME

**Required for auto-creation**

```bash
DJANGO_SUPERUSER_USERNAME=admin
```

#### DJANGO_SUPERUSER_EMAIL

**Required for auto-creation**

```bash
DJANGO_SUPERUSER_EMAIL=admin@example.com
```

#### DJANGO_SUPERUSER_PASSWORD

**Required for auto-creation** | **Security Critical**

```bash
DJANGO_SUPERUSER_PASSWORD=your-secure-admin-password
```

!!! warning "Change After First Login"
    Change the superuser password immediately after first login!

### Stripe Integration

WiWebb integrates with Stripe for payment processing.

#### USE_LOCALSTRIPE

**Optional** | Default: `True`

Use LocalStripe (mock Stripe server) for development.

```bash
# Development
USE_LOCALSTRIPE=True

# Production
USE_LOCALSTRIPE=False
```

#### STRIPE_PUBLIC_KEY

**Required for production**

Your Stripe publishable key.

```bash
# Development (with LocalStripe)
STRIPE_PUBLIC_KEY=pk_test_local

# Production
STRIPE_PUBLIC_KEY=pk_live_your_actual_public_key
```

!!! info "Getting Stripe Keys"
    1. Create an account at [stripe.com](https://stripe.com)
    2. Navigate to **Developers** → **API keys**
    3. Copy your publishable and secret keys

#### STRIPE_SECRET_KEY

**Required for production** | **Security Critical**

Your Stripe secret key.

```bash
# Development
STRIPE_SECRET_KEY=sk_test_local

# Production
STRIPE_SECRET_KEY=sk_live_your_actual_secret_key
```

!!! danger "Keep Secret Keys Private"
    **NEVER** expose your Stripe secret key in frontend code or public repositories!

#### STRIPE_WEBHOOK_SECRET

**Required for webhooks**

Stripe webhook signing secret for verifying webhook events.

```bash
# Development
STRIPE_WEBHOOK_SECRET=whsec_local

# Production
STRIPE_WEBHOOK_SECRET=whsec_your_actual_webhook_secret
```

??? example "Setting Up Stripe Webhooks"
    1. Go to **Developers** → **Webhooks** in Stripe Dashboard
    2. Click **Add endpoint**
    3. Enter your webhook URL: `https://yourdomain.com/api/v1/payments/webhook/`
    4. Select events to listen for:
        - `payment_intent.succeeded`
        - `payment_intent.payment_failed`
        - `customer.subscription.created`
        - `customer.subscription.updated`
        - `customer.subscription.deleted`
    5. Copy the webhook signing secret

### Frontend Configuration

#### VITE_API_URL

**Required**

The URL where the Django API is accessible.

```bash
# Development
VITE_API_URL=http://localhost:8000

# Production
VITE_API_URL=https://api.wiwebb.example.com
```

!!! warning "CORS Configuration"
    Ensure Django's CORS settings allow requests from your frontend domain.

### Redis Configuration

Redis is used for caching and Celery task queuing.

#### REDIS_HOST

**Optional** | Default: `redis`

```bash
REDIS_HOST=redis
```

#### REDIS_PORT

**Optional** | Default: `6379`

```bash
REDIS_PORT=6379
```

#### REDIS_URL

**Optional**

Full Redis connection URL:

```bash
REDIS_URL=redis://redis:6379/0
```

### Email Configuration

Configure email settings for sending notifications.

#### EMAIL_BACKEND

**Optional** | Default: `django.core.mail.backends.console.EmailBackend`

```bash
# Development (logs to console)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend

# Production (SMTP)
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
```

#### SMTP Settings

```bash
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
DEFAULT_FROM_EMAIL=noreply@wiwebb.example.com
```

??? example "Gmail Configuration"
    For Gmail, create an App Password:

    1. Go to Google Account settings
    2. Navigate to **Security** → **2-Step Verification**
    3. Scroll to **App passwords**
    4. Generate a password for "Mail"
    5. Use the generated password in `EMAIL_HOST_PASSWORD`

### Storage Configuration

#### DEFAULT_STORAGE_DSN

**Optional**

Configure file storage backend.

```bash
# Local filesystem
DEFAULT_STORAGE_DSN=file:///data/media/?url=%2Fmedia%2F

# AWS S3
DEFAULT_STORAGE_DSN=s3://bucket-name?region=us-east-1

# Azure Blob Storage
DEFAULT_STORAGE_DSN=azure://container-name
```

### Logging Configuration

#### LOG_LEVEL

**Optional** | Default: `INFO`

```bash
LOG_LEVEL=DEBUG    # Development
LOG_LEVEL=INFO     # Production
LOG_LEVEL=WARNING  # Minimal logging
```

## Complete Configuration Examples

### Development Configuration

```bash title=".env.local"
# Django Core
SECRET_KEY=dev-secret-key-only-for-local-development
DEBUG=True
DOMAIN=localhost
ALLOWED_HOSTS=localhost,127.0.0.1
SECURE_SSL_REDIRECT=False

# Database
POSTGRES_DB=django_db
POSTGRES_USER=django_user
POSTGRES_PASSWORD=localdev123
POSTGRES_HOST=django_db
POSTGRES_PORT=5432

# Superuser
DJANGO_SUPERUSER_USERNAME=admin
DJANGO_SUPERUSER_EMAIL=admin@localhost
DJANGO_SUPERUSER_PASSWORD=admin123

# Stripe (LocalStripe)
USE_LOCALSTRIPE=True
STRIPE_PUBLIC_KEY=pk_test_local
STRIPE_SECRET_KEY=sk_test_local
STRIPE_WEBHOOK_SECRET=whsec_local

# Frontend
VITE_API_URL=http://localhost:8000

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# Email (Console)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend

# Logging
LOG_LEVEL=DEBUG
```

### Production Configuration

```bash title=".env"
# Django Core
SECRET_KEY=<generate-secure-key>
DEBUG=False
DOMAIN=wiwebb.example.com
ALLOWED_HOSTS=wiwebb.example.com,api.wiwebb.example.com
SECURE_SSL_REDIRECT=True
SECURE_HSTS_SECONDS=31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS=True
SESSION_COOKIE_SECURE=True
CSRF_COOKIE_SECURE=True

# Database
POSTGRES_DB=wiwebb_production
POSTGRES_USER=wiwebb_user
POSTGRES_PASSWORD=<generate-secure-password>
POSTGRES_HOST=db.example.com
POSTGRES_PORT=5432

# Superuser (change password after first login!)
DJANGO_SUPERUSER_USERNAME=admin
DJANGO_SUPERUSER_EMAIL=admin@wiwebb.example.com
DJANGO_SUPERUSER_PASSWORD=<generate-secure-password>

# Stripe (Production)
USE_LOCALSTRIPE=False
STRIPE_PUBLIC_KEY=pk_live_<your-key>
STRIPE_SECRET_KEY=sk_live_<your-key>
STRIPE_WEBHOOK_SECRET=whsec_<your-secret>

# Frontend
VITE_API_URL=https://api.wiwebb.example.com

# Redis
REDIS_HOST=redis.example.com
REDIS_PORT=6379
REDIS_PASSWORD=<redis-password>

# Email (SMTP)
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.sendgrid.net
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=apikey
EMAIL_HOST_PASSWORD=<sendgrid-api-key>
DEFAULT_FROM_EMAIL=noreply@wiwebb.example.com

# Storage (S3)
DEFAULT_STORAGE_DSN=s3://wiwebb-media?region=us-east-1
AWS_ACCESS_KEY_ID=<aws-access-key>
AWS_SECRET_ACCESS_KEY=<aws-secret-key>

# Logging
LOG_LEVEL=INFO
SENTRY_DSN=https://<your-sentry-dsn>@sentry.io/<project>
```

## Security Checklist

Before deploying to production, verify:

!!! danger "Production Security Checklist"
    - [ ] `SECRET_KEY` is unique and randomly generated
    - [ ] `DEBUG=False`
    - [ ] `SECURE_SSL_REDIRECT=True`
    - [ ] Strong database password
    - [ ] Strong admin password (changed after first login)
    - [ ] Production Stripe keys configured
    - [ ] HTTPS certificate installed
    - [ ] `ALLOWED_HOSTS` properly configured
    - [ ] Email configured for notifications
    - [ ] Backups automated
    - [ ] Monitoring/logging configured (e.g., Sentry)

## Applying Configuration Changes

After modifying environment variables:

### 1. Restart Services

```bash
# Restart all services
docker compose down
docker compose up -d

# Or restart specific service
docker compose restart django
docker compose restart frontend
```

### 2. Rebuild if Needed

Some changes (like `VITE_API_URL`) require rebuilding:

```bash
docker compose up --build -d
```

### 3. Run Migrations

If database settings changed:

```bash
docker compose exec django python manage.py migrate
```

### 4. Collect Static Files

For production:

```bash
docker compose exec django python manage.py collectstatic --noinput
```

## Advanced Configuration

### Custom Django Settings

For advanced users, you can override Django settings by creating a custom settings file:

```python title="backend/backend/settings_custom.py"
from .settings import *

# Custom settings here
INSTALLED_APPS += ['your_custom_app']

# Override specific settings
SESSION_COOKIE_AGE = 86400  # 24 hours
```

Then set the Django settings module:

```bash
DJANGO_SETTINGS_MODULE=backend.settings_custom
```

### Multiple Environments

Manage multiple environments with different env files:

```bash
# Development
docker compose --env-file .env.local up -d

# Staging
docker compose --env-file .env.staging up -d

# Production
docker compose --env-file .env up -d
```

## Environment Variable Precedence

Variables are loaded in this order (later overrides earlier):

1. System environment variables
2. `.env` file
3. `.env.local` file (if specified with `--env-file`)
4. Docker Compose `environment` section

## Troubleshooting Configuration

??? warning "Changes Not Taking Effect"
    **Problem:** Modified environment variables but no changes observed.

    **Solution:**
    1. Ensure the correct `.env` file is being used
    2. Restart the affected services
    3. Rebuild if the variable is used at build time
    4. Check for typos in variable names

??? warning "Database Connection Failed"
    **Problem:** `OperationalError: could not connect to server`

    **Solution:**
    1. Verify `POSTGRES_HOST`, `POSTGRES_PORT` are correct
    2. Ensure `POSTGRES_USER` and `POSTGRES_PASSWORD` match
    3. Check PostgreSQL container is running: `docker compose ps`
    4. View PostgreSQL logs: `docker compose logs django_db`

??? warning "Stripe Payments Failing"
    **Problem:** Payment creation returns errors

    **Solution:**
    1. Verify Stripe keys are correct
    2. Check `USE_LOCALSTRIPE` setting matches your environment
    3. For production, ensure webhook endpoint is configured
    4. Check Stripe dashboard for error details

## Next Steps

With WiWebb properly configured, you're ready to:

- **[Explore the User Guide](../user-guide/organizations/tenants.md)** - Learn about WiWebb's features
- **[Read the Architecture docs](../architecture/overview.md)** - Understand how WiWebb works
- **[Use the API](../api-reference/authentication.md)** - Integrate with your applications

---

!!! question "Need Help?"
    - [Troubleshooting Guide](../troubleshooting/common-issues.md)
    - [FAQ](../troubleshooting/faq.md)
    - Support: support@thinesoft.com
