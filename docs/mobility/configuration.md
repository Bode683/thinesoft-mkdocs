# Mobility Configuration

Complete guide to configuring the Digital Mobility Platform for development and production environments.

## Overview

The Mobility platform uses a combination of environment variables, configuration files, and runtime settings to control behavior across different environments and platforms.

## Environment Configuration

### Environment Variables

All environment-specific configuration is managed through the `.env` file.

#### Required Variables

```bash
# Supabase Backend Configuration
EXPO_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY=your-anon-public-key

# Mapbox Maps Configuration
EXPO_PUBLIC_MAPBOX_TOKEN=pk.your_mapbox_public_token
RNMAPBOX_MAPS_DOWNLOAD_TOKEN=sk.your_mapbox_secret_token
```

#### Optional Variables

```bash
# API Configuration
API_TIMEOUT=30000  # API request timeout in milliseconds
API_RETRY_COUNT=3  # Number of retry attempts for failed requests

# Feature Flags
ENABLE_ANALYTICS=true
ENABLE_CRASH_REPORTING=true
ENABLE_DEBUG_MODE=false

# Map Configuration
DEFAULT_MAP_ZOOM=15
DEFAULT_MAP_PITCH=0
DEFAULT_MAP_BEARING=0
```

### Environment-Specific Settings

#### Development Environment

```bash
# .env.development
EXPO_PUBLIC_SUPABASE_URL=https://dev-project.supabase.co
EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY=dev-key
EXPO_PUBLIC_MAPBOX_TOKEN=pk.dev_token
RNMAPBOX_MAPS_DOWNLOAD_TOKEN=sk.dev_token
ENABLE_DEBUG_MODE=true
ENABLE_ANALYTICS=false
```

#### Production Environment

```bash
# .env.production
EXPO_PUBLIC_SUPABASE_URL=https://prod-project.supabase.co
EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY=prod-key
EXPO_PUBLIC_MAPBOX_TOKEN=pk.prod_token
RNMAPBOX_MAPS_DOWNLOAD_TOKEN=sk.prod_token
ENABLE_DEBUG_MODE=false
ENABLE_ANALYTICS=true
ENABLE_CRASH_REPORTING=true
```

## App Configuration

### app.config.js

The main app configuration file controls build settings and metadata.

#### Basic Settings

```javascript
module.exports = {
  expo: {
    name: "Digital Mobility",
    slug: "digital-mobility-platform",
    version: "1.0.0",
    orientation: "portrait",
    scheme: "mobility",
    userInterfaceStyle: "automatic",
  }
}
```

#### Platform-Specific Configuration

##### iOS Configuration

```javascript
ios: {
  supportsTablet: true,
  bundleIdentifier: "com.yourcompany.mobility",
  infoPlist: {
    UIBackgroundModes: ["location"],
    NSLocationWhenInUseUsageDescription: "We need your location to show nearby drivers and provide accurate pickup.",
    NSLocationAlwaysAndWhenInUseUsageDescription: "We need your location to track your ride and provide the best service."
  }
}
```

##### Android Configuration

```javascript
android: {
  package: "com.yourcompany.mobility",
  permissions: [
    "android.permission.ACCESS_COARSE_LOCATION",
    "android.permission.ACCESS_FINE_LOCATION",
    "android.permission.FOREGROUND_SERVICE",
    "android.permission.ACCESS_BACKGROUND_LOCATION"  // For background tracking
  ],
  config: {
    googleMaps: {
      apiKey: "YOUR_GOOGLE_MAPS_API_KEY"  // If using Google Maps
    }
  }
}
```

##### Web Configuration

```javascript
web: {
  output: "static",
  favicon: "./assets/images/favicon.png",
  bundler: "metro"
}
```

### EAS Build Configuration

#### eas.json

Configure Expo Application Services builds:

```json
{
  "cli": {
    "version": ">= 5.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_SUPABASE_URL": "https://dev.supabase.co",
        "EXPO_PUBLIC_MAPBOX_TOKEN": "pk.dev_token"
      }
    },
    "preview": {
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_SUPABASE_URL": "https://staging.supabase.co"
      }
    },
    "production": {
      "env": {
        "EXPO_PUBLIC_SUPABASE_URL": "https://prod.supabase.co"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCD1234"
      },
      "android": {
        "serviceAccountKeyPath": "./path/to/api-key.json",
        "track": "production"
      }
    }
  }
}
```

## Backend Configuration

### Supabase Setup

#### Database Schema

The app expects the following Supabase tables:

**profiles**
```sql
CREATE TABLE profiles (
  id UUID REFERENCES auth.users PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  phone_number TEXT,
  avatar_url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**saved_addresses**
```sql
CREATE TABLE saved_addresses (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES profiles(id),
  label TEXT NOT NULL,  -- 'home', 'work', or custom
  address TEXT NOT NULL,
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  created_at TIMESTAMP DEFAULT NOW()
);
```

**trips**
```sql
CREATE TABLE trips (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES profiles(id),
  driver_id UUID,
  pickup_address TEXT,
  pickup_lat DECIMAL(10, 8),
  pickup_lng DECIMAL(11, 8),
  dropoff_address TEXT,
  dropoff_lat DECIMAL(10, 8),
  dropoff_lng DECIMAL(11, 8),
  distance DECIMAL(10, 2),  -- in kilometers
  duration INTEGER,  -- in minutes
  fare DECIMAL(10, 2),
  status TEXT,  -- 'pending', 'active', 'completed', 'cancelled'
  vehicle_type TEXT,  -- 'economy', 'comfort', 'premium', 'xl'
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);
```

**payment_methods**
```sql
CREATE TABLE payment_methods (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES profiles(id),
  type TEXT NOT NULL,  -- 'card', 'wallet', 'cash'
  last_four TEXT,
  card_brand TEXT,
  is_default BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### Row Level Security (RLS)

Enable RLS and create policies:

```sql
-- Enable RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE saved_addresses ENABLE ROW LEVEL SECURITY;
ALTER TABLE trips ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_methods ENABLE ROW LEVEL SECURITY;

-- Profiles policies
CREATE POLICY "Users can view own profile" ON profiles
  FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can update own profile" ON profiles
  FOR UPDATE USING (auth.uid() = id);

-- Saved addresses policies
CREATE POLICY "Users can view own addresses" ON saved_addresses
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own addresses" ON saved_addresses
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- Trips policies
CREATE POLICY "Users can view own trips" ON trips
  FOR SELECT USING (auth.uid() = user_id);

-- Payment methods policies
CREATE POLICY "Users can view own payment methods" ON payment_methods
  FOR SELECT USING (auth.uid() = user_id);
```

#### Supabase Storage

Configure storage buckets for user uploads:

```sql
-- Create avatars bucket
INSERT INTO storage.buckets (id, name, public)
VALUES ('avatars', 'avatars', true);

-- Create storage policy
CREATE POLICY "Avatar images are publicly accessible"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'avatars');

CREATE POLICY "Users can upload own avatar"
  ON storage.objects FOR INSERT
  WITH CHECK (bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]);
```

### Mapbox Configuration

#### Access Tokens

You need two types of Mapbox tokens:

1. **Public Token** (`EXPO_PUBLIC_MAPBOX_TOKEN`)
   - Used for map display and client-side API calls
   - Can be safely included in client code
   - Created in Mapbox dashboard → Access Tokens

2. **Secret Download Token** (`RNMAPBOX_MAPS_DOWNLOAD_TOKEN`)
   - Used for downloading Mapbox SDK during build
   - Should never be committed to version control
   - Created in Mapbox dashboard → Access Tokens

#### Mapbox Styles

Configure custom map styles in `src/constants/theme.ts`:

```typescript
export const MAPBOX_STYLES = {
  light: 'mapbox://styles/mapbox/light-v11',
  dark: 'mapbox://styles/mapbox/dark-v11',
  streets: 'mapbox://styles/mapbox/streets-v12',
  satellite: 'mapbox://styles/mapbox/satellite-streets-v12',
  custom: 'mapbox://styles/your-username/your-style-id'
}
```

## Application Settings

### Theme Configuration

Configure app theming in `src/constants/theme.ts`:

```typescript
export const theme = {
  colors: {
    primary: '#007AFF',
    secondary: '#5856D6',
    success: '#34C759',
    warning: '#FF9500',
    error: '#FF3B30',
    background: {
      light: '#FFFFFF',
      dark: '#000000'
    },
    text: {
      light: '#000000',
      dark: '#FFFFFF'
    }
  },
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32
  },
  borderRadius: {
    sm: 4,
    md: 8,
    lg: 12,
    full: 9999
  }
}
```

### Map Settings

Configure map defaults in `src/lib/constants.ts`:

```typescript
export const MAP_DEFAULTS = {
  zoom: 15,
  pitch: 0,
  bearing: 0,
  animationDuration: 300,

  // Camerasettings
  followUserMode: 'normal',  // 'normal', 'compass', 'course'

  // Marker icons
  userLocationIcon: 'user-location',
  pickupIcon: 'pickup-marker',
  dropoffIcon: 'dropoff-marker',
  driverIcon: 'car-marker'
}
```

### Location Services

Configure location tracking:

```typescript
export const LOCATION_OPTIONS = {
  // Location accuracy
  accuracy: Location.Accuracy.High,

  // Update frequency
  distanceInterval: 10,  // meters
  timeInterval: 5000,    // milliseconds

  // Background location (if needed)
  activityType: Location.ActivityType.AutomotiveNavigation,
  showsBackgroundLocationIndicator: true,
  foregroundService: {
    notificationTitle: 'Mobility is tracking your ride',
    notificationBody: 'Tap to return to the app',
    notificationColor: '#007AFF'
  }
}
```

### API Configuration

Configure API settings in `src/lib/httpClient.ts`:

```typescript
export const API_CONFIG = {
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 30000,
  retryAttempts: 3,
  retryDelay: 1000,
  headers: {
    'Content-Type': 'application/json'
  }
}
```

### React Query Configuration

Configure caching and data fetching in `src/lib/queryClient.ts`:

```typescript
export const queryClientConfig = {
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,  // 5 minutes
      cacheTime: 10 * 60 * 1000,  // 10 minutes
      retry: 2,
      refetchOnWindowFocus: false,
      refetchOnReconnect: true
    },
    mutations: {
      retry: 1
    }
  }
}
```

## Build Configuration

### TypeScript Configuration

`tsconfig.json` includes path aliases for cleaner imports:

```json
{
  "compilerOptions": {
    "strict": true,
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/hooks/*": ["./src/hooks/*"],
      "@/lib/*": ["./src/lib/*"],
      "@/api/*": ["./src/api/*"],
      "@/features/*": ["./src/features/*"]
    }
  }
}
```

### Babel Configuration

`babel.config.js` with module resolver:

```javascript
module.exports = function(api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      'react-native-reanimated/plugin',
      [
        'module-resolver',
        {
          root: ['./'],
          alias: {
            '@': './src',
            '@/components': './src/components',
            '@/hooks': './src/hooks',
            '@/lib': './src/lib',
            '@/api': './src/api',
            '@/features': './src/features'
          }
        }
      ]
    ]
  };
};
```

## Security Configuration

### API Key Management

**Never commit sensitive keys to version control:**

1. Add `.env` to `.gitignore`
2. Use `.env.example` as template
3. Use EAS Secrets for production builds:

```bash
eas secret:create --scope project --name SUPABASE_URL --value "https://..."
eas secret:create --scope project --name SUPABASE_KEY --value "..."
eas secret:create --scope project --name MAPBOX_TOKEN --value "pk..."
```

### SSL/TLS Configuration

All API communication uses HTTPS:

```typescript
// Enforce HTTPS
const enforceHttps = (url: string) => {
  if (!url.startsWith('https://') && !__DEV__) {
    throw new Error('Production must use HTTPS');
  }
  return url;
};
```

## Performance Configuration

### Image Optimization

Configure image caching:

```typescript
export const IMAGE_CONFIG = {
  cacheControl: 'max-age=31536000',  // 1 year
  formats: ['webp', 'png', 'jpg'],
  quality: 80,
  placeholder: 'blur'
}
```

### Code Splitting

Configure lazy loading for better performance:

```typescript
// Lazy load screens
const HomeScreen = lazy(() => import('./screens/Home'));
const ProfileScreen = lazy(() => import('./screens/Profile'));
```

## Monitoring & Analytics

### Error Tracking

Configure Sentry or similar:

```typescript
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  enableInExpoDevelopment: false,
  environment: __DEV__ ? 'development' : 'production'
});
```

### Analytics

Configure analytics tracking:

```typescript
import analytics from '@react-native-firebase/analytics';

export const logEvent = (event: string, params?: object) => {
  if (process.env.ENABLE_ANALYTICS === 'true') {
    analytics().logEvent(event, params);
  }
};
```

## Troubleshooting Configuration

### Common Configuration Issues

#### Environment Variables Not Loading

```bash
# Clear Metro cache
npx expo start --clear

# Restart development server
npm start
```

#### Mapbox Not Working

- Verify both tokens are set in `.env`
- Check token permissions on Mapbox dashboard
- Ensure tokens have correct scopes

#### Supabase Connection Issues

- Verify project URL is correct
- Check API key has correct permissions
- Ensure RLS policies are configured

### Debug Mode

Enable debug logging:

```typescript
// In src/lib/constants.ts
export const DEBUG = {
  api: __DEV__,
  navigation: __DEV__,
  state: __DEV__,
  location: __DEV__
}
```

## Next Steps

- [Installation](installation.md) - Install and set up
- [Features](features.md) - Explore platform features
- [User Guide](user-guide.md) - Learn how to use the app
- [API Reference](api-reference.md) - API documentation
