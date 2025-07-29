# Deployment Guide

This comprehensive guide covers the deployment process for PadnetXpress, including TestFlight beta distribution and App Store submission using Fastlane automation.

## 📋 Table of Contents

- [Deployment Overview](#deployment-overview)
- [Prerequisites](#prerequisites)
- [Fastlane Setup](#fastlane-setup)
- [TestFlight Deployment](#testflight-deployment)
- [App Store Submission](#app-store-submission)
- [Release Management](#release-management)
- [Automated CI/CD](#automated-cicd)
- [Troubleshooting](#troubleshooting)

## 🚀 Deployment Overview

PadnetXpress uses a multi-stage deployment pipeline:

1. **Development** → Internal testing and development
2. **Staging** → Internal beta testing via TestFlight
3. **Beta** → External beta testing via TestFlight
4. **Production** → App Store release

### Deployment Flow

```mermaid
graph LR
    A[Development] --> B[Code Review]
    B --> C[Staging Build]
    C --> D[Internal Testing]
    D --> E[Beta TestFlight]
    E --> F[External Testing]
    F --> G[Production Build]
    G --> H[App Store Review]
    H --> I[App Store Release]
```

## ✅ Prerequisites

### Apple Developer Account Setup

1. **Apple Developer Program membership** ($99/year)
2. **App Store Connect access** with appropriate roles
3. **Certificates and Provisioning Profiles** properly configured
4. **App Store Connect app** created with correct Bundle ID

### Required Certificates

| Certificate Type | Purpose | Environment |
|-----------------|---------|-------------|
| iOS Development | Local development and testing | Development |
| iOS Distribution | Ad Hoc and App Store distribution | Staging/Production |

### Provisioning Profiles

| Profile Type | Bundle ID | Usage |
|-------------|-----------|-------|
| Development | `com.padnetxpress.app.dev` | Development builds |
| Ad Hoc | `com.padnetxpress.app.staging` | Internal testing |
| App Store | `com.padnetxpress.app` | Production release |

### Local Environment

```bash
# Required tools
brew install fastlane
gem install fastlane
xcode-select --install

# Verify installation
fastlane --version
```

## 🚄 Fastlane Setup

### Installation and Initialization

```bash
# Navigate to project directory
cd PadnetXpress

# Initialize Fastlane
fastlane init

# Follow the setup wizard and provide:
# 1. Apple ID (developer@padnetxpress.com)
# 2. App Bundle ID (com.padnetxpress.app)
# 3. App Store Connect Team ID
```

### Fastlane Configuration Files

#### Fastfile
```ruby
# fastlane/Fastfile

default_platform(:ios)

# Global variables
WORKSPACE = "PadnetXpress.xcworkspace"
PROJECT = "PadnetXpress.xcodeproj"
APP_IDENTIFIER = "com.padnetxpress.app"
TEAM_ID = "YOUR_TEAM_ID"

platform :ios do
  
  # Development Lane
  desc "Build and test development version"
  lane :development do
    clear_derived_data
    cocoapods
    
    build_app(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Debug",
      configuration: "Debug",
      destination: "generic/platform=iOS Simulator",
      skip_archive: true
    )
    
    run_tests(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Debug",
      device: "iPhone 14"
    )
  end
  
  # Staging Lane - Internal TestFlight
  desc "Build and upload to TestFlight for internal testing"
  lane :staging do
    ensure_git_status_clean
    increment_build_number(xcodeproj: PROJECT)
    
    clear_derived_data
    cocoapods
    
    # Certificate and profile management
    match(
      type: "adhoc",
      app_identifier: "com.padnetxpress.app.staging",
      readonly: true
    )
    
    build_app(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Staging",
      configuration: "Staging",
      export_method: "ad-hoc",
      export_options: {
        provisioningProfiles: {
          "com.padnetxpress.app.staging" => "match AdHoc com.padnetxpress.app.staging"
        }
      }
    )
    
    upload_to_testflight(
      app_identifier: "com.padnetxpress.app.staging",
      skip_waiting_for_build_processing: false,
      groups: ["Internal Team"],
      notify_external_testers: false
    )
    
    # Commit build number increment
    commit_version_bump(
      message: "Bump build number for staging release",
      xcodeproj: PROJECT
    )
    
    push_to_git_remote
  end
  
  # Beta Lane - External TestFlight
  desc "Build and upload to TestFlight for external testing"
  lane :beta do
    ensure_git_status_clean
    increment_version_number(bump_type: "patch")
    increment_build_number(xcodeproj: PROJECT)
    
    clear_derived_data
    cocoapods
    
    # Run comprehensive tests
    run_tests(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Release",
      device: "iPhone 14"
    )
    
    # Certificate and profile management
    match(
      type: "appstore",
      app_identifier: APP_IDENTIFIER,
      readonly: true
    )
    
    build_app(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Release",
      configuration: "Release",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          APP_IDENTIFIER => "match AppStore #{APP_IDENTIFIER}"
        }
      }
    )
    
    upload_to_testflight(
      app_identifier: APP_IDENTIFIER,
      skip_waiting_for_build_processing: false,
      groups: ["Beta Testers"],
      notify_external_testers: true,
      beta_app_review_info: {
        contact_email: "support@padnetxpress.com",
        contact_first_name: "PadnetXpress",
        contact_last_name: "Support",
        contact_phone: "+1-555-0123",
        demo_account_name: "beta@padnetxpress.com",
        demo_account_password: "BetaTest123!",
        notes: "Health monitoring app for tracking heart rate, blood pressure, and sleep patterns."
      }
    )
    
    # Tag release
    add_git_tag(
      tag: "v#{get_version_number(xcodeproj: PROJECT)}-beta.#{get_build_number(xcodeproj: PROJECT)}"
    )
    
    commit_version_bump(
      message: "Release v#{get_version_number(xcodeproj: PROJECT)} beta",
      xcodeproj: PROJECT
    )
    
    push_to_git_remote(tags: true)
    
    # Notify team
    slack(
      message: "🚀 PadnetXpress v#{get_version_number(xcodeproj: PROJECT)} beta is now available on TestFlight!",
      channel: "#releases",
      success: true
    )
  end
  
  # Production Lane - App Store Release
  desc "Build and submit to App Store for review"
  lane :release do
    ensure_git_status_clean
    ensure_git_branch(branch: "main")
    
    # Version management
    increment_version_number(bump_type: "minor")
    increment_build_number(xcodeproj: PROJECT)
    
    clear_derived_data
    cocoapods
    
    # Comprehensive testing
    run_tests(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Release",
      device: "iPhone 14"
    )
    
    # Certificate and profile management
    match(
      type: "appstore",
      app_identifier: APP_IDENTIFIER,
      readonly: true
    )
    
    build_app(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Release",
      configuration: "Release",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          APP_IDENTIFIER => "match AppStore #{APP_IDENTIFIER}"
        }
      }
    )
    
    # App Store metadata update
    deliver(
      app_identifier: APP_IDENTIFIER,
      submit_for_review: false,
      automatic_release: false,
      force: true,
      metadata_path: "./fastlane/metadata",
      screenshots_path: "./fastlane/screenshots"
    )
    
    # Upload binary
    upload_to_app_store(
      app_identifier: APP_IDENTIFIER,
      skip_binary_upload: false,
      skip_metadata: true,
      skip_screenshots: true,
      submit_for_review: true,
      automatic_release: false,
      submission_information: {
        add_id_info_limits_tracking: true,
        add_id_info_serves_ads: false,
        add_id_info_tracks_action: true,
        add_id_info_tracks_install: true,
        add_id_info_uses_idfa: false,
        content_rights_has_rights: true,
        content_rights_contains_third_party_content: false,
        export_compliance_platform: "ios",
        export_compliance_compliance_required: false,
        export_compliance_encryption_updated: false,
        export_compliance_app_type: nil,
        export_compliance_uses_encryption: true,
        export_compliance_is_exempt: false,
        export_compliance_contains_third_party_cryptography: false,
        export_compliance_contains_proprietary_cryptography: false,
        export_compliance_available_on_french_store: true
      }
    )
    
    # Tag release
    add_git_tag(
      tag: "v#{get_version_number(xcodeproj: PROJECT)}"
    )
    
    commit_version_bump(
      message: "Release v#{get_version_number(xcodeproj: PROJECT)}",
      xcodeproj: PROJECT
    )
    
    push_to_git_remote(tags: true)
    
    # Create GitHub release
    github_release = set_github_release(
      repository_name: "your-org/PadnetXpress",
      api_token: ENV["GITHUB_TOKEN"],
      name: "v#{get_version_number(xcodeproj: PROJECT)}",
      tag_name: "v#{get_version_number(xcodeproj: PROJECT)}",
      description: "See CHANGELOG.md for details",
      is_draft: false,
      is_prerelease: false
    )
    
    # Notify team
    slack(
      message: "🎉 PadnetXpress v#{get_version_number(xcodeproj: PROJECT)} has been submitted to the App Store!",
      channel: "#releases",
      success: true
    )
  end
  
  # Utility Lanes
  desc "Update certificates and provisioning profiles"
  lane :certificates do
    match(type: "development", readonly: false)
    match(type: "adhoc", readonly: false)
    match(type: "appstore", readonly: false)
  end
  
  desc "Run all tests"
  lane :test do
    run_tests(
      workspace: WORKSPACE,
      scheme: "PadnetXpress Debug",
      device: "iPhone 14",
      code_coverage: true
    )
  end
  
  desc "Create app screenshots"
  lane :screenshots do
    snapshot
  end
  
  # Error handling
  error do |lane, exception|
    slack(
      message: "❌ Error in lane #{lane}: #{exception.message}",
      channel: "#releases",
      success: false
    )
  end
end
```

#### Appfile
```ruby
# fastlane/Appfile

# Apple ID for App Store Connect
apple_id("developer@padnetxpress.com")

# Team ID for App Store Connect
itc_team_id("YOUR_ITC_TEAM_ID")

# Developer Portal Team ID
team_id("YOUR_TEAM_ID")

# App Bundle Identifiers
app_identifier("com.padnetxpress.app")

# For multiple targets/schemes
for_platform :ios do
  for_lane :staging do
    app_identifier("com.padnetxpress.app.staging")
  end
end
```

#### Matchfile
```ruby
# fastlane/Matchfile

git_url("https://github.com/your-org/ios-certificates")
storage_mode("git")

type("development") # Default type

app_identifier([
  "com.padnetxpress.app",
  "com.padnetxpress.app.staging",
  "com.padnetxpress.app.dev"
])

username("developer@padnetxpress.com")
team_id("YOUR_TEAM_ID")

# Additional configuration
clone_branch_directly(false)
generate_apple_certs(true)
```

## 🧪 TestFlight Deployment

### Internal Testing

```bash
# Deploy to internal testers
fastlane staging

# This will:
# 1. Increment build number
# 2. Build staging configuration
# 3. Upload to TestFlight
# 4. Add to "Internal Team" group
```

### External Beta Testing

```bash
# Deploy to external beta testers
fastlane beta

# This will:
# 1. Increment version and build numbers
# 2. Run comprehensive tests
# 3. Build release configuration
# 4. Upload to TestFlight
# 5. Submit for beta review
# 6. Notify external testers
```

### TestFlight Configuration

#### Beta App Information

Create `fastlane/metadata/review_information/notes.txt`:
```
PadnetXpress is a health monitoring application that helps users track:

1. Heart Rate Monitoring - Real-time tracking with Apple Watch integration
2. Blood Pressure Management - Manual entry and Bluetooth device support
3. Sleep Analysis - Comprehensive sleep quality tracking

BETA TESTING NOTES:
- Test account: beta@padnetxpress.com / BetaTest123!
- All features require iOS 15.0+
- HealthKit permissions will be requested on first launch
- Sample data is available in test mode

PRIVACY:
- All health data is encrypted and stored securely
- No data is shared without explicit user consent
- Full privacy policy available in app

For support: support@padnetxpress.com
```

#### Beta Groups

Configure in App Store Connect:
- **Internal Team** - Developers and QA team
- **Beta Testers** - External beta testers
- **Healthcare Partners** - Medical professionals for feedback

### TestFlight Metadata

```bash
# Generate TestFlight metadata
mkdir -p fastlane/metadata/en-US
echo "PadnetXpress - Health Monitoring" > fastlane/metadata/en-US/name.txt
echo "Track your heart rate, blood pressure, and sleep patterns with PadnetXpress." > fastlane/metadata/en-US/description.txt
```

## 🏪 App Store Submission

### Pre-submission Checklist

- [ ] All features tested thoroughly
- [ ] App metadata updated in App Store Connect
- [ ] Screenshots generated and uploaded
- [ ] Privacy policy updated
- [ ] App Store review guidelines compliance verified
- [ ] Version number incremented appropriately
- [ ] Release notes prepared

### App Store Metadata

#### App Information
```
# App Name
PadnetXpress

# Subtitle
Health Monitoring Made Simple

# Category
Medical

# Content Rating
4+ (Medical/Treatment Information)

# Keywords
health, monitoring, heart rate, blood pressure, sleep, tracking, wellness, medical
```

#### Description Template

Create `fastlane/metadata/en-US/description.txt`:
```
PadnetXpress is your comprehensive health monitoring companion, designed to help you track and understand your vital health metrics with ease and precision.

🫀 HEART RATE MONITORING
• Real-time heart rate tracking
• Apple Watch integration
• Heart rate zone analysis
• Trend visualization and insights

🩺 BLOOD PRESSURE MANAGEMENT
• Easy manual entry interface
• Bluetooth device integration
• Trend analysis and goal tracking
• Medication reminder support

😴 SLEEP ANALYSIS
• Comprehensive sleep tracking
• Sleep quality scoring
• Sleep stage breakdown
• Personalized recommendations

🔒 PRIVACY & SECURITY
• End-to-end encryption
• Local data storage options
• HIPAA-compliant security
• No data sharing without consent

🍎 HEALTHKIT INTEGRATION
• Seamless Apple Health sync
• Two-way data synchronization
• Supports multiple data sources
• Conflict resolution

✨ ADDITIONAL FEATURES
• Beautiful data visualizations
• Export capabilities
• Goal setting and tracking
• Accessibility support

PadnetXpress is designed for wellness tracking and should not replace professional medical advice. Always consult healthcare providers for medical decisions.

Privacy Policy: https://padnetxpress.com/privacy
Terms of Service: https://padnetxpress.com/terms
Support: support@padnetxpress.com
```

### Release Process

```bash
# Full App Store submission
fastlane release

# This will:
# 1. Increment version number
# 2. Run comprehensive tests
# 3. Build production configuration
# 4. Update App Store metadata
# 5. Upload binary to App Store Connect
# 6. Submit for review
# 7. Create GitHub release
```

### App Store Review

#### Review Information

```ruby
# In Fastfile - submission_information
{
  add_id_info_limits_tracking: true,
  add_id_info_serves_ads: false,
  add_id_info_tracks_action: true,
  add_id_info_tracks_install: true,
  add_id_info_uses_idfa: false,
  content_rights_has_rights: true,
  content_rights_contains_third_party_content: false,
  export_compliance_compliance_required: false,
  export_compliance_encryption_updated: false,
  export_compliance_uses_encryption: true,
  export_compliance_is_exempt: false
}
```

#### Review Notes

Create `fastlane/metadata/review_information/notes.txt`:
```
REVIEW NOTES FOR APPLE:

App Overview:
PadnetXpress is a health monitoring application that tracks heart rate, blood pressure, and sleep patterns.

Key Features to Test:
1. Heart Rate Monitoring - Requires HealthKit permissions
2. Blood Pressure Entry - Manual data input
3. Sleep Tracking - HealthKit integration
4. Data Export - PDF and CSV formats

Test Account:
Email: reviewer@padnetxpress.com
Password: AppReview2024!

Special Instructions:
- Grant HealthKit permissions when prompted
- The app requires iOS 15.0+ for full functionality
- Sample data will be available for testing

Health Data Compliance:
- All health data is encrypted and stored securely
- User consent is required for all data collection
- Full privacy policy implemented

Contact: support@padnetxpress.com
```

## 📦 Release Management

### Version Numbering Strategy

PadnetXpress follows semantic versioning:

```
MAJOR.MINOR.PATCH (Build)

Examples:
1.0.0 (1) - Initial release
1.0.1 (2) - Bug fix
1.1.0 (3) - New features
2.0.0 (4) - Major update
```

### Release Types

#### Patch Release (1.0.x)
- Bug fixes
- Performance improvements
- Minor UI adjustments

#### Minor Release (1.x.0)
- New features
- Enhanced functionality
- Non-breaking changes

#### Major Release (x.0.0)
- Significant new features
- UI overhaul
- Breaking changes

### Release Schedule

| Release Type | Frequency | Examples |
|-------------|-----------|----------|
| Patch | As needed | Bug fixes, critical issues |
| Minor | Monthly | Feature updates |
| Major | Quarterly | Major new functionality |

### Release Notes Template

```markdown
# PadnetXpress v1.2.0

## 🆕 New Features
- Enhanced heart rate zone analysis
- Improved sleep quality scoring
- New data export formats

## 🐛 Bug Fixes
- Fixed sync issue with Apple Watch
- Resolved crash on iOS 15.7
- Improved battery usage

## 🔧 Improvements
- Faster app startup time
- Enhanced accessibility support
- Updated privacy controls

## 📊 Technical Updates
- Updated dependencies
- Improved data encryption
- Performance optimizations
```

## 🤖 Automated CI/CD

### GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to App Store

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
        - staging
        - beta
        - production

jobs:
  deploy:
    runs-on: macos-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    
    - name: Setup Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: 3.0
        bundler-cache: true
    
    - name: Setup Xcode
      uses: maxim-lobanov/setup-xcode@v1
      with:
        xcode-version: '15.0'
    
    - name: Install dependencies
      run: |
        gem install fastlane
        sudo gem install cocoapods
        pod install
    
    - name: Setup Fastlane Match
      env:
        MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
        FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
      run: |
        echo "${{ secrets.MATCH_DEPLOY_KEY }}" > /tmp/match_deploy_key
        chmod 600 /tmp/match_deploy_key
        ssh-add /tmp/match_deploy_key
    
    - name: Deploy Staging
      if: github.event.inputs.environment == 'staging'
      env:
        FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
        MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      run: fastlane staging
    
    - name: Deploy Beta
      if: github.event.inputs.environment == 'beta'
      env:
        FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
        MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      run: fastlane beta
    
    - name: Deploy Production
      if: github.event.inputs.environment == 'production'
      env:
        FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.APP_SPECIFIC_PASSWORD }}
        MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      run: fastlane release
    
    - name: Upload Build Artifacts
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: build-logs
        path: |
          fastlane/logs/
          *.ipa
          *.dSYM.zip
```

### Required Secrets

Configure in GitHub repository settings:

```bash
# Apple Developer credentials
FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD
MATCH_PASSWORD

# Certificates repository access
MATCH_DEPLOY_KEY

# Slack integration (optional)
SLACK_WEBHOOK_URL

# GitHub token for releases
GITHUB_TOKEN
```

## 🔧 Troubleshooting

### Common Deployment Issues

#### 1. Certificate Problems

**Issue**: "No matching provisioning profiles found"
```bash
# Solution
fastlane match development --force
fastlane match appstore --force
```

**Issue**: "Certificate has expired"
```bash
# Renew certificates
fastlane match appstore --force_for_new_certificates
```

#### 2. TestFlight Issues

**Issue**: "Missing compliance for encryption"
```ruby
# Add to submission_information
export_compliance_uses_encryption: true,
export_compliance_is_exempt: false
```

**Issue**: "Beta review rejected"
- Check beta app review notes
- Ensure test account credentials are valid
- Verify app functionality works as described

#### 3. App Store Review Issues

**Issue**: "Guideline 2.1 - Performance - App Crashes"
- Review crash logs in Xcode Organizer
- Test on actual devices
- Fix memory leaks and crashes

**Issue**: "Guideline 5.1.1 - Privacy - Data Collection and Storage"
- Update privacy policy
- Review data collection practices
- Ensure proper consent mechanisms

### Emergency Rollback

#### Rollback TestFlight Build
```bash
# Disable problematic build in App Store Connect
# Promote previous stable build to latest
```

#### Rollback App Store Release
```bash
# If app is in review - reject current version
# If already released - prepare hotfix version
fastlane release # with hotfix version
```

### Monitoring and Alerts

#### Build Status Monitoring
```bash
# Setup Slack notifications for build status
# Monitor App Store Connect for review status
# Track crash reports and user feedback
```

#### Performance Monitoring
```bash
# Firebase Crashlytics for crash reporting
# App Store Connect analytics for usage metrics
# Custom analytics for health data insights
```

---

For deployment support and questions, contact: deployment@padnetxpress.com 