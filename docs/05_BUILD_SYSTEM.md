# Build System

## Overview

The Build System is the Builder agent's domain. It takes a spec (one-pager) and produces a complete, compilable Xcode project with SwiftUI views, data models, StoreKit 2 integration, onboarding, and all the polish expected of a production iOS app.

## Build Environment

### Requirements
- **macOS** (required for Xcode and iOS builds)
- **Xcode 16+** (latest stable)
- **iOS 17.0+ deployment target** (for StoreKit 2 SwiftUI views and @Observable)
- **Swift 6.0+**
- **Fastlane** (installed via Homebrew or Bundler)
- **CocoaPods/SPM** (Swift Package Manager preferred for dependencies)

### Why iOS 17.0 Minimum
- `@Observable` macro (replaces `ObservableObject` boilerplate)
- `SubscriptionStoreView` (one-line paywall UI)
- `NavigationStack` improvements
- `SwiftData` (optional, replaces Core Data)
- `TipKit` (for feature discovery)
- iOS 17 has 90%+ adoption as of early 2026

## Project Template

Every app starts from `templates/swiftui/AppTemplate/`:

```
AppTemplate/
├── AppTemplate.xcodeproj/
│   └── project.pbxproj
├── AppTemplate/
│   ├── AppTemplateApp.swift          # @main entry point
│   ├── ContentView.swift             # Root navigation
│   ├── Info.plist                    # Configured per app
│   ├── PrivacyInfo.xcprivacy         # Privacy manifest
│   ├── Assets.xcassets/
│   │   ├── AccentColor.colorset/
│   │   ├── AppIcon.appiconset/
│   │   └── Colors/                   # Custom color assets
│   ├── Models/
│   │   └── .gitkeep
│   ├── Views/
│   │   ├── Onboarding/
│   │   │   └── OnboardingView.swift
│   │   ├── Paywall/
│   │   │   └── PaywallView.swift
│   │   └── Settings/
│   │       └── SettingsView.swift
│   ├── ViewModels/
│   │   └── .gitkeep
│   ├── Services/
│   │   ├── StoreKitService.swift     # StoreKit 2 manager
│   │   └── AnalyticsService.swift    # RevenueCat events
│   ├── Utilities/
│   │   ├── Constants.swift           # App-wide constants
│   │   └── Extensions.swift          # Swift extensions
│   └── Resources/
│       └── Localizable.xcstrings     # String catalog
└── AppTemplateTests/
    └── AppTemplateTests.swift
```

## Code Templates

### Onboarding Template (`Onboarding.swift`)

A configurable `TabView`-based onboarding flow:

```swift
// Template structure (Builder fills in content per spec)
struct OnboardingView: View {
    @AppStorage("hasSeenOnboarding") private var hasSeenOnboarding = false
    @State private var currentPage = 0

    let pages: [OnboardingPage]  // Builder populates from spec

    var body: some View {
        TabView(selection: $currentPage) {
            ForEach(pages.indices, id: \.self) { index in
                OnboardingPageView(page: pages[index])
                    .tag(index)
            }
        }
        .tabViewStyle(.page(indexDisplayMode: .always))
        .overlay(alignment: .bottom) {
            // Continue / Get Started button
        }
    }
}

struct OnboardingPage: Identifiable {
    let id = UUID()
    let imageName: String      // SF Symbol or asset
    let title: String           // Localized
    let description: String     // Localized
    let accentColor: Color
}
```

### Paywall Template (`Paywall.swift`)

StoreKit 2 with `SubscriptionStoreView`:

```swift
// Template structure
struct PaywallView: View {
    let groupID: String  // Set per app

    var body: some View {
        SubscriptionStoreView(groupID: groupID) {
            // Marketing content above the subscription options
            VStack(spacing: 16) {
                Image(systemName: "star.fill")
                    .font(.system(size: 50))
                    .foregroundStyle(.accent)
                Text("Unlock Premium")
                    .font(.title.bold())
                Text("Get access to all features")
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
            .padding()
        }
        .subscriptionStoreButtonLabel(.multiline)
        .subscriptionStorePickerItemBackground(.thinMaterial)
        .storeButton(.visible, for: .restorePurchases)
    }
}
```

### Settings Template (`Settings.swift`)

Standard settings screen with required elements:

```swift
// Template structure
struct SettingsView: View {
    var body: some View {
        List {
            Section("Account") {
                // Subscription status
                // Restore purchases
            }
            Section("Preferences") {
                // App-specific settings
                // Notifications toggle
                // Theme selection
            }
            Section("Support") {
                Link("Privacy Policy", destination: privacyURL)
                Link("Terms of Use", destination: termsURL)
                Link("Contact Support", destination: supportURL)
            }
            Section("About") {
                LabeledContent("Version", value: appVersion)
            }
        }
        .navigationTitle("Settings")
    }
}
```

### Gemini Flash Wrapper (`GeminiWrapper.swift`)

For apps that include AI-powered features:

```swift
// Template structure
actor GeminiService {
    private let apiKey: String
    private let model = "gemini-2.0-flash"
    private let baseURL = "https://generativelanguage.googleapis.com/v1beta"

    init() {
        // API key from environment or secure storage
        // NEVER hardcoded
        self.apiKey = Configuration.geminiAPIKey
    }

    func generate(prompt: String, context: String = "") async throws -> String {
        // Build request
        // Handle streaming response
        // Return generated text
        // Handle errors gracefully
    }

    func generateStream(prompt: String) -> AsyncThrowingStream<String, Error> {
        // Streaming variant for real-time UI updates
    }
}
```

## Coding Standards

The Builder MUST follow these standards. The Reviewer checks compliance.

### 1. No Force Unwraps
```swift
// WRONG
let name = user.name!

// RIGHT
guard let name = user.name else {
    return
}
```

### 2. Localization
```swift
// WRONG
Text("Welcome back")

// RIGHT
Text("welcome_back", tableName: "Localizable")
// Or using String Catalogs:
Text(String(localized: "Welcome back"))
```

### 3. Error Handling
```swift
// WRONG
let data = try! JSONDecoder().decode(Model.self, from: data)

// RIGHT
do {
    let model = try JSONDecoder().decode(Model.self, from: data)
} catch {
    logger.error("Failed to decode: \(error)")
    // Show user-facing error
}
```

### 4. Navigation
```swift
// WRONG: NavigationView (deprecated)
NavigationView { ... }

// RIGHT: NavigationStack with typed destinations
NavigationStack {
    List(items) { item in
        NavigationLink(value: item) {
            ItemRow(item: item)
        }
    }
    .navigationDestination(for: Item.self) { item in
        ItemDetailView(item: item)
    }
}
```

### 5. State Management
```swift
// WRONG: ObservableObject (old pattern)
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
}

// RIGHT: @Observable macro (iOS 17+)
@Observable
class ViewModel {
    var items: [Item] = []
}
```

### 6. Concurrency
```swift
// WRONG: Completion handlers
func fetchData(completion: @escaping (Result<Data, Error>) -> Void) { ... }

// RIGHT: async/await
func fetchData() async throws -> Data { ... }
```

### 7. File Organization
```
Features/
├── Sleep/
│   ├── SleepView.swift
│   ├── SleepViewModel.swift
│   └── SleepModel.swift
├── Routine/
│   ├── RoutineView.swift
│   ├── RoutineViewModel.swift
│   └── RoutineModel.swift
└── Insights/
    ├── InsightsView.swift
    ├── InsightsViewModel.swift
    └── InsightsModel.swift
```

## ICP Bridge (Optional)

For apps that need audit trails or trust scoring, the Builder integrates an ICP canister bridge. This is lightweight — it's an HTTP call to a canister endpoint, not a full IcpKit integration.

### Architecture

```
iOS App → HTTPS → ICP Boundary Node → Canister (Motoko/Rust)
```

Rather than using IcpKit (which has low maintenance activity and unclear Internet Identity support on iOS), we use a **REST API bridge**:

1. Deploy a simple canister that exposes HTTP endpoints via `http_request`
2. The iOS app makes standard `URLSession` HTTPS calls to the canister's endpoint
3. The canister handles Candid serialization internally
4. This avoids any dependency on IcpKit in the Swift code

### What Gets Audited
- App creation timestamp
- Quality scores (immutable once written)
- Submission decisions and reasoning
- Revenue milestones
- Marketing campaign performance

### Canister Design

```motoko
// Simplified audit canister
actor AuditTrail {
    stable var entries: [AuditEntry] = [];

    type AuditEntry = {
        timestamp: Int;
        project_slug: Text;
        event_type: Text;  // "quality_score", "submission", "revenue_milestone"
        data: Text;         // JSON payload
        hash: Text;         // SHA-256 of data for integrity
    };

    public shared func log(entry: AuditEntry) : async () {
        entries := Array.append(entries, [entry]);
    };

    public query func getEntries(slug: Text) : async [AuditEntry] {
        Array.filter(entries, func(e: AuditEntry) : Bool {
            e.project_slug == slug
        });
    };
}
```

## Build Verification

Before the Builder declares a project complete, it must:

1. **Compile check**: Run `xcodebuild build -scheme AppName -destination 'platform=iOS Simulator,name=iPhone 16 Pro Max'` and verify zero errors.
2. **Warning audit**: Review all warnings. Fix any that could cause App Store rejection.
3. **Manifest check**: Verify `PrivacyInfo.xcprivacy` lists all data collection accurately.
4. **Signing check**: Ensure the project's signing configuration references the correct team and provisioning profile.

If compilation fails, the Builder fixes errors and retries within the same session. The session only exits when the project compiles cleanly or the Builder has exhausted its ability to fix errors (in which case it flags the project for human review).

## Dependencies Policy

**Default: Zero external dependencies.**

Every app should use only Apple's frameworks (SwiftUI, StoreKit 2, HealthKit, CloudKit, etc.) unless the spec explicitly requires a third-party integration.

**Approved exceptions:**
- **RevenueCat SDK**: For subscription analytics (added via SPM)
- **Gemini Flash SDK**: For AI features (or raw HTTP if SDK is too heavy)

**Why zero dependencies by default:**
- Faster build times
- Smaller binary size
- No supply chain risk
- No version conflicts
- Easier for the Reviewer to audit
- Apple prefers apps that use their frameworks

## Build Output

The Builder's output is a complete directory:

```
projects/<app-slug>/src/
├── <AppName>.xcodeproj/
├── <AppName>/
│   ├── <AppName>App.swift
│   ├── ContentView.swift
│   ├── Info.plist
│   ├── PrivacyInfo.xcprivacy
│   ├── Assets.xcassets/
│   ├── Features/
│   │   └── ... (feature modules)
│   ├── Views/
│   │   ├── Onboarding/
│   │   ├── Paywall/
│   │   └── Settings/
│   ├── ViewModels/
│   ├── Models/
│   ├── Services/
│   ├── Utilities/
│   └── Resources/
│       └── Localizable.xcstrings
└── <AppName>Tests/
    └── <AppName>Tests.swift
```
