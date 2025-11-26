# TinyShop Setup Guide

Get your TinyShop store up and running quickly with this step-by-step guide.

## Prerequisites

Before setting up TinyShop, ensure you have:

- A domain name
- Web hosting with PHP support
- MySQL or PostgreSQL database
- SSL certificate (recommended)

## Installation Steps

### Step 1: Download TinyShop

Download the latest version from your account dashboard.

### Step 2: Upload Files

Upload all files to your web server:

```bash
# Using FTP/SFTP
# Upload the tinyshop folder to your web root
```

### Step 3: Database Setup

Create a new database and import the schema:

```sql
CREATE DATABASE tinyshop;
GRANT ALL PRIVILEGES ON tinyshop.* TO 'tinyshop_user'@'localhost';
```

### Step 4: Configuration

Edit the configuration file:

```php
// config.php
define('DB_HOST', 'localhost');
define('DB_NAME', 'tinyshop');
define('DB_USER', 'tinyshop_user');
define('DB_PASS', 'your_password');
```

### Step 5: Run Setup Wizard

Navigate to `https://yourstore.com/setup` and complete the setup wizard:

1. Verify system requirements
2. Configure database connection
3. Set up admin account
4. Configure basic store settings

## Initial Configuration

### Store Information

Set up your basic store information:

- Store name
- Logo and branding
- Contact information
- Business address

### Payment Methods

Configure payment gateways:

- PayPal
- Stripe
- Bank transfer
- Cash on delivery

### Shipping Options

Set up shipping methods:

- Flat rate
- Free shipping
- Location-based rates

## Security

### Important Security Steps

1. Delete the `/setup` folder after installation
2. Change default admin credentials
3. Enable SSL/HTTPS
4. Set proper file permissions

## Verification

Verify your installation:

- Admin panel is accessible
- Store front page loads
- Test checkout process

## Next Steps

- [User Guide](user-guide.md) - Learn to manage your store
- [Configuration](configuration.md) - Advanced settings

## Troubleshooting

Common setup issues:

- **Database connection error** - Verify credentials
- **Permission denied** - Check file permissions
- **White screen** - Enable error logging

Need help? Visit our [Support](../shared/support.md) page.
