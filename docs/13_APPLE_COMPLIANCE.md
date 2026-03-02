# Apple Compliance

## The Core Risk: Guideline 4.3 (Spam)

Apple's Guideline 4.3 states that apps which "duplicate the content and functionality of other apps submitted by you or another developer" will be rejected. This is the single biggest risk to the app factory model.

**If Apple flags your developer account as a spammer:**
- All future submissions get automatic rejections
- Every app requires manual re-review (adding weeks to review times)
- Repeated violations lead to removal from the Apple Developer Program
- This is effectively a death sentence for the factory

This guide exists to prevent that outcome.

## Differentiation Strategy

### The Rule: Every App Must Be Genuinely Different

Not "slightly different." Not "different colors." **Genuinely different** in a way that an Apple reviewer would recognize within 30 seconds of opening the app.

### Differentiation Checklist

Every app in the factory must differ from every other AppFactory app on ALL of these dimensions:

| Dimension | What "Different" Means | Example |
|-----------|----------------------|---------|
| **Core value proposition** | Solves a fundamentally different problem | Sleep routines ≠ Expense tracking |
| **Target user** | Different persona with different needs | Parents ≠ Students ≠ Professionals |
| **Primary framework** | Different Apple frameworks used | HealthKit ≠ StoreKit ≠ CloudKit |
| **Navigation pattern** | Different UI structure | Tab bar ≠ Sidebar ≠ Single-scroll |
| **Color scheme** | Different primary and accent colors | Blue/white ≠ Green/dark ≠ Orange/cream |
| **Icon style** | Different visual approach | Geometric ≠ Organic ≠ Abstract |
| **Feature set** | No overlapping core features | Habit tracking ≠ Budget tracking ≠ Timer |

### Category Diversification

The factory maintains a **maximum of 2 apps per App Store category**:

| Category | Max Apps | Why |
|----------|----------|-----|
| Health & Fitness | 2 | Large category, many distinct sub-niches (sleep, fitness, nutrition) |
| Productivity | 2 | Broad (timers, planners, notes) |
| Personal Finance | 2 | Specific (budgeting, expense tracking) |
| Lifestyle | 2 | Very broad (cooking, gardening, home) |
| Education | 2 | Distinct sub-niches (study, language, flashcards) |
| Utilities | 2 | Simple tools, high variety |

**Even within the same category**, apps must solve different problems. Two fitness apps are fine if one tracks workouts and the other plans meals. Two workout trackers are not fine.

### The Differentiation Matrix

Maintained in `factory.db`, the differentiation matrix tracks every shipping/shipped app:

```sql
CREATE TABLE app_catalog (
    slug TEXT PRIMARY KEY,
    app_name TEXT NOT NULL,
    category TEXT NOT NULL,
    subcategory TEXT,
    primary_color TEXT,
    secondary_color TEXT,
    nav_pattern TEXT,        -- 'tab_bar', 'sidebar', 'single_scroll', 'navigation_stack'
    icon_style TEXT,         -- 'geometric', 'organic', 'abstract', 'gradient'
    primary_framework TEXT,  -- 'healthkit', 'storekit', 'cloudkit', 'coredata', etc.
    core_features TEXT,      -- JSON array of feature tags
    target_persona TEXT,
    status TEXT              -- 'building', 'live', 'killed'
);
```

Before any new app enters the build phase, the Architect checks:
1. No other app in the catalog has the same `subcategory`
2. No other app uses the same `primary_color`
3. No other app uses the same `nav_pattern` + `category` combination
4. No other app has overlapping `core_features` (Jaccard similarity < 0.2)
5. No other app targets the same `target_persona`

If any check fails, the Architect must adjust the spec to ensure differentiation.

## Apple Review Guidelines: Key Rules

### Rules That Apply to Every App

| Guideline | What It Requires | How We Comply |
|-----------|-----------------|---------------|
| 2.1 App Completeness | App must be complete and functional | Quality Gate 3 (Feature Completeness) |
| 2.3 Accurate Metadata | Description must match functionality | Shipper verifies metadata against spec |
| 3.1.1 In-App Purchase | Use Apple's IAP for digital goods | StoreKit 2 template, no external payment links |
| 3.1.2 Subscriptions | Clear pricing, terms, and cancellation | Paywall template shows all terms |
| 4.0 Design | App must be polished and functional | Quality Gate 4 (UI/UX Quality) |
| 4.2 Minimum Functionality | Must provide value beyond a website | Research phase validates the need for native |
| 4.3 Spam | Must not duplicate other apps | Differentiation strategy (this document) |
| 5.1.1 Data Collection | Must disclose all data collection | Privacy manifest verification in Gate 2 |
| 5.1.2 Data Use and Sharing | Must get consent for AI data sharing | Template includes consent flow |

### Rules Specific to AI-Powered Apps

Since November 2025, Apple requires:
1. **Explicit disclosure** of any third-party AI service used (OpenAI, Google, Anthropic)
2. **User consent** before sharing personal data with AI services
3. **Clear indication** of what data is sent to AI and how it's processed
4. **Option to use the app** without AI features (for privacy-conscious users)

Our Gemini Flash wrapper template handles all of these:
- Disclosure in the Privacy manifest
- Consent dialog on first AI feature use
- Data minimization (only send what's necessary)
- Graceful degradation when AI is unavailable or declined

### Rules Specific to Subscriptions

| Requirement | How We Comply |
|-------------|---------------|
| Display price before purchase | SubscriptionStoreView shows pricing automatically |
| Show auto-renewal terms | Template includes "Renews automatically" text |
| Restore Purchases button | Settings template includes restore functionality |
| Free trial terms | Template shows "7-day free trial, then $X.XX/month" |
| Easy cancellation info | Settings links to Apple's subscription management |
| No misleading free claims | Paywall clearly shows what's free vs. premium |

## Privacy Manifest (PrivacyInfo.xcprivacy)

Every app must include a complete Privacy manifest. This is the most common reason for automated rejection.

### Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <!-- Add entries based on actual data collection -->
    </array>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <!-- Required reason APIs -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string><!-- Access within app group -->
            </array>
        </dict>
    </array>
</dict>
</plist>
```

### Required Reason APIs

If the app uses ANY of these APIs, it must declare a reason:

| API Category | Common Use | Required Reason Code |
|-------------|-----------|---------------------|
| UserDefaults | Storing settings, onboarding state | CA92.1 (app container) |
| File timestamp | File modification dates | DDA9.1 (display to user) |
| System boot time | Analytics timing | 35F9.1 (elapsed time measurement) |
| Disk space | Storage management | E174.1 (display to user) |
| Active keyboards | Input method detection | 54BD.1 (custom keyboard) |

The Builder adds the appropriate entries based on which APIs the app uses. The Reviewer verifies completeness.

## Rejection Recovery

### Common Rejections and Automated Fixes

| Rejection Code | Meaning | Automated Fix |
|---------------|---------|---------------|
| 2.1 | App crashes or has bugs | Send back to Builder with crash report |
| 2.3 | Metadata doesn't match app | Shipper rewrites listing |
| 3.1.1 | Payment issue | Check StoreKit configuration |
| 4.3 | Spam / duplicate | Requires human review (may need to kill the app) |
| 5.1.1 | Privacy manifest incomplete | Builder adds missing entries |
| 5.1.2 | AI data sharing not disclosed | Builder adds consent flow |

### 4.3 Rejection Response Protocol

If an app is rejected for Guideline 4.3:

1. **DO NOT resubmit immediately.** This will aggravate the issue.
2. **Analyze the rejection notes.** Apple will specify what they consider a duplicate.
3. **If it's a cross-catalog issue** (too similar to our other apps):
   - Assess whether the app can be made more distinct
   - If yes: redesign UI, change navigation pattern, adjust feature set, resubmit with an appeal explaining the differences
   - If no: kill the project and learn from it
4. **If it's an external duplicate** (too similar to someone else's app):
   - Reassess the differentiation angle
   - Consider pivoting the feature set to create genuine uniqueness
   - If the market is truly saturated, kill the project
5. **Update the differentiation matrix** to prevent future similar issues

### Appeal Template

If we believe the rejection is incorrect:

```
Dear App Review Team,

Thank you for reviewing [App Name]. We'd like to address the Guideline 4.3 concern.

[App Name] is fundamentally different from [cited similar app] in the following ways:

1. Core Value Proposition: [Our app does X while the other does Y]
2. Target Audience: [We serve X demographic while they serve Y]
3. Feature Set: [List 3-5 features unique to our app]
4. Technical Implementation: [We use X framework while they use Y]

We've also included screenshots highlighting these differences for your reference.

We believe [App Name] provides unique value to App Store users and respectfully request a re-review.

Thank you for your time.
```

## Proactive Compliance Measures

1. **Pre-submission compliance scan**: The Reviewer's Gate 5 simulates an Apple review
2. **Metadata review**: The Shipper runs `fastlane precheck` to catch common issues
3. **Build with latest SDK**: Always use the latest Xcode and iOS SDK to avoid "update required" rejections
4. **Test on real devices**: Use TestFlight to verify on physical devices when possible
5. **Screenshot accuracy**: Screenshots must show the actual app, not mockups
6. **Content rating accuracy**: If the app allows user-generated content or has AI features, rate appropriately (usually 12+ or 17+)

## Account Health

### Monitoring

Track account health metrics:
- Rejection rate (should be < 20% of submissions)
- Average review time (increasing times may indicate account scrutiny)
- Number of appeals (more than 2/month is a warning sign)
- Review team notes (any mention of "pattern" or "similar" is a red flag)

### If Account Health Degrades

1. **Slow down submissions**: Reduce from 5 concurrent to 2-3
2. **Increase quality threshold**: Raise the pass score from 8.0 to 9.0
3. **Manual review all specs**: Require human approval before every build
4. **Diversify harder**: Ensure maximum differentiation between apps
5. **Contact Apple Developer support**: Proactively explain your development approach

The factory's long-term success depends on maintaining a healthy relationship with Apple's review team. This is a marathon, not a sprint.
