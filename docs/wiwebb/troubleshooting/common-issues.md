# Common Issues

This guide covers common issues you might encounter with WiWebb and how to resolve them.

## Installation Issues

### Docker Not Starting

??? warning "Error: Cannot connect to Docker daemon"
    **Symptoms:**
    ```
    Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
    Is the docker daemon running?
    ```

    **Cause:** Docker service is not running or user lacks permissions.

    **Solutions:**

    === "Linux"
        ```bash
        # Check if Docker is running
        sudo systemctl status docker

        # Start Docker
        sudo systemctl start docker

        # Enable Docker to start on boot
        sudo systemctl enable docker

        # Add user to docker group (logout/login required)
        sudo usermod -aG docker $USER
        newgrp docker
        ```

    === "macOS"
        ```bash
        # Start Docker Desktop from Applications
        open -a Docker

        # Or verify it's running
        docker --version
        ```

    === "Windows"
        ```powershell
        # Start Docker Desktop
        # Check system tray for Docker icon

        # Verify WSL2 is installed
        wsl --list --verbose
        ```

### Port Already in Use

??? warning "Error: Port is already allocated"
    **Symptoms:**
    ```
    Error starting userland proxy: listen tcp4 0.0.0.0:8000:
    bind: address already in use
    ```

    **Cause:** Another service is using port 8000, 5173, or 5432.

    **Solutions:**

    ```bash
    # Find what's using the port
    # Linux/macOS
    lsof -i :8000
    netstat -tulpn | grep 8000

    # Windows
    netstat -ano | findstr :8000

    # Option 1: Stop the conflicting service
    kill <PID>

    # Option 2: Change WiWebb ports in docker-compose.yml
    services:
      django:
        ports:
          - "8001:8000"  # Change 8000 to 8001
      frontend:
        ports:
          - "5174:5173"  # Change 5173 to 5174
    ```

### Database Connection Failed

??? warning "Error: could not connect to server"
    **Symptoms:**
    ```
    django.db.utils.OperationalError: could not connect to server:
    Connection refused
    ```

    **Cause:** PostgreSQL container not ready or connection settings incorrect.

    **Solutions:**

    ```bash
    # Check if database container is running
    docker compose ps django_db

    # View database logs
    docker compose logs django_db

    # Wait for database to initialize (first run takes longer)
    # Look for: "database system is ready to accept connections"

    # Verify environment variables
    cat .env.local | grep POSTGRES

    # Restart services
    docker compose restart django django_db

    # If still failing, recreate database
    docker compose down -v  # WARNING: Deletes data!
    docker compose up -d
    ```

### Build Fails with Permission Errors

??? warning "Error: Permission denied"
    **Symptoms:**
    ```
    ERROR: for django  Cannot start service django: OCI runtime create failed
    ```

    **Cause:** File permission issues or SELinux restrictions.

    **Solutions:**

    ```bash
    # Linux: Fix ownership
    sudo chown -R $USER:$USER .

    # Linux: SELinux (if enabled)
    sudo chcon -Rt svirt_sandbox_file_t .

    # Fix Docker socket permissions
    sudo chmod 666 /var/run/docker.sock

    # Or add user to docker group
    sudo usermod -aG docker $USER
    ```

## Authentication Issues

### 401 Unauthorized

??? warning "API returns 401 for authenticated requests"
    **Symptoms:**
    ```json
    {
      "detail": "Invalid token."
    }
    ```

    **Causes & Solutions:**

    **1. Token format incorrect:**
    ```bash
    # ❌ Wrong
    Authorization: abc123def456

    # ❌ Wrong (using Bearer)
    Authorization: Bearer abc123def456

    # ✅ Correct
    Authorization: Token abc123def456
    ```

    **2. Token expired or invalid:**
    ```bash
    # Login again to get new token
    curl -X POST http://localhost:8000/api/v1/auth/login/ \
      -H "Content-Type: application/json" \
      -d '{"username":"admin","password":"yourpassword"}'
    ```

    **3. User account deactivated:**
    ```bash
    # Check user status (admin)
    curl -X GET http://localhost:8000/api/v1/users/{id}/ \
      -H "Authorization: Token {admin-token}"

    # Reactivate if needed
    curl -X POST http://localhost:8000/api/v1/users/{id}/activate/ \
      -H "Authorization: Token {admin-token}" \
      -d '{"is_active": true}'
    ```

### 403 Forbidden

??? warning "Permission denied errors"
    **Symptoms:**
    ```json
    {
      "detail": "You do not have permission to perform this action."
    }
    ```

    **Causes & Solutions:**

    **1. Insufficient role permissions:**
    ```bash
    # Check your role
    curl -X GET http://localhost:8000/api/v1/auth/user/ \
      -H "Authorization: Token {your-token}"

    # Review permission matrix in docs
    # Subscribers cannot create tenants/users
    # Tenant Owners can only manage their tenant
    ```

    **2. Accessing resources outside your tenant:**
    ```bash
    # Tenant Owners can only access their own tenant's resources
    # Verify you're accessing the correct tenant ID

    # Get your tenant
    curl -X GET http://localhost:8000/api/v1/tenants/me/ \
      -H "Authorization: Token {your-token}"
    ```

### Login Fails with Correct Credentials

??? warning "Unable to log in with provided credentials"
    **Symptoms:**
    ```json
    {
      "non_field_errors": ["Unable to log in with provided credentials."]
    }
    ```

    **Causes & Solutions:**

    **1. Account deactivated:**
    ```bash
    # Admin needs to reactivate your account
    # Contact system administrator
    ```

    **2. Wrong username/password:**
    ```bash
    # Request password reset
    curl -X POST http://localhost:8000/api/v1/auth/password/reset/ \
      -H "Content-Type: application/json" \
      -d '{"email":"your@email.com"}'
    ```

    **3. Case sensitivity:**
    ```bash
    # Usernames are case-sensitive
    # "Admin" ≠ "admin"
    ```

## API Issues

### CORS Errors

??? warning "CORS policy blocked the request"
    **Symptoms:**
    ```
    Access to fetch at 'http://localhost:8000/api/v1/users/'
    from origin 'http://localhost:5173' has been blocked by CORS policy
    ```

    **Cause:** Frontend and backend on different origins.

    **Solutions:**

    ```python
    # backend/backend/settings.py
    CORS_ALLOWED_ORIGINS = [
        "http://localhost:5173",  # Frontend dev server
        "http://127.0.0.1:5173",
        # Add production domain
        "https://yourdomain.com",
    ]

    CORS_ALLOW_CREDENTIALS = True
    ```

    ```bash
    # Restart Django after changes
    docker compose restart django
    ```

### Network Errors

??? warning "Failed to fetch / Network request failed"
    **Symptoms:**
    - `TypeError: Failed to fetch`
    - `Network request failed`
    - `ERR_CONNECTION_REFUSED`

    **Causes & Solutions:**

    **1. Backend not running:**
    ```bash
    # Check if Django is running
    docker compose ps django

    # Check logs
    docker compose logs django

    # Restart if needed
    docker compose up -d django
    ```

    **2. Wrong API URL:**
    ```bash
    # Check VITE_API_URL in .env.local
    VITE_API_URL=http://localhost:8000

    # Rebuild frontend after changes
    docker compose up --build frontend
    ```

    **3. Firewall blocking:**
    ```bash
    # Allow ports through firewall
    sudo ufw allow 8000
    sudo ufw allow 5173

    # Or temporarily disable for testing
    sudo ufw disable  # Re-enable after testing!
    ```

### Rate Limiting

??? warning "Too many requests"
    **Symptoms:**
    ```json
    {
      "detail": "Request was throttled. Expected available in 60 seconds."
    }
    ```

    **Cause:** Too many requests in short time.

    **Solutions:**

    - Wait for the cooldown period
    - Implement exponential backoff in your client
    - Contact admin to increase rate limits

    ```javascript
    // Implement retry with backoff
    async function fetchWithRetry(url, options, maxRetries = 3) {
      for (let i = 0; i < maxRetries; i++) {
        const response = await fetch(url, options);

        if (response.status === 429) {
          const waitTime = Math.pow(2, i) * 1000; // Exponential backoff
          await new Promise(resolve => setTimeout(resolve, waitTime));
          continue;
        }

        return response;
      }
    }
    ```

## Database Issues

### Migrations Failing

??? warning "Migration errors"
    **Symptoms:**
    ```
    django.db.migrations.exceptions.InconsistentMigrationHistory
    ```

    **Cause:** Migration history out of sync.

    **Solutions:**

    ```bash
    # View migration status
    docker compose exec django python manage.py showmigrations

    # Option 1: Fake the problematic migration
    docker compose exec django python manage.py migrate --fake api 0001

    # Option 2: Reset migrations (DEVELOPMENT ONLY!)
    docker compose down -v
    docker compose up -d
    docker compose exec django python manage.py migrate

    # Option 3: Manual database reset (DELETES ALL DATA!)
    docker compose exec django_db psql -U django_user -d django_db -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
    docker compose exec django python manage.py migrate
    ```

### Database Locked

??? warning "Database is locked"
    **Symptoms:**
    ```
    OperationalError: database is locked
    ```

    **Cause:** Using SQLite with multiple connections (shouldn't happen with PostgreSQL).

    **Solutions:**

    ```bash
    # Verify PostgreSQL is being used
    docker compose exec django python manage.py dbshell
    # You should see PostgreSQL prompt, not SQLite

    # Check DATABASE setting in .env.local
    DATABASE=postgres  # Not sqlite

    # Restart services
    docker compose restart
    ```

## Stripe / Payment Issues

### LocalStripe Not Working

??? warning "Stripe connection errors in development"
    **Symptoms:**
    - Cannot create subscriptions
    - Payment errors in dev mode

    **Cause:** LocalStripe not configured or not running.

    **Solutions:**

    ```bash
    # Check .env.local
    USE_LOCALSTRIPE=True
    STRIPE_PUBLIC_KEY=pk_test_local
    STRIPE_SECRET_KEY=sk_test_local

    # Verify LocalStripe container running
    docker compose ps localstripe

    # Check logs
    docker compose logs localstripe

    # Restart if needed
    docker compose restart localstripe
    ```

### Stripe Webhooks Not Received

??? warning "Subscription status not updating"
    **Symptoms:**
    - Payment succeeds but subscription stays "incomplete"
    - Webhook events not processed

    **Cause:** Webhook endpoint not configured or accessible.

    **Solutions:**

    **Development (LocalStripe):**
    ```bash
    # LocalStripe automatically sends webhooks locally
    # Check Django logs
    docker compose logs django | grep webhook
    ```

    **Production:**
    ```bash
    # 1. Verify webhook endpoint in Stripe Dashboard
    # URL should be: https://yourdomain.com/api/v1/payments/webhook/

    # 2. Check webhook signing secret
    STRIPE_WEBHOOK_SECRET=whsec_your_actual_secret

    # 3. Verify endpoint is accessible
    curl -X POST https://yourdomain.com/api/v1/payments/webhook/ \
      -H "Content-Type: application/json" \
      -d '{}'
    # Should return 400 (not 404 or 500)

    # 4. Check Django logs for webhook errors
    docker compose logs django | grep -i stripe
    ```

## Frontend Issues

### Frontend Shows Blank Page

??? warning "White screen / blank page"
    **Symptoms:**
    - Browser shows empty page
    - Console errors

    **Solutions:**

    ```bash
    # Check console for errors
    # Open browser DevTools (F12)

    # Check if frontend is running
    docker compose ps frontend
    docker compose logs frontend

    # Rebuild frontend
    docker compose up --build frontend

    # Clear browser cache
    # Ctrl+Shift+R (hard refresh)

    # Check for JS errors
    docker compose logs frontend | grep -i error
    ```

### API Calls Failing from Frontend

??? warning "API requests return errors"
    **Symptoms:**
    - Network errors
    - 401/403 errors
    - CORS errors

    **Solutions:**

    **1. Check API URL:**
    ```bash
    # Verify VITE_API_URL in .env.local
    VITE_API_URL=http://localhost:8000

    # Must rebuild after changing
    docker compose up --build frontend
    ```

    **2. Check token storage:**
    ```javascript
    // In browser console
    localStorage.getItem('auth_token')
    // Should return your token

    // If null, login again
    ```

    **3. Verify backend is accessible:**
    ```bash
    # Test backend directly
    curl http://localhost:8000/api/v1/auth/user/ \
      -H "Authorization: Token {your-token}"
    ```

## Performance Issues

### Slow API Responses

??? warning "API endpoints taking too long"
    **Symptoms:**
    - Requests taking >2 seconds
    - Timeouts

    **Causes & Solutions:**

    **1. Missing database indexes:**
    ```bash
    # Run migrations to ensure indexes exist
    docker compose exec django python manage.py migrate

    # Check for N+1 queries in logs
    # Set DEBUG=True temporarily to see query counts
    ```

    **2. Large result sets:**
    ```bash
    # Use pagination
    curl "http://localhost:8000/api/v1/users/?page=1&page_size=20"

    # Filter results
    curl "http://localhost:8000/api/v1/users/?tenant=1"
    ```

    **3. Cold start:**
    ```bash
    # First request after restart is slower
    # Subsequent requests should be faster due to caching
    ```

### High Memory Usage

??? warning "Docker containers using too much memory"
    **Symptoms:**
    - System slowdown
    - Out of memory errors

    **Solutions:**

    ```bash
    # Check container stats
    docker stats

    # Limit container memory in docker-compose.yml
    services:
      django:
        mem_limit: 1g
        mem_reservation: 512m

    # Restart with limits
    docker compose down
    docker compose up -d
    ```

## Development Issues

### Hot Reload Not Working

??? warning "Changes not reflecting in browser"
    **Symptoms:**
    - Code changes don't appear
    - Need to restart manually

    **Solutions:**

    **Frontend:**
    ```bash
    # Check volume mounts
    docker compose exec frontend ls -la /app/src

    # Verify Vite is in dev mode
    docker compose logs frontend | grep "running at"

    # Restart frontend
    docker compose restart frontend
    ```

    **Backend:**
    ```bash
    # Django auto-reloads in DEBUG mode
    # Check DEBUG=True in .env.local

    # Restart if needed
    docker compose restart django
    ```

### Database Changes Not Persisting

??? warning "Data disappears after restart"
    **Symptoms:**
    - Created data is lost after `docker compose down`

    **Cause:** Using anonymous volumes or `-v` flag.

    **Solutions:**

    ```bash
    # ❌ Don't use -v flag (deletes volumes)
    docker compose down -v

    # ✅ Use this instead
    docker compose down

    # Verify named volumes exist
    docker volume ls | grep wiwebb

    # Backup important data
    docker compose exec django_db pg_dump -U django_user django_db > backup.sql
    ```

## Getting Help

If you're still experiencing issues:

1. **Check Logs:**
   ```bash
   # All services
   docker compose logs

   # Specific service
   docker compose logs django
   docker compose logs frontend

   # Follow logs in real-time
   docker compose logs -f
   ```

2. **Verify Environment:**
   ```bash
   # Check environment variables
   docker compose config

   # Verify services are running
   docker compose ps

   # Check resource usage
   docker stats
   ```

3. **Contact Support:**
   - Email: support@thinesoft.com
   - Include logs and error messages
   - Describe steps to reproduce

4. **Community:**
   - Check [FAQ](faq.md)
   - Search [GitHub Issues](https://github.com/itlds-cmr/labdemo/issues)
   - Ask in community forums

---

!!! tip "Prevention Tips"
    - Keep Docker updated
    - Regularly backup database
    - Monitor logs for warnings
    - Test changes in development first
    - Document custom configurations
