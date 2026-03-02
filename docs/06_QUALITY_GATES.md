# Quality Gates

## Philosophy

Quality gates exist for one reason: to prevent Apple rejections and ensure users get apps worth paying for. Every app that leaves the factory must be indistinguishable from a hand-crafted indie app. If it feels auto-generated, it fails.

An app needs a score of **8.0/10 or higher** to proceed past review. Three consecutive failures flag the project for human intervention. The threshold is intentionally high — it's cheaper to fix issues before submission than to deal with Apple rejections and resubmission delays.

## The 6 Gates

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
| Pull-to-refresh | Low | Where data can change, pull-to-refresh should be available |
| Keyboard handling | Medium | Keyboard dismissal, scroll-to-avoid, proper input types |
| Safe area respect | Medium | Content shouldn't be clipped by notch, home indicator, or status bar |
| Scroll behavior | Low | Long lists should scroll smoothly, no janky animations |
| Haptic feedback | Low | Key interactions should have appropriate haptic response |

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
| No hardcoded colors | Low | Colors should reference named color assets |
| Test coverage | Low | At minimum, model layer should have unit tests |

**Scoring:**
- 10: Clean, idiomatic Swift code that any developer would be proud of
- 9: Minor style inconsistencies
- 8: Some business logic in views, but contained
- 7: Significant separation of concerns violations
- 6 or below: Spaghetti code, no pattern followed — fail

## Scoring Calculation

The overall score is the **average of all 6 gates, rounded down to one decimal place**.

```
overall = floor(mean([gate1, gate2, gate3, gate4, gate5, gate6]) * 10) / 10
```

**Examples:**
- Gates: [9, 10, 8, 9, 10, 8] → mean = 9.0 → **PASS**
- Gates: [9, 10, 7, 9, 10, 8] → mean = 8.83 → **PASS** (8.8)
- Gates: [8, 9, 7, 8, 8, 7] → mean = 7.83 → **FAIL** (7.8)
- Gates: [10, 10, 10, 10, 5, 10] → mean = 9.16 → **PASS** (9.1) — but the 5 in App Store Compliance likely has critical issues that should be addressed

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
    gate_1_score REAL,
    gate_2_score REAL,
    gate_3_score REAL,
    gate_4_score REAL,
    gate_5_score REAL,
    gate_6_score REAL,
    overall_score REAL,
    pass BOOLEAN,
    timestamp TEXT NOT NULL
);
```

This data feeds the Control Panel's quality trends view, showing:
- Average first-attempt pass rate
- Most common failure gates
- Score trends over time (are we getting better?)
- Per-app quality comparison
