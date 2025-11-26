# Mobility Installation Guide

Complete guide to installing and setting up the Digital Mobility Platform on your development environment.

## Overview

The Digital Mobility Platform is a modern taxi/ride-sharing application built with Expo and React Native. It runs on Android, iOS, and Web platforms, featuring real-time location tracking, ride booking, and payment processing.

## Prerequisites

Before you begin, ensure you have the following installed on your system:

### Required Software

- **Node.js** (v18 or later) - [Download](https://nodejs.org/)
- **npm** or **yarn** - Comes with Node.js
- **Git** - [Download](https://git-scm.com/)
- **Expo CLI** - Will be installed with dependencies

### Platform-Specific Requirements

#### For Android Development

- **Android Studio** - [Download](https://developer.android.com/studio)
- **Android SDK** (API level 26 or higher)
- **Java Development Kit (JDK)** - Version 17 recommended

#### For iOS Development (macOS only)

- **Xcode** (latest version) - [Download from App Store](https://apps.apple.com/app/xcode/id497799835)
- **CocoaPods** - Install via: `sudo gem install cocoapods`
- **iOS Simulator** - Included with Xcode

#### For Web Development

- Modern web browser (Chrome, Firefox, Safari, or Edge)

## Step 1: Clone the Repository

Clone the Mobility platform repository to your local machine:

```bash
git clone <https://github.com/Bode683/digital-mobility-platform>
cd mobility
```

## Step 2: Install Dependencies

Install all required npm packages:

```bash
npm install
```

This will install all dependencies including:

- Expo SDK (~54.0)
- React Native (0.81.5)
- Mapbox Maps integration
- Supabase client
- React Query for state management
- And many more...

## Step 3: Configure Environment Variables

The application requires several API keys and configuration values to function properly.

### Create Environment File

Copy the example environment file:

```bash
cp .env.example .env
```

### Configure Required Variables

Edit the `.env` file and add your credentials:

```bash
# Supabase Configuration
# Get these from: https://supabase.com/dashboard/project/_/settings/api
EXPO_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY=your-publishable-key

# Mapbox Configuration
# Get from: https://account.mapbox.com/access-tokens/
EXPO_PUBLIC_MAPBOX_TOKEN=your-mapbox-public-token
RNMAPBOX_MAPS_DOWNLOAD_TOKEN=your-mapbox-download-token
```

### Obtaining API Keys

#### Supabase Setup

1. Go to [Supabase Dashboard](https://supabase.com/dashboard)
2. Create a new project or select existing one
3. Navigate to **Settings → API**
4. Copy the **Project URL** (EXPO_PUBLIC_SUPABASE_URL)
5. Copy the **anon/public** key (EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY)

#### Mapbox Setup

1. Go to [Mapbox Account](https://account.mapbox.com/)
2. Sign up or log in
3. Navigate to **Access Tokens**
4. Create a new token or use the default public token
5. Copy the token for both EXPO_PUBLIC_MAPBOX_TOKEN and RNMAPBOX_MAPS_DOWNLOAD_TOKEN

## Step 4: Platform-Specific Setup

### Android Setup

#### Configure Android Studio

1. Open Android Studio
2. Go to **SDK Manager** (Tools → SDK Manager)
3. Install Android SDK Platform 26 or higher
4. Install Android SDK Build-Tools
5. Install Android Emulator (if not already installed)

#### Set Environment Variables

Add to your `~/.bashrc`, `~/.zshrc`, or equivalent:

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk  # macOS
# OR
export ANDROID_HOME=$HOME/Android/Sdk  # Linux
# OR
export ANDROID_HOME=%LOCALAPPDATA%\Android\Sdk  # Windows

export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

### iOS Setup (macOS Only)

#### Install CocoaPods Dependencies

```bash
cd ios
pod install
cd ..
```

#### Configure Xcode

1. Open Xcode
2. Go to **Preferences → Locations**
3. Ensure Command Line Tools is set to your Xcode version

## Step 5: Start Development Server

Start the Expo development server:

```bash
npm start
```

Or using Expo CLI directly:

```bash
npx expo start
```

You'll see a QR code and several options in the terminal.

## Step 6: Run on Your Platform

### Run on Android

#### Using Emulator

1. Start Android emulator from Android Studio
2. In the Expo terminal, press `a` to open on Android

Or run directly:

```bash
npm run android
```

#### Using Physical Device

1. Enable **Developer Mode** on your Android device
2. Enable **USB Debugging**
3. Connect device via USB
4. Run `npm run android`

### Run on iOS (macOS Only)

#### Using Simulator

In the Expo terminal, press `i` to open on iOS simulator.

Or run directly:

```bash
npm run ios
```

#### Using Physical Device

1. Register your device in your Apple Developer account
2. Configure code signing in Xcode
3. Build and run through Xcode

### Run on Web

Start the web development server:

```bash
npm run web
```

The app will open in your default browser at `http://localhost:8081`.

## Step 7: Build for Production (Optional)

For production builds, the project uses **Expo Application Services (EAS Build)**.

### Install EAS CLI

```bash
npm install -g eas-cli
```

### Login to EAS

```bash
eas login
```

### Configure EAS Build

The project includes an `eas.json` configuration file. Review and modify if needed.

### Build for Android

```bash
eas build --platform android
```

### Build for iOS

```bash
eas build --platform ios
```

## Verification

After installation, verify everything works:

### Check App Launch

1. App should display animated splash screen
2. Onboarding carousel should appear for first-time users
3. Authentication screen should load properly

### Test Map Functionality

1. Grant location permissions when prompted
2. Map should load with Mapbox tiles
3. Current location marker should appear

### Test Authentication

1. Try creating a new account
2. Verify email/password validation
3. Test sign-in functionality

## Troubleshooting

### Common Issues

#### "Metro bundler failed to start"

```bash
# Clear Metro cache
npx expo start --clear
```

#### "Unable to resolve module"

```bash
# Clean install
rm -rf node_modules
npm install
```

#### "Mapbox not loading"

- Verify your Mapbox tokens in `.env`
- Ensure RNMAPBOX_MAPS_DOWNLOAD_TOKEN is set
- For native builds, you need EAS Development Build

#### Android Build Fails

```bash
# Clean Android build
cd android
./gradlew clean
cd ..
npm run android
```

#### iOS Pod Install Fails

```bash
# Update CocoaPods
cd ios
pod deintegrate
pod install
cd ..
```

### Getting Help

- Check [Configuration](configuration.md) for advanced settings
- See [User Guide](user-guide.md) for usage instructions
- Visit [Support](../shared/support.md) for additional help

## Development Tools

### Clear AsyncStorage

To clear all app data during development:

```bash
npm run clear-async-storage
```

### Linting

Check code quality:

```bash
npm run lint
```

## Next Steps

Now that you have the Mobility platform installed:

1. [Configure](configuration.md) advanced settings
2. Read the [User Guide](user-guide.md) to understand features
3. Explore the [API Documentation](api-reference.md) for integration

## Additional Resources

- [Expo Documentation](https://docs.expo.dev/)
- [React Native Documentation](https://reactnative.dev/)
- [Mapbox GL Documentation](https://github.com/rnmapbox/maps)
- [Supabase Documentation](https://supabase.com/docs)
