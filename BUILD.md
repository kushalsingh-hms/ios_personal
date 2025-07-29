# Build Guide

This document provides comprehensive information about the PadnetXpress build system, including Xcode configuration, schemes, environment management, and deployment preparation.

## 📋 Table of Contents

- [Build System Overview](#build-system-overview)
- [Xcode Setup](#xcode-setup)
- [Project Configuration](#project-configuration)
- [Build Schemes](#build-schemes)
- [Environment Configuration](#environment-configuration)
- [Provisioning Profiles](#provisioning-profiles)
- [Build Scripts](#build-scripts)
- [Dependencies Management](#dependencies-management)
- [Troubleshooting](#troubleshooting)

## 🏗️ Build System Overview

PadnetXpress uses a modern iOS build system based on:

- **Xcode 14.0+** for development and building
- **CocoaPods** for dependency management
- **Swift Package Manager** for select dependencies
- **Fastlane** for automation and deployment
- **xcconfig files** for environment-specific configurations

### Build Requirements

| Component | Version | Purpose |
|-----------|---------|---------|
| Xcode | 14.0+ | Primary IDE and build system |
| iOS Deployment Target | 15.0+ | Minimum supported iOS version |
| Swift | 5.5+ | Programming language |
| CocoaPods | 1.11+ | Dependency management |
| Ruby | 2.7+ | For Fastlane and CocoaPods |

## 🔧 Xcode Setup

### Initial Setup

1. **Install Xcode**
   ```bash
   # Install from App Store or download from developer.apple.com
   xcode-select --install
   ```

2. **Install Command Line Tools**
   ```bash
   xcode-select --install
   ```

3. **Install CocoaPods**
   ```bash
   sudo gem install cocoapods
   pod setup
   ```

4. **Clone and Setup Project**
   ```bash
   git clone https://github.com/your-org/PadnetXpress.git
   cd PadnetXpress
   pod install
   open PadnetXpress.xcworkspace
   ```

### Xcode Configuration

#### Build Settings

Key build settings configured for PadnetXpress:

```
IPHONEOS_DEPLOYMENT_TARGET = 15.0
SWIFT_VERSION = 5.5
ENABLE_BITCODE = NO (required for some dependencies)
CLANG_ENABLE_MODULES = YES
SWIFT_COMPILATION_MODE = wholemodule (Release)
SWIFT_OPTIMIZATION_LEVEL = -O (Release)
GCC_OPTIMIZATION_LEVEL = s (Release)
```

#### Code Signing Settings

```
CODE_SIGN_STYLE = Manual
DEVELOPMENT_TEAM = [YOUR_TEAM_ID]
CODE_SIGN_IDENTITY = iPhone Developer (Debug)
CODE_SIGN_IDENTITY = iPhone Distribution (Release)
```

## 📁 Project Configuration

### Project Structure

```
PadnetXpress.xcworkspace/
├── PadnetXpress.xcodeproj
├── Pods/
├── Configurations/
│   ├── Debug.xcconfig
│   ├── Release.xcconfig
│   ├── Staging.xcconfig
│   └── Shared.xcconfig
├── Scripts/
│   ├── build.sh
│   ├── test.sh
│   └── deploy.sh
└── Fastlane/
    ├── Fastfile
    ├── Appfile
    └── Matchfile
```

### Build Configurations

#### Shared.xcconfig
```bash
// Shared configuration for all environments
IPHONEOS_DEPLOYMENT_TARGET = 15.0
SWIFT_VERSION = 5.5
ENABLE_BITCODE = NO
CLANG_ENABLE_MODULES = YES
CLANG_ENABLE_OBJC_ARC = YES

// Health Kit and Background Capabilities
HEALTH_KIT_ENABLED = YES
BACKGROUND_MODES_ENABLED = YES
```

#### Debug.xcconfig
```bash
#include "Shared.xcconfig"

// Debug specific settings
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG
SWIFT_OPTIMIZATION_LEVEL = -Onone
ENABLE_TESTABILITY = YES
GCC_OPTIMIZATION_LEVEL = 0
DEBUG_INFORMATION_FORMAT = dwarf

// API Configuration
API_BASE_URL = https://dev-api.padnetxpress.com
LOGGING_LEVEL = DEBUG
```

#### Release.xcconfig
```bash
#include "Shared.xcconfig"

// Release specific settings
SWIFT_COMPILATION_MODE = wholemodule
SWIFT_OPTIMIZATION_LEVEL = -O
GCC_OPTIMIZATION_LEVEL = s
ENABLE_TESTABILITY = NO
DEBUG_INFORMATION_FORMAT = dwarf-with-dsym

// API Configuration
API_BASE_URL = https://api.padnetxpress.com
LOGGING_LEVEL = ERROR
```

#### Staging.xcconfig
```bash
#include "Shared.xcconfig"

// Staging specific settings
SWIFT_COMPILATION_MODE = wholemodule
SWIFT_OPTIMIZATION_LEVEL = -O
GCC_OPTIMIZATION_LEVEL = s
ENABLE_TESTABILITY = YES

// API Configuration
API_BASE_URL = https://staging-api.padnetxpress.com
LOGGING_LEVEL = INFO
```

## 🎯 Build Schemes

### Scheme Configuration

PadnetXpress uses multiple schemes for different purposes:

#### 1. PadnetXpress (Debug)
- **Purpose**: Development and debugging
- **Configuration**: Debug
- **Bundle ID**: `com.padnetxpress.app.dev`
- **Capabilities**: All development features enabled

```json
{
  "name": "PadnetXpress Debug",
  "buildConfiguration": "Debug",
  "bundleIdentifier": "com.padnetxpress.app.dev",
  "codeSigningRequired": true,
  "provisioningProfile": "PadnetXpress Development",
  "capabilities": {
    "healthKit": true,
    "backgroundModes": true,
    "keychain": true,
    "pushNotifications": true
  }
}
```

#### 2. PadnetXpress (Staging)
- **Purpose**: Pre-production testing
- **Configuration**: Staging
- **Bundle ID**: `com.padnetxpress.app.staging`
- **Capabilities**: Production-like environment

```json
{
  "name": "PadnetXpress Staging",
  "buildConfiguration": "Staging",
  "bundleIdentifier": "com.padnetxpress.app.staging",
  "codeSigningRequired": true,
  "provisioningProfile": "PadnetXpress Staging",
  "capabilities": {
    "healthKit": true,
    "backgroundModes": true,
    "keychain": true,
    "pushNotifications": true
  }
}
```

#### 3. PadnetXpress (Release)
- **Purpose**: Production builds
- **Configuration**: Release
- **Bundle ID**: `com.padnetxpress.app`
- **Capabilities**: Full production setup

```json
{
  "name": "PadnetXpress Release",
  "buildConfiguration": "Release",
  "bundleIdentifier": "com.padnetxpress.app",
  "codeSigningRequired": true,
  "provisioningProfile": "PadnetXpress Distribution",
  "capabilities": {
    "healthKit": true,
    "backgroundModes": true,
    "keychain": true,
    "pushNotifications": true
  }
}
```

### Test Schemes

#### Unit Tests
```bash
# Test configuration
TEST_HOST = $(BUILT_PRODUCTS_DIR)/PadnetXpress.app/PadnetXpress
BUNDLE_LOADER = $(TEST_HOST)
CODE_COVERAGE_ENABLED = YES
```

#### UI Tests
```bash
# UI Test specific settings
TEST_TARGET_NAME = PadnetXpress
UI_TESTING_ENABLED = YES
```

## 🌍 Environment Configuration

### Environment Variables

Environment-specific variables are managed through:

1. **xcconfig files** for build-time configuration
2. **Info.plist** for runtime configuration
3. **Environment-specific plists** for sensitive data

#### Info.plist Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<dict>
    <key>CFBundleDisplayName</key>
    <string>$(APP_DISPLAY_NAME)</string>
    
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    
    <key>APIBaseURL</key>
    <string>$(API_BASE_URL)</string>
    
    <key>LoggingLevel</key>
    <string>$(LOGGING_LEVEL)</string>
    
    <!-- Health Kit Configuration -->
    <key>NSHealthShareUsageDescription</key>
    <string>PadnetXpress uses HealthKit to read your health data for comprehensive monitoring.</string>
    
    <key>NSHealthUpdateUsageDescription</key>
    <string>PadnetXpress uses HealthKit to store your health measurements securely.</string>
    
    <!-- Background Modes -->
    <key>UIBackgroundModes</key>
    <array>
        <string>background-processing</string>
        <string>health-kit</string>
    </array>
</dict>
```

### Environment-Specific Settings

#### Development Environment
```bash
# Debug flags
DEBUG_MODE=1
VERBOSE_LOGGING=1
MOCK_DATA_ENABLED=1
API_TIMEOUT=30
HEALTH_KIT_SIMULATION=1
```

#### Staging Environment
```bash
# Staging flags
DEBUG_MODE=0
VERBOSE_LOGGING=0
MOCK_DATA_ENABLED=0
API_TIMEOUT=15
HEALTH_KIT_SIMULATION=0
ANALYTICS_ENABLED=1
```

#### Production Environment
```bash
# Production flags
DEBUG_MODE=0
VERBOSE_LOGGING=0
MOCK_DATA_ENABLED=0
API_TIMEOUT=10
HEALTH_KIT_SIMULATION=0
ANALYTICS_ENABLED=1
CRASH_REPORTING=1
```

## 🔐 Provisioning Profiles

### Profile Management

PadnetXpress requires specific provisioning profiles for different environments:

#### Development Profile
- **Name**: PadnetXpress Development
- **Type**: iOS App Development
- **Bundle ID**: `com.padnetxpress.app.dev`
- **Capabilities**: HealthKit, Background Modes, Push Notifications, Keychain Sharing

#### Staging Profile
- **Name**: PadnetXpress Staging
- **Type**: Ad Hoc Distribution
- **Bundle ID**: `com.padnetxpress.app.staging`
- **Capabilities**: All production capabilities

#### Distribution Profile
- **Name**: PadnetXpress Distribution
- **Type**: App Store Distribution
- **Bundle ID**: `com.padnetxpress.app`
- **Capabilities**: All production capabilities

### Manual Profile Setup

1. **Download profiles from Apple Developer Portal**
2. **Install profiles** by double-clicking or using Xcode
3. **Verify installation** in Xcode Preferences > Accounts

### Automatic Profile Management (Recommended)

Using Fastlane Match for automatic profile management:

```ruby
# Matchfile
git_url("https://github.com/your-org/certificates")
storage_mode("git")
type("development")
app_identifier(["com.padnetxpress.app", "com.padnetxpress.app.dev", "com.padnetxpress.app.staging"])
username("developer@padnetxpress.com")
```

## 📝 Build Scripts

### Manual Build Commands

#### Debug Build
```bash
# Clean and build debug version
xcodebuild clean -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Debug"
xcodebuild build -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Debug" -destination 'platform=iOS Simulator,name=iPhone 14'
```

#### Release Build
```bash
# Build for device (requires provisioning profile)
xcodebuild build -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Release" -destination 'generic/platform=iOS'
```

#### Archive Build
```bash
# Create archive for distribution
xcodebuild archive -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Release" -archivePath "build/PadnetXpress.xcarchive"
```

### Automated Build Script

Create `scripts/build.sh`:

```bash
#!/bin/bash

set -e

# Configuration
WORKSPACE="PadnetXpress.xcworkspace"
PROJECT_NAME="PadnetXpress"
BUILD_DIR="build"
ARCHIVE_PATH="$BUILD_DIR/$PROJECT_NAME.xcarchive"

# Parse command line arguments
SCHEME="PadnetXpress Release"
CONFIGURATION="Release"
EXPORT_METHOD="app-store"

while [[ $# -gt 0 ]]; do
    case $1 in
        --scheme)
            SCHEME="$2"
            shift 2
            ;;
        --configuration)
            CONFIGURATION="$2"
            shift 2
            ;;
        --export-method)
            EXPORT_METHOD="$2"
            shift 2
            ;;
        *)
            echo "Unknown option $1"
            exit 1
            ;;
    esac
done

echo "🏗️ Building PadnetXpress"
echo "Scheme: $SCHEME"
echo "Configuration: $CONFIGURATION"
echo "Export Method: $EXPORT_METHOD"

# Clean build directory
echo "🧹 Cleaning build directory..."
rm -rf "$BUILD_DIR"
mkdir -p "$BUILD_DIR"

# Install dependencies
echo "📦 Installing dependencies..."
pod install

# Clean Xcode build
echo "🧹 Cleaning Xcode project..."
xcodebuild clean -workspace "$WORKSPACE" -scheme "$SCHEME"

# Build archive
echo "📦 Creating archive..."
xcodebuild archive \
    -workspace "$WORKSPACE" \
    -scheme "$SCHEME" \
    -configuration "$CONFIGURATION" \
    -archivePath "$ARCHIVE_PATH" \
    -allowProvisioningUpdates

# Export IPA
echo "📱 Exporting IPA..."
xcodebuild -exportArchive \
    -archivePath "$ARCHIVE_PATH" \
    -exportPath "$BUILD_DIR" \
    -exportOptionsPlist "ExportOptions.plist"

echo "✅ Build completed successfully!"
echo "Archive: $ARCHIVE_PATH"
echo "IPA: $BUILD_DIR/$PROJECT_NAME.ipa"
```

### Export Options

Create `ExportOptions.plist` for different export methods:

#### App Store Export
```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
    <key>compileBitcode</key>
    <false/>
</dict>
</plist>
```

#### Ad Hoc Export
```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>method</key>
    <string>ad-hoc</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadBitcode</key>
    <false/>
    <key>compileBitcode</key>
    <false/>
</dict>
</plist>
```

## 📦 Dependencies Management

### CocoaPods Configuration

#### Podfile
```ruby
# Minimum iOS version
platform :ios, '15.0'

# Use frameworks
use_frameworks!

# Disable Bitcode (required for some pods)
install! 'cocoapods', :disable_input_output_paths => true

target 'PadnetXpress' do
  # Networking
  pod 'Alamofire', '~> 5.6'
  pod 'Moya', '~> 15.0'
  
  # Data Visualization
  pod 'DSFSparkline', :git => 'https://github.com/dagronf/DSFSparkline/'
  
  # Security
  pod 'KeychainSwift', '~> 20.0'
  pod 'CryptoSwift', '~> 1.6'
  
  # Cloud Services
  pod 'AWSS3', '2.10.0'
  pod 'SwiftProtobuf', '~> 1.20'
  
  # Analytics & Monitoring
  pod 'Firebase/Crashlytics'
  pod 'Firebase/Analytics'
  
  # UI
  pod 'AlertToast', '~> 1.3'
  
  # Utilities
  pod 'SSZipArchive', '~> 2.4'
  
  target 'PadnetXpressTests' do
    inherit! :search_paths
    # Test dependencies if needed
  end
  
  target 'PadnetXpressUITests' do
    inherit! :search_paths
  end
end

# Post-install configuration
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      config.build_settings['IPHONEOS_DEPLOYMENT_TARGET'] = '15.0'
      config.build_settings['ENABLE_BITCODE'] = 'NO'
    end
  end
end
```

### Dependency Updates

#### Update Dependencies
```bash
# Update all pods to latest versions
pod update

# Update specific pod
pod update Alamofire

# Install new dependencies
pod install
```

#### Version Locking
```bash
# Generate Podfile.lock for version consistency
pod install --deployment
```

## 🚨 Troubleshooting

### Common Build Issues

#### 1. CocoaPods Issues

**Problem**: "Unable to find a specification for..."
```bash
# Solution
pod repo update
pod install --repo-update
```

**Problem**: Version conflicts
```bash
# Solution
pod deintegrate
pod install
```

#### 2. Code Signing Issues

**Problem**: "No matching provisioning profiles found"
- Verify Bundle ID matches provisioning profile
- Check team membership and certificates
- Download and install latest profiles

**Problem**: "Code signing is required for product type 'Application'"
- Ensure development team is selected
- Verify code signing identity is set
- Check provisioning profile is valid

#### 3. Build Performance Issues

**Slow compilation times**:
```bash
# Enable build timing
defaults write com.apple.dt.Xcode ShowBuildOperationDuration -bool YES

# Use incremental builds
# Set SWIFT_COMPILATION_MODE = incremental for Debug builds
```

#### 4. Simulator Issues

**Problem**: App doesn't run in simulator
- Reset simulator: Device → Erase All Content and Settings
- Clean build folder: Product → Clean Build Folder
- Delete derived data: ~/Library/Developer/Xcode/DerivedData

### Build Logs Analysis

#### Verbose Build Logs
```bash
# Enable verbose logging
xcodebuild -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Debug" -destination 'platform=iOS Simulator,name=iPhone 14' build | xcpretty --color
```

#### Build Time Analysis
```bash
# Analyze build times
xcodebuild -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Debug" build -quiet | grep -E "^\s*[0-9]+\.[0-9]+s"
```

### Emergency Build Recovery

#### Clean Everything
```bash
#!/bin/bash
# Nuclear option - clean everything and rebuild

echo "🧹 Cleaning everything..."

# Clean Xcode
xcodebuild clean -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Debug"

# Remove derived data
rm -rf ~/Library/Developer/Xcode/DerivedData/PadnetXpress-*

# Clean CocoaPods
pod deintegrate
pod clean
rm -rf Pods/
rm Podfile.lock

# Reinstall
pod install

echo "✅ Clean complete. Try building again."
```

## 📊 Build Metrics

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Clean Build Time | < 3 minutes | Debug configuration |
| Incremental Build | < 30 seconds | Single file change |
| Archive Time | < 5 minutes | Release configuration |
| Test Execution | < 2 minutes | All unit tests |

### Monitoring Build Performance

```bash
# Measure build time
time xcodebuild build -workspace PadnetXpress.xcworkspace -scheme "PadnetXpress Debug"

# Track build size
du -sh build/PadnetXpress.xcarchive

# Monitor dependency impact
pod install --verbose
```

---

For build support and questions, contact: build-support@padnetxpress.com 