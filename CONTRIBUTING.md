# Contributing to PadnetXpress

Thank you for your interest in contributing to PadnetXpress! This document provides guidelines and information for contributors.

## 📋 Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Code Style Guidelines](#code-style-guidelines)
- [Testing Requirements](#testing-requirements)
- [Pull Request Process](#pull-request-process)
- [Issue Reporting](#issue-reporting)
- [Security Vulnerabilities](#security-vulnerabilities)

## 🤝 Code of Conduct

This project adheres to a code of conduct that we expect all contributors to follow:

- **Be respectful** - Treat everyone with respect and consideration
- **Be inclusive** - Welcome people of all backgrounds and identities
- **Be collaborative** - Work together constructively
- **Be patient** - Remember that people have varying skill levels

## 🚀 Getting Started

### Prerequisites

- Xcode 14.0 or later
- iOS 15.0+ development target
- Swift 5.5+ knowledge
- Git and GitHub account
- CocoaPods installed

### Fork and Clone

1. Fork the repository on GitHub
2. Clone your fork locally:
   ```bash
   git clone https://github.com/YOUR_USERNAME/PadnetXpress.git
   cd PadnetXpress
   ```

3. Add the upstream remote:
   ```bash
   git remote add upstream https://github.com/ORIGINAL_OWNER/PadnetXpress.git
   ```

4. Install dependencies:
   ```bash
   pod install
   open PadnetXpress.xcworkspace
   ```

## 🔄 Development Workflow

### Branch Strategy

We use a simplified Git flow:

- `main` - Production-ready code
- `develop` - Integration branch for features
- `feature/description` - Feature development
- `bugfix/description` - Bug fixes
- `hotfix/description` - Critical production fixes

### Creating a Feature Branch

```bash
git checkout develop
git pull upstream develop
git checkout -b feature/your-feature-name
```

### Keeping Your Branch Updated

```bash
git checkout develop
git pull upstream develop
git checkout feature/your-feature-name
git rebase develop
```

## 📝 Code Style Guidelines

### Swift Style Guide

We follow the [Ray Wenderlich Swift Style Guide](https://github.com/raywenderlich/swift-style-guide) with these additions:

#### Naming Conventions

- **Classes/Structs**: PascalCase (`HealthDataModel`)
- **Variables/Functions**: camelCase (`heartRateValue`)
- **Constants**: camelCase (`maxHeartRate`)
- **Enums**: PascalCase with camelCase cases

```swift
enum HeartRateZone {
    case resting
    case fatBurning
    case aerobic
    case anaerobic
    case maximum
}
```

#### MVVM Architecture Rules

**ViewModels**:
- Suffix with `ViewModel` (`HeartRateViewModel`)
- Use `@Published` for observable properties
- Keep business logic in ViewModels, not Views

```swift
class HeartRateViewModel: ObservableObject {
    @Published var heartRate: Double = 0
    @Published var isMonitoring: Bool = false
    
    private let healthRepository: HealthRepositoryProtocol
    
    func startMonitoring() {
        // Implementation
    }
}
```

**Views**:
- Keep Views lightweight
- Extract complex UI into components
- Use proper SwiftUI modifiers

```swift
struct HeartRateView: View {
    @StateObject private var viewModel = HeartRateViewModel()
    
    var body: some View {
        VStack {
            HeartRateChart(data: viewModel.heartRateData)
            MonitoringButton(isMonitoring: viewModel.isMonitoring) {
                viewModel.toggleMonitoring()
            }
        }
    }
}
```

#### File Organization

```swift
// MARK: - Imports
import SwiftUI
import HealthKit

// MARK: - Main Structure
struct ContentView: View {
    // MARK: - Properties
    @StateObject private var viewModel = ContentViewModel()
    
    // MARK: - Body
    var body: some View {
        // Implementation
    }
}

// MARK: - Preview
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```

### Documentation Standards

- Use Swift documentation comments for public APIs
- Include parameter descriptions and return values
- Provide usage examples for complex functions

```swift
/// Calculates the heart rate zone based on age and current heart rate
/// - Parameters:
///   - heartRate: Current heart rate in BPM
///   - age: User's age in years
/// - Returns: The corresponding heart rate zone
/// - Example:
///   ```swift
///   let zone = calculateHeartRateZone(heartRate: 150, age: 30)
///   ```
func calculateHeartRateZone(heartRate: Double, age: Int) -> HeartRateZone {
    // Implementation
}
```

## 🧪 Testing Requirements

### Unit Testing

- Write unit tests for all ViewModels
- Test business logic and data transformations
- Use dependency injection for testability
- Minimum 80% code coverage for new features

```swift
class HeartRateViewModelTests: XCTestCase {
    var viewModel: HeartRateViewModel!
    var mockRepository: MockHealthRepository!
    
    override func setUp() {
        super.setUp()
        mockRepository = MockHealthRepository()
        viewModel = HeartRateViewModel(healthRepository: mockRepository)
    }
    
    func testStartMonitoring() {
        // Given
        XCTAssertFalse(viewModel.isMonitoring)
        
        // When
        viewModel.startMonitoring()
        
        // Then
        XCTAssertTrue(viewModel.isMonitoring)
        XCTAssertTrue(mockRepository.startMonitoringCalled)
    }
}
```

### UI Testing

- Test critical user flows
- Verify accessibility labels and hints
- Test on different screen sizes

```swift
class PadnetXpressUITests: XCTestCase {
    var app: XCUIApplication!
    
    override func setUp() {
        super.setUp()
        app = XCUIApplication()
        app.launch()
    }
    
    func testHeartRateMonitoringFlow() {
        // Test complete heart rate monitoring flow
        let heartRateTab = app.tabBars.buttons["Heart Rate"]
        heartRateTab.tap()
        
        let startButton = app.buttons["Start Monitoring"]
        XCTAssertTrue(startButton.exists)
        startButton.tap()
        
        // Verify monitoring state
        XCTAssertTrue(app.buttons["Stop Monitoring"].exists)
    }
}
```

### Running Tests

```bash
# Run all tests
cmd+U in Xcode

# Run specific test target
xcodebuild test -workspace PadnetXpress.xcworkspace -scheme PadnetXpress -destination 'platform=iOS Simulator,name=iPhone 14'

# Generate coverage report
Product → Test → Code Coverage → Show Code Coverage
```

## 🔍 Pull Request Process

### Before Submitting

1. **Ensure tests pass**: All unit and UI tests must pass
2. **Update documentation**: Update relevant documentation
3. **Check code coverage**: New code should have appropriate test coverage
4. **Review your changes**: Self-review your code for issues
5. **Update CHANGELOG**: Add your changes to the unreleased section

### PR Title Format

Use conventional commit format:

```
type(scope): description

Examples:
feat(heart-rate): add real-time monitoring
fix(ui): resolve chart rendering issue
docs(readme): update installation instructions
test(heart-rate): add unit tests for ViewModel
```

### PR Description Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update
- [ ] Performance improvement

## Testing
- [ ] Unit tests added/updated
- [ ] UI tests added/updated
- [ ] Manual testing completed

## Screenshots
<!-- If UI changes, include before/after screenshots -->

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Tests added and passing
- [ ] Documentation updated
- [ ] CHANGELOG.md updated
```

### Review Process

1. **Automated checks**: CI/CD pipeline runs tests and linting
2. **Code review**: At least one team member reviews
3. **Testing**: QA testing for significant features
4. **Approval**: PR approved by maintainer
5. **Merge**: Squash and merge to maintain clean history

## 🐛 Issue Reporting

### Bug Reports

Use the bug report template:

```markdown
## Bug Description
Clear description of the bug

## Steps to Reproduce
1. Go to '...'
2. Tap on '....'
3. See error

## Expected Behavior
What you expected to happen

## Screenshots
If applicable, add screenshots

## Environment
- iOS version:
- Device:
- App version:

## Additional Context
Any other context about the problem
```

### Feature Requests

```markdown
## Feature Description
Clear description of the proposed feature

## Problem Statement
What problem does this solve?

## Proposed Solution
How would you like it to work?

## Alternatives Considered
Other solutions you've considered

## Additional Context
Mockups, examples, or references
```

## 🔒 Security Vulnerabilities

**Do not** report security vulnerabilities through public GitHub issues.

Instead, please email security@padnetxpress.com with:

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We'll respond within 48 hours and work with you to resolve the issue.

## 🎖️ Recognition

Contributors will be recognized in:

- Release notes for significant contributions
- Contributors section in README
- Special thanks in app credits

## 📞 Questions?

- Create a discussion for general questions
- Join our development chat (link provided separately)
- Email: dev@padnetxpress.com

Thank you for contributing to PadnetXpress! 🎉 