# Installation Guide

This guide will walk you through installing WiWebb using Docker. The entire platform runs in containers, making installation straightforward and consistent across all platforms.

## Prerequisites

Before installing WiWebb, ensure your system meets these requirements:

!!! warning "Required Software"
    - **Docker** 20.10 or higher
    - **Docker Compose** 2.0 or higher
    - **Git** for cloning the repository
    - **4GB RAM** minimum (8GB recommended for production)
    - **10GB free disk space**

### Installing Docker

=== "Linux"
    ```bash
    # Install Docker using the convenience script
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh

    # Add your user to the docker group
    sudo usermod -aG docker $USER

    # Install Docker Compose
    sudo apt-get update
    sudo apt-get install docker-compose-plugin

    # Verify installation
    docker --version
    docker compose version
    ```

=== "macOS"
    ```bash
    # Install Docker Desktop using Homebrew
    brew install --cask docker

    # Start Docker Desktop from Applications
    # Then verify installation
    docker --version
    docker compose version
    ```

=== "Windows"
    ```powershell
    # Install Docker Desktop from docker.com
    # Ensure WSL2 is enabled and installed
    # Download from: https://www.docker.com/products/docker-desktop

    # After installation, verify in PowerShell:
    docker --version
    docker compose version
    ```

## Installation Steps

### Step 1: Clone the Repository

Clone the WiWebb repository to your local machine:

```bash
git clone https://github.com/itlds-cmr/labdemo.git wiwebb
cd wiwebb
```

!!! tip "Repository Location"
    The repository is located at `/home/nkem/Desktop/itlds/labdemo-django-stack` on the server.
    For your installation, clone it to a location of your choice.

### Step 2: Configure Environment Variables

WiWebb uses environment files to manage configuration. Create your environment file from the example:

```bash
# Copy the example environment file
cp .env.example .env.local
```

Now edit `.env.local` with your preferred text editor:

```bash
nano .env.local  # or use vim, code, etc.
```

!!! danger "Security - Change These Values!"
    **CRITICAL:** You MUST change these values before deploying to production:

    - `SECRET_KEY` - Django secret key
    - `POSTGRES_PASSWORD` - Database password
    - `DJANGO_SUPERUSER_PASSWORD` - Admin password

**Minimum required configuration:**

```bash title=".env.local"
# Django Settings
SECRET_KEY=your-secret-key-here  # (1)!
DEBUG=True  # Set to False in production
ALLOWED_HOSTS=localhost,127.0.0.1

# Database
POSTGRES_DB=django_db
POSTGRES_USER=django_user
POSTGRES_PASSWORD=your-secure-password-here  # (2)!
POSTGRES_HOST=django_db
POSTGRES_PORT=5432

# Superuser credentials (created automatically)
DJANGO_SUPERUSER_USERNAME=admin
DJANGO_SUPERUSER_EMAIL=admin@example.com
DJANGO_SUPERUSER_PASSWORD=your-admin-password  # (3)!

# Stripe (Development)
USE_LOCALSTRIPE=True  # Use local Stripe simulator for development
STRIPE_PUBLIC_KEY=pk_test_local
STRIPE_SECRET_KEY=sk_test_local
STRIPE_WEBHOOK_SECRET=whsec_local

# Frontend
VITE_API_URL=http://localhost:8000
```

1. Generate a secure secret key using: `python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'`
2. Use a strong password with letters, numbers, and special characters
3. This will be your admin login password

!!! example "Generating a Secret Key"
    To generate a secure Django secret key, run:

    ```bash
    python3 -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
    ```

    Copy the output and use it as your `SECRET_KEY`.

### Step 3: Build and Start Services

Build the Docker images and start all services:

```bash
# Build and start all containers
docker compose up --build -d
```

This command will:

1. Build the Django backend image
2. Build the React frontend image
3. Pull PostgreSQL, Redis, and LocalStripe images
4. Create and start all containers
5. Create a network for inter-container communication

!!! info "First Build Time"
    The first build may take 5-10 minutes depending on your internet connection,
    as Docker needs to download all base images and dependencies.

**Monitor the build progress:**

```bash
# Watch logs from all services
docker compose logs -f

# Watch specific service
docker compose logs -f django
```

### Step 4: Run Database Migrations

Once the containers are running, apply database migrations:

```bash
# Run Django migrations
docker compose exec django python manage.py migrate
```

Expected output:
```
Operations to perform:
  Apply all migrations: admin, api, auth, contenttypes, payment_gateway, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying api.0001_initial... OK
  ...
```

### Step 5: Create Superuser (if not auto-created)

If the superuser wasn't created automatically from environment variables:

```bash
docker compose exec django python manage.py createsuperuser
```

Follow the prompts to enter username, email, and password.

### Step 6: Verify Installation

Check that all services are running:

```bash
docker compose ps
```

Expected output:
```
NAME                  SERVICE      STATUS       PORTS
wiwebb-django-1       django       running      0.0.0.0:8000->8000/tcp
wiwebb-frontend-1     frontend     running      0.0.0.0:5173->5173/tcp
wiwebb-db-1           django_db    running      5432/tcp
wiwebb-redis-1        redis        running      6379/tcp
wiwebb-localstripe-1  localstripe  running      8420/tcp
```

### Step 7: Access WiWebb

Open your web browser and navigate to:

- **Frontend:** [http://localhost:5173](http://localhost:5173)
- **API:** [http://localhost:8000/api/v1/](http://localhost:8000/api/v1/)
- **Django Admin:** [http://localhost:8000/admin/](http://localhost:8000/admin/)

!!! success "Installation Complete!"
    If you can access the login page, WiWebb is successfully installed!

    Default login credentials:
    - **Username:** admin (or what you set in `.env.local`)
    - **Password:** (from `DJANGO_SUPERUSER_PASSWORD` in `.env.local`)

## Post-Installation Steps

### Collect Static Files (Production Only)

For production deployments, collect static files:

```bash
docker compose exec django python manage.py collectstatic --noinput
```

### Create Initial Data

You may want to create some initial data for testing:

```bash
# Access Django shell
docker compose exec django python manage.py shell

# Then in the shell:
from api.models import Tenant, User
from payment_gateway.models import Plan, PlanPricing

# Create a tenant
tenant = Tenant.objects.create(
    name="Example Organization",
    slug="example-org",
    email="contact@example.com"
)

# Create a subscription plan
plan = Plan.objects.create(
    name="Basic Plan",
    description="Basic features for small teams",
    daily_time_minutes=480,  # 8 hours
    daily_data_mb=1024,  # 1 GB
    requires_payment=True
)

# Exit the shell
exit()
```

## Directory Structure

After installation, your directory structure looks like this:

```
wiwebb/
├── docker-compose.yml       # Docker services configuration
├── .env.local              # Your environment variables
├── backend/                # Django backend
│   ├── api/               # User and tenant management
│   ├── payment_gateway/   # Payment and subscriptions
│   ├── backend/           # Django settings
│   └── manage.py          # Django CLI
├── frontend/              # React frontend
│   ├── src/              # Source code
│   ├── public/           # Static assets
│   └── package.json      # Dependencies
└── docs/                 # Documentation
```

## Common Installation Issues

??? warning "Port Already in Use"
    **Error:** `Bind for 0.0.0.0:8000 failed: port is already allocated`

    **Solution:** Another service is using port 8000, 5173, or 5432.

    ```bash
    # Find what's using the port
    lsof -i :8000  # Linux/macOS
    netstat -ano | findstr :8000  # Windows

    # Either stop that service or change ports in docker-compose.yml
    ```

??? warning "Permission Denied"
    **Error:** `Got permission denied while trying to connect to the Docker daemon`

    **Solution:** Add your user to the docker group (Linux):

    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```

??? warning "Database Connection Failed"
    **Error:** `could not connect to server: Connection refused`

    **Solution:** Wait for PostgreSQL to fully initialize (first startup takes longer):

    ```bash
    docker compose logs django_db
    # Wait for: "database system is ready to accept connections"
    ```

??? warning "Frontend Can't Connect to Backend"
    **Error:** Network errors when logging in

    **Solution:** Check the `VITE_API_URL` in `.env.local`:

    ```bash
    # Should be:
    VITE_API_URL=http://localhost:8000

    # Rebuild frontend:
    docker compose up --build frontend
    ```

## Docker Commands Reference

Useful Docker commands for managing WiWebb:

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f [service]

# Restart a service
docker compose restart [service]

# Access Django shell
docker compose exec django python manage.py shell

# Access Django database shell
docker compose exec django python manage.py dbshell

# Access container bash
docker compose exec django bash

# View running containers
docker compose ps

# Remove all containers and volumes (DESTRUCTIVE!)
docker compose down -v
```

!!! danger "Data Loss Warning"
    The command `docker compose down -v` will delete all data including the database!
    Only use this if you want to start fresh.

## Next Steps

Now that WiWebb is installed, proceed to the [Quick Start Guide](quick-start.md) to learn how to:

- Log in to the web interface
- Create your first tenant
- Add users
- Navigate the dashboard

---

!!! question "Need Help?"
    - [Quick Start Guide](quick-start.md) - Learn the basics
    - [Configuration Guide](configuration.md) - Configure WiWebb
    - [Troubleshooting](../troubleshooting/common-issues.md) - Common issues
    - [FAQ](../troubleshooting/faq.md) - Frequently asked questions
