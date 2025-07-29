# Testing Guide

This comprehensive guide covers testing strategies, implementation patterns, and best practices for PadnetXpress. Our testing approach ensures reliability, maintainability, and confidence in health-critical functionality.

## 📋 Table of Contents

- [Testing Strategy](#testing-strategy)
- [Test Structure](#test-structure)
- [Unit Testing](#unit-testing)
- [Integration Testing](#integration-testing)
- [UI Testing](#ui-testing)
- [Mocking & Test Doubles](#mocking--test-doubles)
- [Coverage Reporting](#coverage-reporting)
- [Performance Testing](#performance-testing)
- [Accessibility Testing](#accessibility-testing)
- [CI/CD Integration](#cicd-integration)

## 🎯 Testing Strategy

### Testing Pyramid

```mermaid
pyramid
    title PadnetXpress Testing Strategy
    "UI Tests (10%)" : 10
    "Integration Tests (30%)" : 30
    "Unit Tests (60%)" : 60
```

### Test Distribution Goals

| Test Type | Percentage | Focus Area | Execution Speed |
|-----------|------------|------------|----------------|
| **Unit Tests** | 60% | Business logic, ViewModels, Utils | Fast (< 1s) |
| **Integration Tests** | 30% | API integration, HealthKit, Services | Medium (1-10s) |
| **UI Tests** | 10% | Critical user flows, Accessibility | Slow (10s+) |

### Quality Gates

- **Minimum Code Coverage**: 80%
- **Critical Path Coverage**: 95%
- **Zero Tolerance**: Memory leaks, crashes, security vulnerabilities
- **Performance**: All tests complete within 10 minutes

## 🏗️ Test Structure

### Project Organization

```
PadnetXpressTests/
├── UnitTests/
│   ├── ViewModels/
│   │   ├── HeartRateViewModelTests.swift
│   │   ├── BloodPressureViewModelTests.swift
│   │   └── SleepViewModelTests.swift
│   ├── Repositories/
│   │   ├── HealthRepositoryTests.swift
│   │   └── UserRepositoryTests.swift
│   ├── Services/
│   │   ├── NetworkServiceTests.swift
│   │   ├── EncryptionServiceTests.swift
│   │   └── HealthKitServiceTests.swift
│   └── Utils/
│       ├── DateFormatterTests.swift
│       └── ValidationTests.swift
├── IntegrationTests/
│   ├── API/
│   │   ├── AuthenticationIntegrationTests.swift
│   │   └── HealthDataAPITests.swift
│   ├── HealthKit/
│   │   └── HealthKitIntegrationTests.swift
│   └── Database/
│       └── CoreDataIntegrationTests.swift
├── UITests/
│   ├── Flows/
│   │   ├── OnboardingUITests.swift
│   │   ├── HeartRateMonitoringUITests.swift
│   │   └── BloodPressureUITests.swift
│   ├── Accessibility/
│   │   └── AccessibilityUITests.swift
│   └── Performance/
│       └── PerformanceUITests.swift
├── Mocks/
│   ├── MockHealthRepository.swift
│   ├── MockNetworkService.swift
│   └── MockHealthKitService.swift
└── TestUtilities/
    ├── TestData.swift
    ├── TestHelpers.swift
    └── XCTestCase+Extensions.swift
```

### Naming Conventions

```swift
// Test class naming
class [ClassName]Tests: XCTestCase { }

// Test method naming
func test[MethodName]_[Scenario]_[ExpectedResult]() { }

// Examples
func testStartMonitoring_WhenHealthKitAuthorized_ShouldBeginDataCollection()
func testCalculateHeartRateZone_WithValidAge_ReturnsCorrectZone()
func testLoginViewModel_WithInvalidCredentials_ShowsErrorMessage()
```

## 🧪 Unit Testing

### ViewModel Testing

ViewModels contain the core business logic and should have comprehensive test coverage.

#### Example: HeartRateViewModel Tests

```swift
import XCTest
import Combine
@testable import PadnetXpress

class HeartRateViewModelTests: XCTestCase {
    
    // MARK: - Properties
    var sut: HeartRateViewModel!
    var mockHealthRepository: MockHealthRepository!
    var mockNotificationService: MockNotificationService!
    var cancellables: Set<AnyCancellable>!
    
    // MARK: - Setup & Teardown
    override func setUp() {
        super.setUp()
        mockHealthRepository = MockHealthRepository()
        mockNotificationService = MockNotificationService()
        sut = HeartRateViewModel(
            healthRepository: mockHealthRepository,
            notificationService: mockNotificationService
        )
        cancellables = Set<AnyCancellable>()
    }
    
    override func tearDown() {
        cancellables = nil
        sut = nil
        mockHealthRepository = nil
        mockNotificationService = nil
        super.tearDown()
    }
    
    // MARK: - Test Cases
    func testStartMonitoring_WhenHealthKitAuthorized_ShouldBeginDataCollection() {
        // Given
        mockHealthRepository.authorizationStatus = .authorized
        let expectation = expectation(description: "Monitoring started")
        
        sut.$isMonitoring
            .dropFirst()
            .sink { isMonitoring in
                if isMonitoring {
                    expectation.fulfill()
                }
            }
            .store(in: &cancellables)
        
        // When
        sut.startMonitoring()
        
        // Then
        waitForExpectations(timeout: 1.0)
        XCTAssertTrue(sut.isMonitoring)
        XCTAssertTrue(mockHealthRepository.startHeartRateQueryCalled)
    }
    
    func testStopMonitoring_WhenCurrentlyMonitoring_ShouldStopDataCollection() {
        // Given
        sut.startMonitoring()
        XCTAssertTrue(sut.isMonitoring)
        
        // When
        sut.stopMonitoring()
        
        // Then
        XCTAssertFalse(sut.isMonitoring)
        XCTAssertTrue(mockHealthRepository.stopHeartRateQueryCalled)
    }
    
    func testCalculateHeartRateZone_WithValidAge_ReturnsCorrectZone() {
        // Given
        let age = 30
        let heartRate = 150.0
        let expectedZone = HeartRateZone.aerobic
        
        // When
        let result = sut.calculateHeartRateZone(heartRate: heartRate, age: age)
        
        // Then
        XCTAssertEqual(result, expectedZone)
    }
    
    func testHandleAbnormalReading_WhenHeartRateTooHigh_TriggersAlert() {
        // Given
        let abnormalHeartRate = 200.0
        let expectation = expectation(description: "Alert triggered")
        
        sut.$showingAlert
            .dropFirst()
            .sink { showingAlert in
                if showingAlert {
                    expectation.fulfill()
                }
            }
            .store(in: &cancellables)
        
        // When
        sut.handleNewHeartRateReading(abnormalHeartRate)
        
        // Then
        waitForExpectations(timeout: 1.0)
        XCTAssertTrue(mockNotificationService.sendAlertCalled)
        XCTAssertEqual(sut.alertMessage, "Abnormally high heart rate detected")
    }
}
```

### Repository Testing

Test data access and transformation logic.

```swift
class HealthRepositoryTests: XCTestCase {
    
    var sut: HealthRepository!
    var mockHealthKitService: MockHealthKitService!
    var mockNetworkService: MockNetworkService!
    var mockStorageService: MockStorageService!
    
    override func setUp() {
        super.setUp()
        mockHealthKitService = MockHealthKitService()
        mockNetworkService = MockNetworkService()
        mockStorageService = MockStorageService()
        
        sut = HealthRepository(
            healthKitService: mockHealthKitService,
            networkService: mockNetworkService,
            storageService: mockStorageService
        )
    }
    
    func testFetchHeartRateData_WhenDataAvailable_ReturnsCorrectData() {
        // Given
        let expectedData = [
            HeartRateReading(value: 72, timestamp: Date()),
            HeartRateReading(value: 75, timestamp: Date().addingTimeInterval(-3600))
        ]
        mockHealthKitService.heartRateData = expectedData
        
        let expectation = expectation(description: "Data fetched")
        var result: [HeartRateReading]?
        
        // When
        sut.fetchHeartRateData(for: Date().addingTimeInterval(-86400))
            .sink(
                receiveCompletion: { _ in },
                receiveValue: { data in
                    result = data
                    expectation.fulfill()
                }
            )
            .store(in: &cancellables)
        
        // Then
        waitForExpectations(timeout: 1.0)
        XCTAssertEqual(result?.count, 2)
        XCTAssertEqual(result?.first?.value, 72)
    }
}
```

### Service Testing

Test external service integrations and error handling.

```swift
class NetworkServiceTests: XCTestCase {
    
    var sut: NetworkService!
    var mockURLSession: MockURLSession!
    
    override func setUp() {
        super.setUp()
        mockURLSession = MockURLSession()
        sut = NetworkService(session: mockURLSession)
    }
    
    func testLogin_WithValidCredentials_ReturnsAuthToken() {
        // Given
        let credentials = LoginCredentials(email: "test@example.com", password: "password")
        let expectedResponse = AuthResponse(
            accessToken: "valid_token",
            refreshToken: "refresh_token",
            expiresIn: 3600
        )
        
        mockURLSession.data = try! JSONEncoder().encode(expectedResponse)
        mockURLSession.response = HTTPURLResponse(
            url: URL(string: "https://api.example.com")!,
            statusCode: 200,
            httpVersion: nil,
            headerFields: nil
        )
        
        let expectation = expectation(description: "Login completed")
        var result: AuthResponse?
        
        // When
        sut.login(credentials: credentials)
            .sink(
                receiveCompletion: { _ in },
                receiveValue: { response in
                    result = response
                    expectation.fulfill()
                }
            )
            .store(in: &cancellables)
        
        // Then
        waitForExpectations(timeout: 1.0)
        XCTAssertEqual(result?.accessToken, "valid_token")
    }
    
    func testLogin_WithNetworkError_ReturnsError() {
        // Given
        let credentials = LoginCredentials(email: "test@example.com", password: "password")
        mockURLSession.error = NetworkError.connectionFailed
        
        let expectation = expectation(description: "Error received")
        var receivedError: Error?
        
        // When
        sut.login(credentials: credentials)
            .sink(
                receiveCompletion: { completion in
                    if case .failure(let error) = completion {
                        receivedError = error
                        expectation.fulfill()
                    }
                },
                receiveValue: { _ in }
            )
            .store(in: &cancellables)
        
        // Then
        waitForExpectations(timeout: 1.0)
        XCTAssertNotNil(receivedError)
    }
}
```

## 🔗 Integration Testing

### HealthKit Integration

```swift
class HealthKitIntegrationTests: XCTestCase {
    
    var healthKitService: HealthKitService!
    
    override func setUp() {
        super.setUp()
        healthKitService = HealthKitService()
    }
    
    func testRequestAuthorization_WhenUserGrants_ReturnsSuccess() {
        // Note: This test requires actual HealthKit simulator
        let expectation = expectation(description: "Authorization completed")
        var authorizationGranted = false
        
        healthKitService.requestAuthorization { success, error in
            authorizationGranted = success
            expectation.fulfill()
        }
        
        waitForExpectations(timeout: 5.0)
        XCTAssertTrue(authorizationGranted)
    }
    
    func testFetchHeartRateData_WithValidDateRange_ReturnsData() {
        // Given
        let startDate = Calendar.current.date(byAdding: .day, value: -7, to: Date())!
        let endDate = Date()
        
        let expectation = expectation(description: "Data fetched")
        var heartRateData: [HeartRateReading] = []
        
        // When
        healthKitService.fetchHeartRateData(from: startDate, to: endDate) { data, error in
            heartRateData = data ?? []
            expectation.fulfill()
        }
        
        // Then
        waitForExpectations(timeout: 10.0)
        // Note: Data availability depends on simulator/device state
        XCTAssertNotNil(heartRateData)
    }
}
```

### API Integration Testing

```swift
class HealthDataAPIIntegrationTests: XCTestCase {
    
    var apiClient: APIClient!
    
    override func setUp() {
        super.setUp()
        // Use test environment
        apiClient = APIClient(baseURL: "https://test-api.padnetxpress.com")
    }
    
    func testSyncHeartRateData_WithValidToken_CompletesSuccessfully() {
        // Given
        let testToken = "test_jwt_token"
        let heartRateData = [
            HeartRateReading(value: 72, timestamp: Date())
        ]
        
        let expectation = expectation(description: "Sync completed")
        var syncSuccessful = false
        
        // When
        apiClient.syncHeartRateData(heartRateData, token: testToken) { success, error in
            syncSuccessful = success
            expectation.fulfill()
        }
        
        // Then
        waitForExpectations(timeout: 30.0)
        XCTAssertTrue(syncSuccessful)
    }
}
```

## 📱 UI Testing

### Critical Flow Testing

```swift
class HeartRateMonitoringUITests: XCTestCase {
    
    var app: XCUIApplication!
    
    override func setUp() {
        super.setUp()
        app = XCUIApplication()
        app.launchArguments.append("--uitesting")
        app.launch()
    }
    
    func testHeartRateMonitoringFlow_FromDashboardToResults() {
        // Navigate to Heart Rate tab
        let heartRateTab = app.tabBars.buttons["Heart Rate"]
        XCTAssertTrue(heartRateTab.exists)
        heartRateTab.tap()
        
        // Start monitoring
        let startButton = app.buttons["Start Monitoring"]
        XCTAssertTrue(startButton.exists)
        startButton.tap()
        
        // Verify monitoring state
        let stopButton = app.buttons["Stop Monitoring"]
        XCTAssertTrue(stopButton.waitForExistence(timeout: 2.0))
        
        // Check live heart rate display
        let heartRateDisplay = app.staticTexts["heart-rate-value"]
        XCTAssertTrue(heartRateDisplay.exists)
        
        // Stop monitoring
        stopButton.tap()
        
        // Verify return to idle state
        XCTAssertTrue(startButton.waitForExistence(timeout: 2.0))
        
        // Check that data was recorded
        let historyButton = app.buttons["View History"]
        historyButton.tap()
        
        let historyList = app.tables["heart-rate-history"]
        XCTAssertTrue(historyList.exists)
    }
    
    func testBloodPressureEntry_ManualInput_SavesCorrectly() {
        // Navigate to Blood Pressure tab
        app.tabBars.buttons["Blood Pressure"].tap()
        
        // Tap add reading button
        app.buttons["Add Reading"].tap()
        
        // Enter systolic value
        let systolicField = app.textFields["Systolic"]
        systolicField.tap()
        systolicField.typeText("120")
        
        // Enter diastolic value
        let diastolicField = app.textFields["Diastolic"]
        diastolicField.tap()
        diastolicField.typeText("80")
        
        // Save reading
        app.buttons["Save Reading"].tap()
        
        // Verify reading appears in list
        let readingCell = app.cells.containing(.staticText, identifier: "120/80").firstMatch
        XCTAssertTrue(readingCell.waitForExistence(timeout: 2.0))
    }
}
```

### Page Object Pattern

```swift
class DashboardPage {
    let app: XCUIApplication
    
    init(app: XCUIApplication) {
        self.app = app
    }
    
    // MARK: - Elements
    var healthSummaryCard: XCUIElement {
        app.otherElements["health-summary-card"]
    }
    
    var heartRateQuickAction: XCUIElement {
        app.buttons["quick-action-heart-rate"]
    }
    
    var trendsChart: XCUIElement {
        app.otherElements["trends-chart"]
    }
    
    // MARK: - Actions
    func tapHeartRateQuickAction() {
        heartRateQuickAction.tap()
    }
    
    func waitForDashboardToLoad() -> Bool {
        return healthSummaryCard.waitForExistence(timeout: 5.0)
    }
    
    // MARK: - Assertions
    func verifyHealthDataDisplayed() {
        XCTAssertTrue(healthSummaryCard.exists)
        XCTAssertTrue(trendsChart.exists)
    }
}

// Usage in tests
class DashboardUITests: XCTestCase {
    var app: XCUIApplication!
    var dashboardPage: DashboardPage!
    
    override func setUp() {
        super.setUp()
        app = XCUIApplication()
        app.launch()
        dashboardPage = DashboardPage(app: app)
    }
    
    func testDashboardLoadsCorrectly() {
        XCTAssertTrue(dashboardPage.waitForDashboardToLoad())
        dashboardPage.verifyHealthDataDisplayed()
    }
}
```

## 🎭 Mocking & Test Doubles

### Repository Mocks

```swift
class MockHealthRepository: HealthRepositoryProtocol {
    
    // MARK: - Call Tracking
    var fetchHeartRateDataCalled = false
    var saveHeartRateDataCalled = false
    var startHeartRateQueryCalled = false
    var stopHeartRateQueryCalled = false
    
    // MARK: - Configurable Responses
    var heartRateDataToReturn: [HeartRateReading] = []
    var errorToReturn: Error?
    var authorizationStatus: HealthKitAuthorizationStatus = .authorized
    
    // MARK: - Protocol Implementation
    func fetchHeartRateData(from startDate: Date, to endDate: Date) -> AnyPublisher<[HeartRateReading], Error> {
        fetchHeartRateDataCalled = true
        
        if let error = errorToReturn {
            return Fail(error: error).eraseToAnyPublisher()
        }
        
        return Just(heartRateDataToReturn)
            .setFailureType(to: Error.self)
            .eraseToAnyPublisher()
    }
    
    func saveHeartRateData(_ data: [HeartRateReading]) -> AnyPublisher<Void, Error> {
        saveHeartRateDataCalled = true
        
        if let error = errorToReturn {
            return Fail(error: error).eraseToAnyPublisher()
        }
        
        return Just(()).setFailureType(to: Error.self).eraseToAnyPublisher()
    }
    
    func startHeartRateQuery() -> AnyPublisher<HeartRateReading, Error> {
        startHeartRateQueryCalled = true
        
        // Simulate real-time data
        return Timer.publish(every: 1.0, on: .main, in: .default)
            .autoconnect()
            .map { _ in
                HeartRateReading(
                    value: Double.random(in: 60...100),
                    timestamp: Date()
                )
            }
            .setFailureType(to: Error.self)
            .eraseToAnyPublisher()
    }
    
    func stopHeartRateQuery() {
        stopHeartRateQueryCalled = true
    }
}
```

### Network Service Mocks

```swift
class MockNetworkService: NetworkServiceProtocol {
    
    // MARK: - Call Tracking
    var loginCalled = false
    var syncDataCalled = false
    var fetchUserProfileCalled = false
    
    // MARK: - Configurable Responses
    var authResponseToReturn: AuthResponse?
    var userProfileToReturn: UserProfile?
    var errorToReturn: NetworkError?
    var networkDelay: TimeInterval = 0.1
    
    // MARK: - Protocol Implementation
    func login(credentials: LoginCredentials) -> AnyPublisher<AuthResponse, NetworkError> {
        loginCalled = true
        
        return Future { promise in
            DispatchQueue.main.asyncAfter(deadline: .now() + self.networkDelay) {
                if let error = self.errorToReturn {
                    promise(.failure(error))
                } else if let response = self.authResponseToReturn {
                    promise(.success(response))
                } else {
                    promise(.failure(.invalidResponse))
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    func syncHealthData(_ data: [HealthDataPoint]) -> AnyPublisher<Void, NetworkError> {
        syncDataCalled = true
        
        if let error = errorToReturn {
            return Fail(error: error).eraseToAnyPublisher()
        }
        
        return Just(())
            .delay(for: .seconds(networkDelay), scheduler: DispatchQueue.main)
            .setFailureType(to: NetworkError.self)
            .eraseToAnyPublisher()
    }
}
```

### Test Data Builders

```swift
class TestDataBuilder {
    
    static func heartRateReading(
        value: Double = 72.0,
        timestamp: Date = Date(),
        source: String = "test"
    ) -> HeartRateReading {
        return HeartRateReading(
            value: value,
            timestamp: timestamp,
            source: source
        )
    }
    
    static func bloodPressureReading(
        systolic: Int = 120,
        diastolic: Int = 80,
        timestamp: Date = Date()
    ) -> BloodPressureReading {
        return BloodPressureReading(
            systolic: systolic,
            diastolic: diastolic,
            timestamp: timestamp
        )
    }
    
    static func sleepData(
        bedTime: Date = Date(),
        wakeTime: Date = Date().addingTimeInterval(8 * 3600),
        efficiency: Double = 0.85
    ) -> SleepData {
        return SleepData(
            bedTime: bedTime,
            wakeTime: wakeTime,
            efficiency: efficiency
        )
    }
}

// Usage
let testHeartRate = TestDataBuilder.heartRateReading(value: 85.0)
let testBP = TestDataBuilder.bloodPressureReading(systolic: 130, diastolic: 85)
```

## 📊 Coverage Reporting

### Xcode Code Coverage

1. **Enable Code Coverage**
   ```
   Product → Scheme → Edit Scheme → Test → Options → Code Coverage (Check "Gather coverage for some targets")
   ```

2. **Run Tests with Coverage**
   ```bash
   xcodebuild test -workspace PadnetXpress.xcworkspace -scheme PadnetXpress -destination 'platform=iOS Simulator,name=iPhone 14' -enableCodeCoverage YES
   ```

3. **View Coverage Report**
   ```
   Product → Test → Code Coverage → Show Code Coverage
   ```

### Command Line Coverage

```bash
# Generate coverage report
xcodebuild test \
  -workspace PadnetXpress.xcworkspace \
  -scheme PadnetXpress \
  -destination 'platform=iOS Simulator,name=iPhone 14' \
  -enableCodeCoverage YES \
  -resultBundlePath TestResults.xcresult

# Convert to readable format
xcrun xccov view --report TestResults.xcresult
```

### Coverage Targets

| Component | Target Coverage | Critical Areas |
|-----------|----------------|----------------|
| ViewModels | 95% | Business logic, state management |
| Repositories | 90% | Data access, error handling |
| Services | 85% | External integrations |
| Utils/Extensions | 80% | Helper functions |
| UI Components | 70% | SwiftUI views (where testable) |

### Coverage Exclusions

```swift
// Coverage exclusions using comments
// swiftlint:disable:next function_body_length
func largeFunction() {
    // MARK: - Coverage Ignore
    // This section handles edge cases that are difficult to test
    // coverage:ignore:start
    if someRareCondition {
        handleRareCase()
    }
    // coverage:ignore:end
}
```

## ⚡ Performance Testing

### Measure Block Testing

```swift
class PerformanceTests: XCTestCase {
    
    func testHeartRateCalculationPerformance() {
        let heartRateProcessor = HeartRateProcessor()
        let testData = Array(1...1000).map { _ in
            Double.random(in: 60...120)
        }
        
        measure {
            for heartRate in testData {
                _ = heartRateProcessor.calculateZone(heartRate: heartRate, age: 30)
            }
        }
    }
    
    func testDataEncryptionPerformance() {
        let encryptionService = EncryptionService()
        let testData = TestDataBuilder.generateLargeHealthDataSet(count: 1000)
        
        measure {
            do {
                _ = try encryptionService.encrypt(testData)
            } catch {
                XCTFail("Encryption failed: \(error)")
            }
        }
    }
}
```

### Memory Leak Testing

```swift
class MemoryLeakTests: XCTestCase {
    
    func testViewModelMemoryLeak() {
        weak var weakViewModel: HeartRateViewModel?
        
        autoreleasepool {
            let mockRepository = MockHealthRepository()
            let viewModel = HeartRateViewModel(healthRepository: mockRepository)
            weakViewModel = viewModel
            
            viewModel.startMonitoring()
            viewModel.stopMonitoring()
        }
        
        XCTAssertNil(weakViewModel, "ViewModel should be deallocated")
    }
}
```

## ♿ Accessibility Testing

### VoiceOver Testing

```swift
class AccessibilityTests: XCTestCase {
    
    func testHeartRateViewAccessibility() {
        let app = XCUIApplication()
        app.launch()
        
        // Navigate to Heart Rate tab
        let heartRateTab = app.tabBars.buttons["Heart Rate"]
        XCTAssertNotNil(heartRateTab.label)
        XCTAssertNotNil(heartRateTab.accessibilityHint)
        
        heartRateTab.tap()
        
        // Test monitoring button accessibility
        let startButton = app.buttons["Start Monitoring"]
        XCTAssertEqual(startButton.accessibilityLabel, "Start heart rate monitoring")
        XCTAssertEqual(startButton.accessibilityHint, "Double tap to begin monitoring your heart rate")
        XCTAssertEqual(startButton.accessibilityTraits, .button)
        
        // Test heart rate display accessibility
        let heartRateDisplay = app.staticTexts["heart-rate-value"]
        XCTAssertTrue(heartRateDisplay.accessibilityLabel?.contains("beats per minute") ?? false)
    }
    
    func testDynamicTypeSupport() {
        let app = XCUIApplication()
        
        // Test with different Dynamic Type sizes
        let typeSizes: [UIContentSizeCategory] = [
            .small,
            .medium,
            .large,
            .extraLarge,
            .accessibilityMedium,
            .accessibilityExtraLarge
        ]
        
        for typeSize in typeSizes {
            app.launchArguments = ["--dynamic-type-size", typeSize.rawValue]
            app.launch()
            
            // Verify text is readable and UI elements are accessible
            let heartRateTab = app.tabBars.buttons["Heart Rate"]
            XCTAssertTrue(heartRateTab.exists)
            
            app.terminate()
        }
    }
}
```

## 🔄 CI/CD Integration

### GitHub Actions Configuration

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: macos-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Xcode
      uses: maxim-lobanov/setup-xcode@v1
      with:
        xcode-version: '15.0'
    
    - name: Install CocoaPods
      run: |
        sudo gem install cocoapods
        pod install
    
    - name: Run Unit Tests
      run: |
        xcodebuild test \
          -workspace PadnetXpress.xcworkspace \
          -scheme PadnetXpress \
          -destination 'platform=iOS Simulator,name=iPhone 14' \
          -testPlan UnitTests \
          -enableCodeCoverage YES \
          -resultBundlePath TestResults.xcresult
    
    - name: Run UI Tests
      run: |
        xcodebuild test \
          -workspace PadnetXpress.xcworkspace \
          -scheme PadnetXpress \
          -destination 'platform=iOS Simulator,name=iPhone 14' \
          -testPlan UITests
    
    - name: Generate Coverage Report
      run: |
        xcrun xccov view --report TestResults.xcresult > coverage.txt
        cat coverage.txt
    
    - name: Upload Coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage.txt
        fail_ci_if_error: true
```

### Test Plans

Create test plans for different scenarios:

```json
// UnitTests.xctestplan
{
  "configurations" : [
    {
      "id" : "unit-tests",
      "name" : "Unit Tests",
      "options" : {
        "codeCoverage" : true,
        "testExecutionOrdering" : "random"
      }
    }
  ],
  "defaultOptions" : {
    "testExecutionOrdering" : "random"
  },
  "testTargets" : [
    {
      "target" : {
        "containerPath" : "container:PadnetXpress.xcodeproj",
        "identifier" : "PadnetXpressTests",
        "name" : "PadnetXpressTests"
      }
    }
  ],
  "version" : 1
}
```

### Test Scripts

```bash
#!/bin/bash
# scripts/run_tests.sh

set -e

echo "🧪 Running PadnetXpress Tests"

# Clean build folder
echo "🧹 Cleaning build folder..."
xcodebuild clean -workspace PadnetXpress.xcworkspace -scheme PadnetXpress

# Install dependencies
echo "📦 Installing dependencies..."
pod install

# Run unit tests
echo "🔬 Running unit tests..."
xcodebuild test \
  -workspace PadnetXpress.xcworkspace \
  -scheme PadnetXpress \
  -destination 'platform=iOS Simulator,name=iPhone 14' \
  -testPlan UnitTests \
  -enableCodeCoverage YES \
  -resultBundlePath UnitTestResults.xcresult

# Run integration tests
echo "🔗 Running integration tests..."
xcodebuild test \
  -workspace PadnetXpress.xcworkspace \
  -scheme PadnetXpress \
  -destination 'platform=iOS Simulator,name=iPhone 14' \
  -testPlan IntegrationTests

# Run UI tests
echo "📱 Running UI tests..."
xcodebuild test \
  -workspace PadnetXpress.xcworkspace \
  -scheme PadnetXpress \
  -destination 'platform=iOS Simulator,name=iPhone 14' \
  -testPlan UITests

# Generate coverage report
echo "📊 Generating coverage report..."
xcrun xccov view --report UnitTestResults.xcresult

echo "✅ All tests completed successfully!"
```

## 🚀 Running Tests

### Xcode IDE

1. **Run all tests**: `Cmd+U`
2. **Run specific test**: Click play button next to test method
3. **Run test class**: Click play button next to class name
4. **Debug test**: Set breakpoint and run test

### Command Line

```bash
# Run all tests
xcodebuild test -workspace PadnetXpress.xcworkspace -scheme PadnetXpress -destination 'platform=iOS Simulator,name=iPhone 14'

# Run specific test target
xcodebuild test -workspace PadnetXpress.xcworkspace -scheme PadnetXpress -destination 'platform=iOS Simulator,name=iPhone 14' -only-testing:PadnetXpressTests

# Run specific test class
xcodebuild test -workspace PadnetXpress.xcworkspace -scheme PadnetXpress -destination 'platform=iOS Simulator,name=iPhone 14' -only-testing:PadnetXpressTests/HeartRateViewModelTests

# Run with code coverage
xcodebuild test -workspace PadnetXpress.xcworkspace -scheme PadnetXpress -destination 'platform=iOS Simulator,name=iPhone 14' -enableCodeCoverage YES
```

### FastLane Integration

```ruby
# fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "Run all tests"
  lane :test do
    run_tests(
      workspace: "PadnetXpress.xcworkspace",
      scheme: "PadnetXpress",
      device: "iPhone 14",
      code_coverage: true
    )
  end
  
  desc "Run unit tests only"
  lane :unit_tests do
    run_tests(
      workspace: "PadnetXpress.xcworkspace",
      scheme: "PadnetXpress",
      device: "iPhone 14",
      testplan: "UnitTests"
    )
  end
end
```

---

For testing support and questions, contact: testing@padnetxpress.com 