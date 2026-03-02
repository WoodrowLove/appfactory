# Quality Gates

## Philosophy

Quality gates exist for one reason: to prevent Apple rejections and ensure users get apps worth paying for. Every app that leaves the factory must be indistinguishable from a hand-crafted indie app. If it feels auto-generated, it fails.

An app needs a score of **8.0/10 or higher** to proceed past review. Three consecutive failures flag the project for human intervention. The threshold is intentionally high — it's cheaper to fix issues before submission than to deal with Apple rejections and resubmission delays.

## Pre-Review Lint Pass

Before the Codex Reviewer is spawned, a **Haiku-based Linter agent** performs a fast, cheap static analysis pass over the Builder's output. This pass checks for:

| Check | Severity |
|-------|----------|
| Force unwraps (`!`) | Critical flag |
| Missing `NS*UsageDescription` strings for used permissions | Critical flag |
| Dead code (unused imports, unreachable functions) | Warning |
| Placeholder text ("Lorem ipsum", "TODO", "FIXME" visible in UI strings) | Critical flag |

**Decision logic:**
- If the Linter finds **>5 total issues** or **any critical flag**, the build is sent straight back to the Builder with the Linter's report — the Codex Reviewer is never spawned.
- If the Linter finds 5 or fewer non-critical issues, those are attached as advisory notes to the Reviewer's input context, and review proceeds normally.

**Why this matters:** Codex review tokens are expensive. Catching obvious problems with a fast Haiku pass before invoking the full Reviewer **cuts review failure cost by ~40%**. Most first-attempt Builder outputs have at least one force unwrap or placeholder string — catching these early saves a full review cycle.

## Incremental Review

On **revision attempts 2+** (after a previous review failure), the Reviewer only re-checks the gates that previously failed, not all 8 scored gates. Gates that scored 8 or above on the prior attempt are marked as **locked-pass** and skipped.

**How it works:**
1. The Router reads the previous `quality.json` and identifies which gates scored below 8.
2. Only those gates (plus Gate 0: Compilation, which always runs) are included in the Reviewer's prompt.
3. The Reviewer outputs scores only for re-checked gates; locked-pass scores carry forward.
4. The overall score is recalculated from the combination of locked-pass and new scores.

**Impact:** This cuts revision review time by **50-60%** and reduces token spend proportionally. It also focuses the Reviewer's attention on the specific areas that need improvement, producing more targeted feedback.

## The Gates

### Gate 0: Compilation (pass/fail)

**Automated pre-review gate — run by the Router, not the Reviewer.**

After the Builder completes and outputs `src/`, the Router runs `xcodebuild build` independently before spawning the Reviewer. This is a binary pass/fail check — the code either compiles or it doesn't.

**What runs:**
```bash
xcodebuild build \
  -scheme AppName \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=latest' \
  -quiet \
  2>&1
```

**Decision logic:**
- **Pass (exit code 0):** Compilation succeeded. The Router proceeds to spawn the Reviewer for Gates 1-8.
- **Fail (non-zero exit code):** Compilation failed. The Router sends the compiler error output back to the Builder immediately. The Reviewer is never spawned.

**Why this gate exists:** There is no point spending Codex tokens reviewing code that doesn't compile. A surprising number of first-attempt Builder outputs have minor syntax errors, missing imports, or type mismatches. Catching these before review saves the cost of an entire Reviewer invocation. Unlike Gates 1-8, this gate is not scored 0-10 — it is strictly binary and is **not included in the overall score average**.

### Gate 1: Crash Safety (0-10)

**What the Reviewer checks:**

| Check | Severity | Example |
|-------|----------|---------|
| Force unwraps (`!`) | Critical | `let name = user.name!` → crash if nil |
| Unhandled optionals | High | `array[index]` without bounds check |
| Retain cycles | High | Closures capturing `self` without `[weak self]` |
| Unhandled async errors | High | `try await` without `do/catch` |
| Implicit main thread violations | High | UI updates from background thread |
| Division by zero risks | Medium | Any division without zero-check |
| Array index access | Medium | Direct subscript without bounds validation |
| Force cast (`as!`) | Medium | `object as! SpecificType` |

**Scoring:**
- 10: Zero issues found
- 9: 1-2 minor issues (medium severity only)
- 8: 3-5 minor issues, zero high/critical
- 7: 1-2 high severity issues
- 6: 3+ high severity issues or 1 critical
- 5 or below: Multiple critical issues — immediate fail

### Gate 2: Permission Handling (0-10)

**What the Reviewer checks:**

| Check | Severity | Notes |
|-------|----------|-------|
| Info.plist permission strings | Critical | Every `NS*UsageDescription` key must have a clear, specific, non-generic description |
| Permission timing | High | Must be requested at point of use, never at launch |
| Denial handling | High | App must function (with reduced features) if permission denied |
| Unnecessary permissions | High | Only request what the app actually uses |
| PrivacyInfo.xcprivacy | Critical | Must accurately list all data collection, tracking, and API usage |
| Required reason APIs | Critical | Any use of UserDefaults, file timestamp, disk space, etc. must declare required reasons |

**Scoring:**
- 10: All permissions perfectly handled
- 9: Minor wording improvements possible but compliant
- 8: One permission requested too early (fixable)
- 7: Missing required reason API declaration (Apple will reject)
- 6 or below: Missing privacy manifest entries or unnecessary permissions — fail

**Why this gate matters:** Apple has been aggressively rejecting apps with incomplete privacy manifests since iOS 17. This is the most common automated rejection reason.

### Gate 3: Feature Completeness (0-10)

**What the Reviewer checks:**

The Reviewer cross-references `onepager.json` against the codebase:

| Check | Method |
|-------|--------|
| Every core feature implemented | Compare `features.core` array against Views/ |
| Every premium feature implemented | Compare `features.premium` array against Views/ + paywall gating |
| Every screen exists | Compare `screens` array against navigation flow |
| All screens are reachable | Trace navigation paths from ContentView |
| Onboarding matches spec | Compare onboarding content against `screens` where type = "onboarding" |
| Paywall implemented | Verify StoreKit 2 integration exists and is functional |
| Settings screen complete | Check for privacy policy link, restore purchases, version info |
| Empty states | Every list/collection view has an empty state |
| Loading states | Every async operation shows loading indicator |
| Error states | Every network/database call has error handling UI |

**Scoring:**
- 10: Every feature from the spec is implemented with all states handled
- 9: All features present, 1-2 minor polish items missing (empty state on a secondary screen)
- 8: All core features present, 1 premium feature has partial implementation
- 7: 1 core feature missing or incomplete
- 6 or below: Multiple features missing — fail

### Gate 4: UI/UX Quality (0-10)

**What the Reviewer checks:**

| Check | Severity | Notes |
|-------|----------|-------|
| Consistent spacing | Medium | Margins, padding should follow a consistent grid |
| Consistent typography | Medium | Font sizes and weights should follow a clear hierarchy |
| Color consistency | Medium | Same semantic colors used throughout (accent, background, text) |
| Dark mode | High | Every screen must look correct in both light and dark mode |
| Dynamic Type | High | Text must scale with system font size settings |
| VoiceOver labels | High | All interactive elements must have accessibility labels |
| Loading indicators | Medium | Users must know when something is loading |
| Pull-to-refresh | Minor | Where data can change, pull-to-refresh should be available |
| Keyboard handling | Medium | Keyboard dismissal, scroll-to-avoid, proper input types |
| Safe area respect | Medium | Content shouldn't be clipped by notch, home indicator, or status bar |
| Scroll behavior | Minor | Long lists should scroll smoothly, no janky animations |
| Haptic feedback | Minor | Key interactions should have appropriate haptic response |

**Scoring:**
- 10: Pixel-perfect, feels like an Apple first-party app
- 9: Minor spacing inconsistencies or missing haptics
- 8: Dark mode or Dynamic Type has 1-2 issues
- 7: Missing VoiceOver labels on multiple screens
- 6 or below: Broken dark mode or major layout issues — fail

### Gate 5: App Store Compliance (0-10)

This is the most important gate. A perfect app with compliance issues will get rejected by Apple.

**What the Reviewer checks:**

| Check | Severity | Notes |
|-------|----------|-------|
| No private API usage | Critical | Any use of undocumented Apple APIs = instant rejection |
| No competitor platform references | High | No mentions of "Android", "Google Play" in UI or strings |
| Privacy manifest complete | Critical | All data types, tracking domains, and API reasons declared |
| Subscription terms visible | Critical | Price, duration, and renewal terms must be shown before purchase |
| Restore Purchases button | Critical | Must be present and functional in settings or paywall |
| Free trial terms clear | High | Trial length and what happens after must be explicit |
| Age rating appropriate | Medium | Content must match the declared age rating |
| No placeholder content | Critical | No "Lorem ipsum", TODO comments visible to users, or debug text |
| No web view wrapping | High | App must provide native functionality, not just wrap a website |
| Minimum functionality | High | App must do something meaningful beyond what a website could do |
| **Differentiation from our catalog** | Critical | Must look and feel distinct from every other AppFactory app |
| Metadata accuracy | High | App description must match actual functionality |

**Differentiation sub-check (critical):**
The Reviewer compares the current app against all other AppFactory apps in `projects/`:
- Different primary color scheme
- Different navigation pattern (tab bar vs. sidebar vs. single-screen)
- Different onboarding content
- Different app icon style
- Different core feature set
- If any two apps could be confused as "the same app with different content," this gate fails

**Scoring:**
- 10: Fully compliant, no issues
- 9: Minor metadata improvement possible
- 8: One non-critical compliance item to fix
- 7: Missing restore purchases or unclear subscription terms (Apple will reject)
- 6 or below: Private API usage, missing privacy manifest, or differentiation failure — hard fail

### Gate 6: Code Quality (0-10)

**What the Reviewer checks:**

| Check | Severity | Notes |
|-------|----------|-------|
| No dead code | Medium | Unused functions, imports, or variables |
| Naming conventions | Medium | camelCase for variables, PascalCase for types, consistent throughout |
| Separation of concerns | High | No business logic in views |
| MVVM pattern | High | View → ViewModel → Model separation |
| Swift concurrency | Medium | async/await preferred over callbacks |
| File organization | Medium | Feature-module structure, not flat directory |
| Comments on complex logic | Medium | Non-obvious algorithms should have explaining comments |
| No hardcoded strings | Medium | All user-facing strings should use localization |
| No hardcoded colors | Minor | Colors should reference named color assets |
| Test coverage | Minor | At minimum, model layer should have unit tests |

**Scoring:**
- 10: Clean, idiomatic Swift code that any developer would be proud of
- 9: Minor style inconsistencies
- 8: Some business logic in views, but contained
- 7: Significant separation of concerns violations
- 6 or below: Spaghetti code, no pattern followed — fail

### Gate 7: Test Coverage (0-10)

**What the Reviewer checks:**

The Reviewer verifies that meaningful tests exist and that coverage meets minimum thresholds.

| Check | Severity | Notes |
|-------|----------|-------|
| Model layer test files exist | Critical | Every file in `Models/` must have a corresponding `*Tests.swift` |
| Line coverage >= 60% | High | Measured via `xcodebuild test` with code coverage enabled on the model layer |
| Tests actually assert something | High | Tests must contain meaningful assertions, not just `XCTAssertTrue(true)` |
| ViewModel tests for core flows | Medium | At minimum, the primary user flow ViewModel should have tests |
| Edge case coverage | Medium | Nil inputs, empty arrays, boundary values tested |
| No test-only hacks | Medium | Production code should not contain `#if TEST` workarounds or test-specific backdoors |

**How coverage is measured:**

```bash
# Build and run tests with code coverage enabled (for .xcodeproj projects)
xcodebuild test \
  -scheme <AppName> \
  -destination 'platform=iOS Simulator,name=iPhone 16 Pro Max' \
  -enableCodeCoverage YES \
  -resultBundlePath TestResults.xcresult

# Extract coverage report from the result bundle
xcrun llvm-cov report \
  $(find ~/Library/Developer/Xcode/DerivedData -name "<AppName>" -type f | head -1) \
  -instr-profile $(find ~/Library/Developer/Xcode/DerivedData -name "Coverage.profdata" -type f | head -1) \
  -sources Models/
```

**Scoring:**
- 10: >= 80% model coverage, ViewModels tested, edge cases covered
- 9: >= 70% model coverage, core ViewModel tested
- 8: >= 60% model coverage, all Model files have test files
- 7: >= 60% model coverage, but 1-2 Model files missing test files
- 6: < 60% model coverage or multiple Model files untested
- 5 or below: No meaningful tests, or tests are all stubs — fail

### Gate 8: Monetization UX (0-10)

**What the Reviewer checks:**

Monetization must feel fair, transparent, and Apple-compliant. Users should never feel tricked into a subscription.

| Check | Severity | Notes |
|-------|----------|-------|
| Annual savings clearly presented | Critical | Paywall must show the percentage or dollar amount saved by choosing annual over monthly |
| Annual plan visually emphasized | High | Annual plan should be the default selection or visually prominent (larger card, highlighted border, "Best Value" badge) |
| Trial terms unambiguous | Critical | If a free trial is offered, the exact duration and what happens after ("$X.XX/month after 7-day free trial") must be visible before the CTA button |
| Contextual paywall triggers | High | Paywalls should trigger when a user taps a premium feature, not only during onboarding. At minimum, tapping any premium-gated feature must show the paywall |
| Restore Purchases feedback | High | "Restore Purchases" button must show clear feedback: spinner during restore, success confirmation with restored items listed, or "No purchases found" message |
| Subscription management link | Medium | Settings must include a working link to iOS subscription management (`itms-apps://apps.apple.com/account/subscriptions`) |
| No dark patterns | Critical | No pre-selected premium plan in a way that could be missed, no tiny cancel text, no confusing double-negatives in subscription prompts |
| Price formatting | Medium | Prices must be localized and fetched from StoreKit (never hardcoded strings) |

**Scoring:**
- 10: Paywall is clear, fair, and polished — annual savings shown, trial terms explicit, restore works perfectly
- 9: Minor improvements possible (e.g., savings percentage could be more prominent)
- 8: One non-critical issue (e.g., restore purchases lacks a success message, but doesn't crash)
- 7: Contextual paywalls missing (only shows on onboarding) or trial terms are ambiguous
- 6: Annual savings not shown or annual plan not emphasized — Apple may reject or users will churn
- 5 or below: Dark patterns present, missing restore purchases, or hardcoded prices — fail

## Scoring Calculation

The overall score is the **average of all 8 scored gates (1-8), rounded down to one decimal place**.

Gate 0 (Compilation) is a binary pass/fail gate and is **not included** in the average. If Gate 0 fails, review does not proceed, so no scored gates are evaluated.

```
overall = floor(mean([gate1, gate2, gate3, gate4, gate5, gate6, gate7, gate8]) * 10) / 10
```

**Examples:**
- Gates: [9, 10, 8, 9, 10, 8, 8, 9] → mean = 8.875 → **PASS** (8.8)
- Gates: [9, 10, 7, 9, 10, 8, 9, 8] → mean = 8.75 → **PASS** (8.7)
- Gates: [8, 9, 7, 8, 8, 7, 7, 7] → mean = 7.625 → **FAIL** (7.6)
- Gates: [10, 10, 10, 10, 5, 10, 9, 9] → mean = 9.125 → **FAIL** (9.1 average but blocked — Gate 5 scored 5, triggering a blocking issue that must be resolved regardless of overall score)

**Important:** Even if the average passes, any gate with a score of 5 or below generates a **blocking issue** that must be resolved regardless of the overall score.

## Failure Protocol

### First failure (attempt 1 of 3)
1. Reviewer writes `quality.json` with detailed issue descriptions
2. Router sends quality report to Builder
3. Builder receives: original spec + current source + quality feedback
4. Builder fixes flagged issues without breaking existing functionality
5. Builder outputs updated `src/`
6. Router sends back to Reviewer

### Second failure (attempt 2 of 3)
Same process. The Builder now has two sets of feedback to work from.

### Third failure (attempt 3 of 3)
1. Project transitions to `flagged` state
2. Human receives notification in Control Panel
3. All three quality reports are presented alongside the source code and spec
4. Human decides:
   - **Fix manually**: Edit the code yourself
   - **Restart from spec**: The Architect rewrites the spec with adjusted scope
   - **Kill**: Document the failure and free up the project slot

### Why 3 attempts?
- Most issues are fixable on the first revision
- The second revision catches regression issues from the first fix
- If an app can't pass after 3 rounds of feedback, the problem is likely in the spec (wrong scope, infeasible feature), not the code

## Quality Over Time

The factory tracks quality scores in `factory.db`:

```sql
CREATE TABLE quality_history (
    id INTEGER PRIMARY KEY,
    project_slug TEXT NOT NULL,
    attempt INTEGER NOT NULL,
    gate_0_pass BOOLEAN,          -- Compilation: binary pass/fail
    gate_1_score REAL,
    gate_2_score REAL,
    gate_3_score REAL,
    gate_4_score REAL,
    gate_5_score REAL,
    gate_6_score REAL,
    gate_7_score REAL,
    gate_8_score REAL,
    overall_score REAL,           -- Average of gates 1-8 only
    pass BOOLEAN,
    lint_pass BOOLEAN,            -- Pre-review lint pass result
    incremental_review BOOLEAN,   -- Whether this was an incremental review
    timestamp TEXT NOT NULL
);
```

This data feeds the Control Panel's quality trends view, showing:
- Average first-attempt pass rate
- Most common failure gates
- Score trends over time (are we getting better?)
- Per-app quality comparison
