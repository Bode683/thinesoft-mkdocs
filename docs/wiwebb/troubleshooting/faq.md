# Frequently Asked Questions

Common questions about WiWebb and their answers.

## General Questions

??? question "What is WiWebb?"
    WiWebb is a full-stack multi-tenant network management platform designed for ISPs, enterprises, and network operators. It provides:

    - RADIUS authentication server
    - VPN management (Client and Site-to-Site)
    - IP Address Management (IPAM)
    - Certificate management
    - Subscription billing with Stripe
    - Multi-tenant organization management

    Built with Django (backend) and React (frontend), WiWebb offers a complete solution for managing network infrastructure and billing.

??? question "What does 'multi-tenant' mean?"
    Multi-tenancy means a single WiWebb installation can serve multiple independent organizations (tenants). Each tenant's data is completely isolated from other tenants, but they all share the same application infrastructure.

    **Benefits:**
    - Cost-effective for service providers
    - Easy to manage multiple customers
    - Automatic data isolation
    - Centralized administration

    See the [Multi-Tenancy Architecture](../architecture/multi-tenancy.md) for details.

??? question "Is WiWebb open source?"
    WiWebb is proprietary software developed by Thinesoft. Please contact sales@thinesoft.com for licensing information.

??? question "What are the system requirements?"
    **Minimum:**
    - 4GB RAM
    - 10GB disk space
    - Docker 20.10+
    - Docker Compose 2.0+

    **Recommended:**
    - 8GB+ RAM
    - 50GB+ disk space
    - SSD storage
    - Linux server (Ubuntu 20.04+)

    See the [Installation Guide](../getting-started/installation.md) for details.

## Installation & Setup

??? question "Do I need to know Docker?"
    Basic Docker knowledge is helpful but not required. Our [Installation Guide](../getting-started/installation.md) provides step-by-step instructions for installing and running WiWebb using Docker Compose.

    You'll mainly use these commands:
    ```bash
    docker compose up -d        # Start services
    docker compose down         # Stop services
    docker compose logs         # View logs
    docker compose ps           # Check status
    ```

??? question "Can I install WiWebb without Docker?"
    While possible, we strongly recommend using Docker for:
    - Consistent environment across all platforms
    - Easy dependency management
    - Simplified updates and rollbacks
    - Better isolation and security

    Manual installation would require setting up PostgreSQL, Redis, Python, Node.js, and all dependencies manually.

??? question "How do I update WiWebb?"
    ```bash
    # Pull latest code
    git pull origin main

    # Rebuild and restart containers
    docker compose down
    docker compose up --build -d

    # Run any new migrations
    docker compose exec django python manage.py migrate

    # Collect static files (production)
    docker compose exec django python manage.py collectstatic --noinput
    ```

??? question "How do I backup my data?"
    **Database backup:**
    ```bash
    # Backup
    docker compose exec django_db pg_dump -U django_user django_db > backup.sql

    # Restore
    docker compose exec -T django_db psql -U django_user django_db < backup.sql
    ```

    **Complete backup:**
    ```bash
    # Backup database, media files, and config
    docker compose exec django_db pg_dump -U django_user django_db | gzip > db_backup.sql.gz
    tar -czf media_backup.tar.gz mediafiles/
    cp .env.local env_backup
    ```

    **Automated backups:**
    Set up a cron job to run backups daily. See [Common Issues](common-issues.md#database-changes-not-persisting) for scripts.

## User Management

??? question "What are the different user roles?"
    WiWebb has four user roles with different permissions:

    | Role | Description | Permissions |
    |------|-------------|-------------|
    | **SuperAdmin** | System administrator | Full access to everything |
    | **Admin** | Multi-tenant admin | Manage all tenants and users (except SuperAdmin) |
    | **Tenant Owner** | Organization admin | Manage own tenant and its users |
    | **Subscriber** | End user | Use network services, manage own profile |

    See the [Roles & Permissions Guide](../user-guide/organizations/roles.md) for details.

??? question "How do I reset a user's password?"
    **As an admin:**
    ```bash
    # Via API
    curl -X POST http://localhost:8000/api/v1/users/{id}/set-password/ \
      -H "Authorization: Token {admin-token}" \
      -H "Content-Type: application/json" \
      -d '{"password": "NewPassword123!"}'

    # Via Django shell
    docker compose exec django python manage.py changepassword username
    ```

    **As a user (forgot password):**
    ```bash
    # Request reset email
    curl -X POST http://localhost:8000/api/v1/auth/password/reset/ \
      -H "Content-Type: application/json" \
      -d '{"email": "user@example.com"}'
    ```

??? question "Can users belong to multiple tenants?"
    No. Each user belongs to exactly one tenant (or no tenant for SuperAdmin/Admin). This ensures clear data isolation and simplifies permission management.

    If you need a user to access multiple tenants, create separate accounts for each tenant.

??? question "How do I deactivate a user without deleting them?"
    ```bash
    # Via API
    curl -X POST http://localhost:8000/api/v1/users/{id}/activate/ \
      -H "Authorization: Token {admin-token}" \
      -H "Content-Type: application/json" \
      -d '{"is_active": false}'
    ```

    Deactivated users:
    - Cannot log in
    - Retain all their data
    - Can be reactivated anytime
    - Still count toward tenant user quota

## Billing & Subscriptions

??? question "Do I need Stripe for development?"
    No! WiWebb includes **LocalStripe**, a mock Stripe server for development. It simulates Stripe's API without requiring a real Stripe account.

    ```bash
    # In .env.local
    USE_LOCALSTRIPE=True
    STRIPE_PUBLIC_KEY=pk_test_local
    STRIPE_SECRET_KEY=sk_test_local
    ```

    For production, you'll need a real Stripe account.

??? question "How do I set up Stripe for production?"
    1. Create a Stripe account at [stripe.com](https://stripe.com)
    2. Get your API keys from **Developers → API keys**
    3. Update `.env`:
       ```bash
       USE_LOCALSTRIPE=False
       STRIPE_PUBLIC_KEY=pk_live_your_public_key
       STRIPE_SECRET_KEY=sk_live_your_secret_key
       ```
    4. Set up webhook endpoint:
        - URL: `https://yourdomain.com/api/v1/payments/webhook/`
        - Events: `payment_intent.*`, `customer.subscription.*`
    5. Add webhook secret to `.env`:
       ```bash
       STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret
       ```

    See the [Configuration Guide](../getting-started/configuration.md#stripe-integration) for details.

??? question "Can users have free accounts?"
    Yes! Create a plan with `requires_payment=False`:

    ```python
    # Django shell
    from payment_gateway.models import Plan

    Plan.objects.create(
        name="Free",
        description="Free tier for individuals",
        requires_payment=False,
        daily_time_minutes=60,  # 1 hour/day
        daily_data_mb=100       # 100 MB/day
    )
    ```

??? question "How are usage limits enforced?"
    Usage limits are stored in the subscription plan:
    - `daily_time_minutes`: Maximum connection time per day
    - `daily_data_mb`: Maximum data transfer per day

    Your RADIUS/VPN servers should query WiWebb's API to check limits and enforce them:

    ```bash
    # Check user's current limits
    curl http://localhost:8000/api/v1/subscriptions/me/limits/ \
      -H "Authorization: Token {user-token}"
    ```

    `null` values mean unlimited.

??? question "What happens when a subscription expires?"
    When a subscription expires or payment fails:

    1. **Stripe sends webhook** to WiWebb
    2. **Subscription status** changes to `past_due` or `canceled`
    3. **User retains access** until current period ends
    4. **After grace period**, access is revoked
    5. **User downgrades** to free plan (if available)

    Users can resubscribe anytime to restore access.

## API & Integration

??? question "How do I get an API token?"
    **Via login:**
    ```bash
    curl -X POST http://localhost:8000/api/v1/auth/login/ \
      -H "Content-Type: application/json" \
      -d '{"username": "your-username", "password": "your-password"}'
    ```

    Response includes the token:
    ```json
    {
      "key": "9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b"
    }
    ```

    **Via Django shell (admin only):**
    ```python
    from rest_framework.authtoken.models import Token
    from api.models import User

    user = User.objects.get(username='admin')
    token, created = Token.objects.get_or_create(user=user)
    print(f"Token: {token.key}")
    ```

??? question "Are there rate limits on the API?"
    Yes, to prevent abuse:
    - **Login:** 5 attempts per minute per IP
    - **General API:** 100 requests per hour per user

    If you hit the limit, you'll receive:
    ```json
    {
      "detail": "Request was throttled. Expected available in 60 seconds."
    }
    ```

    Implement exponential backoff in your client. Contact support for higher limits.

??? question "Can I use the API from mobile apps?"
    Yes! WiWebb's REST API works with any HTTP client:

    - **React Native:** Use `fetch` or `axios`
    - **Flutter:** Use `http` package
    - **iOS (Swift):** Use `URLSession`
    - **Android (Kotlin):** Use `Retrofit` or `OkHttp`

    See the [API Reference](../api-reference/authentication.md) for examples.

??? question "Is there an SDK or client library?"
    Currently, WiWebb doesn't have official SDKs, but the API is RESTful and easy to integrate.

    Example client implementations are provided in the documentation:
    - [Python Client](../api-reference/authentication.md#python-client)
    - [TypeScript Client](../api-reference/authentication.md#typescript-client)

    You can use these as starting points for your integration.

??? question "Does WiWebb support GraphQL?"
    No, WiWebb currently only supports REST API. All endpoints follow RESTful conventions with JSON payloads.

## Security

??? question "Is WiWebb secure?"
    WiWebb implements multiple security layers:

    - **Authentication:** Token-based auth (not session-based)
    - **Authorization:** Role-based access control (RBAC)
    - **Data Isolation:** Tenant-scoped querysets
    - **Password Hashing:** PBKDF2 with SHA256
    - **HTTPS:** Enforced in production
    - **CSRF Protection:** Django middleware
    - **SQL Injection:** Prevented by Django ORM
    - **XSS Protection:** Auto-escaping in templates
    - **Audit Logging:** Sensitive actions logged

    See the [Authentication Architecture](../architecture/authentication.md) for details.

??? question "Where should I store API tokens?"
    **Web browsers:**
    - `localStorage` for web apps (accessible to JavaScript)
    - HttpOnly cookies for enhanced security (not accessible to JavaScript)

    **Mobile apps:**
    - iOS: Keychain Services
    - Android: EncryptedSharedPreferences
    - React Native: `expo-secure-store` or `react-native-keychain`

    **Never:**
    - Hard-code tokens in source code
    - Store in plain text files
    - Commit tokens to version control
    - Share tokens between users

??? question "How do I enable HTTPS?"
    For production deployment:

    1. **Get SSL certificate:**
        - Free: Let's Encrypt via Certbot
        - Paid: Purchase from certificate authority

    2. **Configure Nginx:**
       ```nginx
       server {
           listen 443 ssl;
           server_name yourdomain.com;

           ssl_certificate /path/to/cert.pem;
           ssl_certificate_key /path/to/key.pem;

           location / {
               proxy_pass http://frontend:5173;
           }

           location /api/ {
               proxy_pass http://django:8000;
           }
       }
       ```

    3. **Update Django settings:**
       ```bash
       # .env
       SECURE_SSL_REDIRECT=True
       SESSION_COOKIE_SECURE=True
       CSRF_COOKIE_SECURE=True
       ```

??? question "Can I enable two-factor authentication (2FA)?"
    2FA is not currently implemented in WiWebb but is planned for a future release. For now, encourage users to:
    - Use strong, unique passwords
    - Change passwords regularly
    - Avoid sharing accounts

## Performance

??? question "How many users can WiWebb handle?"
    WiWebb's capacity depends on your infrastructure:

    **Single server (4 CPU, 8GB RAM):**
    - ~1,000 concurrent users
    - ~10,000 total users

    **Scaled deployment:**
    - Load balancer + multiple app servers
    - Database read replicas
    - Redis cluster
    - Can handle 100,000+ users

    Bottlenecks are typically database queries and network I/O, not the application itself.

??? question "How can I improve performance?"
    **Database optimization:**
    ```bash
    # Ensure indexes exist
    docker compose exec django python manage.py migrate

    # Use connection pooling (production)
    # Add pgbouncer or use Django connection pooling
    ```

    **Caching:**
    ```python
    # Django settings
    CACHES = {
        'default': {
            'BACKEND': 'django_redis.cache.RedisCache',
            'LOCATION': 'redis://redis:6379/1',
        }
    }
    ```

    **Horizontal scaling:**
    - Multiple Django app servers behind load balancer
    - PostgreSQL read replicas
    - CDN for static assets

    See [Architecture Overview](../architecture/overview.md#scalability-design) for scaling strategies.

??? question "Why are API responses slow?"
    Common causes and solutions:

    1. **Missing indexes:** Run migrations
    2. **N+1 queries:** Use `select_related` / `prefetch_related`
    3. **Large result sets:** Use pagination
    4. **Cold start:** First request after restart is slower
    5. **Network latency:** Deploy closer to users

    Enable Django Debug Toolbar in development to identify slow queries.

## Customization

??? question "Can I customize the UI?"
    Yes! The frontend is built with React and Tailwind CSS.

    **Branding:**
    - Logo: Replace `public/logo.png`
    - Colors: Edit `tailwind.config.js`
    - Favicon: Replace `public/favicon.ico`

    **Components:**
    - Pages: `src/pages/`
    - Components: `src/components/`
    - Layouts: `src/layouts/`

    Rebuild after changes:
    ```bash
    docker compose up --build frontend
    ```

??? question "Can I add custom fields to models?"
    Yes, but requires creating Django migrations:

    ```python
    # backend/api/models.py
    class User(AbstractUser):
        # Add custom field
        department = models.CharField(max_length=100, blank=True)
    ```

    ```bash
    # Create and run migration
    docker compose exec django python manage.py makemigrations
    docker compose exec django python manage.py migrate
    ```

    Remember to update serializers and API documentation.

??? question "Can I integrate with my existing authentication system?"
    WiWebb supports token-based authentication. For integration:

    **Option 1: Use WiWebb as auth provider**
    - Other systems authenticate against WiWebb API
    - Single source of truth

    **Option 2: Sync users from external system**
    - Write script to create/update users via API
    - Run periodically (cron job)

    **Option 3: Custom authentication backend (advanced)**
    - Implement custom Django authentication backend
    - Authenticate against your LDAP/AD/OAuth provider

## Troubleshooting

??? question "Where can I find logs?"
    ```bash
    # All services
    docker compose logs

    # Specific service
    docker compose logs django
    docker compose logs frontend
    docker compose logs django_db

    # Follow logs in real-time
    docker compose logs -f django

    # Last 100 lines
    docker compose logs --tail=100 django
    ```

??? question "How do I debug issues?"
    1. **Check service status:**
       ```bash
       docker compose ps
       ```

    2. **View logs:**
       ```bash
       docker compose logs django
       ```

    3. **Access Django shell:**
       ```bash
       docker compose exec django python manage.py shell
       ```

    4. **Check database:**
       ```bash
       docker compose exec django python manage.py dbshell
       ```

    5. **Review [Common Issues](common-issues.md)**

??? question "The frontend shows a blank page. What do I do?"
    1. Check browser console (F12) for errors
    2. Verify frontend is running: `docker compose ps frontend`
    3. Check logs: `docker compose logs frontend`
    4. Rebuild: `docker compose up --build frontend`
    5. Hard refresh browser: Ctrl+Shift+R

    See [Common Issues - Frontend](common-issues.md#frontend-shows-blank-page) for more solutions.

## Support

??? question "Where can I get help?"
    - **Documentation:** You're reading it!
    - **Common Issues:** [Troubleshooting Guide](common-issues.md)
    - **Email:** support@thinesoft.com
    - **Bug Reports:** [GitHub Issues](https://github.com/itlds-cmr/labdemo/issues)

??? question "How do I report a bug?"
    When reporting bugs, please include:

    1. **Description:** What happened vs. what you expected
    2. **Steps to reproduce:** Detailed steps to trigger the bug
    3. **Environment:** OS, Docker version, WiWebb version
    4. **Logs:** Relevant log output
    5. **Screenshots:** If applicable

    Submit to: https://github.com/itlds-cmr/labdemo/issues

??? question "Can I request features?"
    Yes! Feature requests are welcome. Please:

    1. Check existing feature requests first
    2. Describe the use case clearly
    3. Explain why it's valuable
    4. Submit to GitHub Issues with "Feature Request" label

??? question "Is commercial support available?"
    Yes! Thinesoft offers:

    - **Technical support:** Priority email/chat support
    - **Consulting:** Architecture reviews, optimization
    - **Custom development:** Feature development, integrations
    - **Training:** On-site or remote training for teams
    - **SLA:** Service level agreements for enterprises

    Contact sales@thinesoft.com for pricing.

---

!!! tip "Didn't find your answer?"
    - Check [Common Issues](common-issues.md)
    - Search the documentation
    - Contact support@thinesoft.com
