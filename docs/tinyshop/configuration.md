# TinyShop Configuration

Advanced configuration options for customizing your TinyShop store.

## General Settings

### Store Information

Configure basic store details:

```yaml
Store Name: Your Store Name
Store URL: https://yourstore.com
Email: info@yourstore.com
Phone: +1-234-567-8900
```

### Regional Settings

Set up regional preferences:

- **Currency** - USD, EUR, GBP, etc.
- **Time Zone** - Your local time zone
- **Date Format** - MM/DD/YYYY or DD/MM/YYYY
- **Weight Unit** - Pounds or Kilograms

## Payment Configuration

### Payment Gateways

#### PayPal

```php
'paypal' => [
    'enabled' => true,
    'mode' => 'live', // or 'sandbox'
    'client_id' => 'your_client_id',
    'secret' => 'your_secret'
]
```

#### Stripe

```php
'stripe' => [
    'enabled' => true,
    'publishable_key' => 'pk_live_...',
    'secret_key' => 'sk_live_...'
]
```

### Payment Options

Configure payment behavior:

- Accept credit cards
- Enable PayPal Express Checkout
- Cash on delivery
- Bank transfer instructions

## Shipping Configuration

### Shipping Zones

Define shipping zones:

1. **Domestic** - Same country
2. **International** - Other countries
3. **Local Pickup** - Store pickup option

### Shipping Methods

Configure shipping options:

```yaml
Flat Rate:
  Cost: $5.00
  Estimated Days: 3-5

Free Shipping:
  Minimum Order: $50.00

Express:
  Cost: $15.00
  Estimated Days: 1-2
```

### Shipping Rates

Set up rate calculations:

- Flat rate by order
- Price-based shipping
- Weight-based shipping
- Location-based rates

## Tax Configuration

### Tax Settings

Configure tax calculation:

- Enable/disable taxes
- Display prices with/without tax
- Tax calculation based on shipping or billing address

### Tax Rates

Set up tax rates by location:

```yaml
California:
  Rate: 7.25%

New York:
  Rate: 8.875%
```

## Email Configuration

### SMTP Settings

Configure email delivery:

```php
'mail' => [
    'driver' => 'smtp',
    'host' => 'smtp.mailtrap.io',
    'port' => 587,
    'username' => 'your_username',
    'password' => 'your_password',
    'encryption' => 'tls'
]
```

### Email Templates

Customize email templates for:

- Order confirmation
- Shipping notification
- Order cancellation
- Password reset
- Newsletter

## Theme and Appearance

### Theme Selection

Choose and customize themes:

- Default theme
- Custom theme upload
- Color scheme customization

### Logo and Branding

Upload your branding assets:

- Store logo (recommended: 250x100px)
- Favicon (32x32px)
- Email header image

## SEO Configuration

### Meta Information

Set default SEO settings:

```yaml
Meta Title: Your Store - Quality Products
Meta Description: Shop quality products at great prices
Meta Keywords: products, store, shopping
```

### URL Structure

Configure URL format:

- Product URL pattern
- Category URL pattern
- Enable/disable URL rewriting

## Security Settings

### Security Features

Enable security measures:

- SSL/HTTPS enforcement
- Two-factor authentication for admin
- CAPTCHA on forms
- Login attempt limits

### Backup Configuration

Set up automatic backups:

- Daily database backups
- Weekly file backups
- Backup retention period

## Performance Settings

### Caching

Enable caching for better performance:

- Page caching
- Database query caching
- Image optimization

### Optimization

Performance optimization options:

- Minify CSS/JavaScript
- Enable compression
- Lazy load images

## Advanced Settings

### API Configuration

Enable API access:

```php
'api' => [
    'enabled' => true,
    'key' => 'your_api_key',
    'rate_limit' => 1000 // requests per hour
]
```

### Custom Code

Add custom functionality:

- Header scripts (Google Analytics)
- Footer scripts
- Custom CSS
- Custom JavaScript

## Integration Settings

### Third-Party Integrations

Connect with external services:

- Google Analytics
- Facebook Pixel
- Mailchimp
- Zapier

## Maintenance Mode

Enable maintenance mode:

- Display maintenance message
- Allow admin access only
- Scheduled maintenance

## Troubleshooting

Configuration issues? Check our [Support](../shared/support.md) page.
