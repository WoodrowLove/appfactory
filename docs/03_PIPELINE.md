# Pipeline

## Overview

The AppFactory pipeline has 10 phases: RESEARCH, VALIDATE, SPEC, BUILD, LINT, REVIEW, MONETIZE, PACKAGE, SHIP, and MARKET. Each phase is owned by one agent. The orchestrator manages transitions between phases. Every phase reads from files and writes to files. No state is stored in conversation history.

Total time from idea to App Store submission (estimated): **4-8 hours of compute time** spread across the pipeline, depending on complexity and review cycles.

## Phase 1: Research

**Agent:** Scout (Sonnet 4.6)
**Duration:** 15-30 minutes
**Input:** Category focus, subreddit list, competitor exclusion list, existing AppFactory app catalog
**Output:** `research.json`
**Gate:** Produces at least 3 scored ideas per research cycle

### What Happens

1. **Pain point discovery**: The Scout crawls target subreddits (r/productivity, r/iphone, r/sleep, r/fitness, r/personalfinance, etc.) looking for recurring complaints, broken workflows, and feature requests that existing apps don't address.

2. **Signal extraction from X/Twitter**: Search for phrases like "I wish there was an app", "this app sucks because", "why doesn't [category] have", and "switched from [app] because".

3. **Product Hunt scanning**: Check trending categories for emerging needs and validated demand.

4. **App Store review mining**: For high-download apps in target categories, analyze 1-star and 2-star reviews. These are goldmines — users telling you exactly what's broken and what they'd pay for.

5. **Idea ranking**: Each idea gets three scores:
   - **Demand (1-10)**: How many people are complaining? How often? How passionate?
   - **Competition (1-10)**: How many competitors exist? How good are they? When were they last updated? (Higher score = less competition)
   - **Feasibility (1-10)**: Can this be built as a standalone SwiftUI app? Does it need a server? Does it need hardware access we can't provide?

6. **Output**: A ranked list of ideas in `research.json` with full source attribution (Reddit post URLs, tweet IDs, review excerpts).

### Research Categories

The factory focuses on categories with proven App Store revenue:

| Category | Why |
|----------|-----|
| Health & Fitness | Highest willingness to pay for subscriptions. Sleep, fitness, nutrition, meditation. |
| Productivity | Large market, repeat users, low churn. Timers, planners, habit trackers. |
| Personal Finance | High perceived value. Budget trackers, savings goals, expense categorizers. |
| Lifestyle | Broad category. Home organization, cooking, gardening, pet care. |
| Education | Growing market. Study aids, flashcards, language practice. |
| Utilities | Everyday tools people need. QR scanners, unit converters, file managers. |

Categories we **avoid**:
- Social networking (requires network effects we can't bootstrap)
- Games (entirely different development pipeline)
- Navigation/Maps (dominated by Google/Apple)
- Messaging (requires server infrastructure and critical mass)

---

## Phase 2: Validate

**Agent:** Scout (Sonnet 4.6, validation mode)
**Duration:** 10-20 minutes per idea
**Input:** Single idea from `research.json`, existing app catalog, Apple Search Ads data
**Output:** Updated `research.json` with validation scores
**Gate:** Four-tier scoring: >= 24 fast-track, 21-23 proceed, 18-20 borderline (human review), < 18 kill

### What Happens

1. **Competitor deep-dive**: For the specific idea, search App Store for every direct competitor. Record name, rating, review count, last update date, price, subscription model.

2. **Keyword analysis**: Use Apple Search Ads API (or Search Ads Intelligence tools) to check keyword difficulty and estimated search volume for the app's primary keywords.

3. **Guideline 4.3 check**: Compare the proposed app against every other app in the AppFactory catalog. If any two apps could be seen as "similar" by an Apple reviewer, flag it. This is the most important check.

4. **Technical feasibility audit**: Confirm that the app can be built with SwiftUI + the approved frameworks (HealthKit, CloudKit, StoreKit 2, etc.). Identify any technical risks.

5. **Monetization validation**: Is the proposed price point realistic for this category? Check what competitors charge. Estimate conversion rate based on category benchmarks.

6. **Tiered Go/No-Go decision**: The combined score (demand + competition + feasibility) determines the path forward:
   - **>= 24: Fast-track** — High-confidence idea. Proceed immediately to Spec with priority scheduling.
   - **21-23: Proceed normally** — Solid idea. Proceed to Spec in the standard queue.
   - **18-20: Borderline** — Flag for human review. You review the research data and decide whether to proceed, revise, or kill.
   - **< 18: Kill** — Insufficient signal. Goes to the "killed" pile with a documented reason.

### Why the tiered threshold

A score of 21 means the idea scores at least 7/10 on average across all three dimensions, which is the minimum bar for automated progression. The four-tier system adds nuance:
- **Fast-track (24+)** rewards exceptional ideas with faster pipeline throughput
- **Borderline (18-20)** prevents premature kills on ideas that might succeed with refinement or human insight
- **Kill (< 18)** filters out clearly unviable ideas without wasting your time

Examples:
- Ideas with high demand but impossible feasibility (score: 10 + 2 + 8 = 20, borderline — human review)
- Ideas that are easy to build but nobody wants (score: 3 + 9 + 9 = 21, proceed normally)
- Ideas in oversaturated markets (score: 8 + 3 + 8 = 19, borderline — human review)
- Strong ideas with clear demand and low competition (score: 9 + 8 + 8 = 25, fast-track)

---

## Phase 3: Spec

**Agent:** Architect (Opus 4.6)
**Duration:** 20-40 minutes
**Input:** Validated `research.json`, spec templates, existing app catalog
**Output:** `spec.md` (human-readable one-pager), `onepager.json` (machine-readable)
**Gate:** Human approval (optional — can auto-approve if confidence is high)

### What Happens

1. **Persona development**: Define the target user in detail. Not "people who want better sleep" but "Health-conscious professionals, 28-42, who track their fitness but struggle with sleep quality. They've tried Sleep Cycle but found it only tracks, doesn't help build habits."

2. **Feature scoping**: Map each pain point to a specific feature. Core features (free tier) solve the primary pain point. Premium features (paid tier) add AI personalization, advanced analytics, and customization.

3. **Screen design**: Define every screen in the app with its purpose, key UI elements, and user flow. This isn't wireframing — it's a written description precise enough for the Builder to implement without ambiguity.

4. **Onboarding flow**: Design 3-5 onboarding screens that:
   - Screen 1: Value proposition (what this app does for you)
   - Screen 2: Key benefit demonstration
   - Screen 3: Personalization setup (optional)
   - Screen 4: Notification/permission request (with context)
   - Screen 5: Get started CTA

5. **Monetization design**: Decide between freemium subscription (default), one-time purchase, or consumable IAP. For subscriptions, define what's free vs. premium, trial length (default 7 days), and pricing.

6. **Differentiation statement**: Explicitly state how this app differs from (a) every competitor found in research and (b) every other AppFactory app. This statement is used by the Reviewer in Gate 5.

7. **Technical specification**: List required frameworks, minimum iOS version, data storage approach, API integrations, and estimated complexity.

### The One-Pager as Contract

The spec is a **contract** between the Architect and the Builder. Once approved (by human review or auto-approval), the Builder is bound to implement exactly what's specified. If the Builder encounters an issue (e.g., a specified feature isn't technically possible), it must document the deviation in its output and flag it for review.

This prevents scope creep, feature additions that weren't validated, and subjective design decisions by the Builder.

---

## Phase 4: Build

**Agent:** Builder (Opus 4.6)
**Duration:** 30-90 minutes (depends on complexity)
**Input:** `spec.md`, `onepager.json`, SwiftUI templates, coding standards document
**Output:** `src/` (complete Xcode project)
**Gate:** Compilation (the project must build without errors)

### What Happens

1. **Project scaffolding**: Create the Xcode project structure from the base template. Set up the target, bundle identifier, Info.plist, and build settings.

2. **Model layer**: Define data models for the app's domain (e.g., `SleepRoutine`, `RoutineStep`, `SleepEntry`). Use `@Observable` macro for state management.

3. **Persistence layer**: Implement data storage — typically SwiftData (Core Data successor) for local-first apps, or CloudKit for sync.

4. **View layer**: Build every screen from the spec. Use NavigationStack for routing, proper sheet/alert presentation, and consistent styling.

5. **Onboarding**: Implement the 3-5 onboarding screens from the spec using the onboarding template. Track completion in UserDefaults.

6. **Paywall**: Integrate StoreKit 2 using the paywall template. Configure product IDs, subscription group, and free trial offer.

7. **AI integration** (if specified): Wrap Gemini Flash API using the GeminiWrapper template. Handle API key securely (not hardcoded), implement error states, and provide fallback behavior when offline.

8. **Analytics**: Add RevenueCat event tracking for key conversion events (trial start, conversion, cancellation).

9. **Polish**: Dark mode support, Dynamic Type, VoiceOver labels, empty states, loading states, error states.

10. **Compilation check**: The Builder runs `xcodebuild` to verify the project compiles. If it doesn't compile, the Builder fixes the errors before outputting.

> **Gate 0: Compilation** — After the Builder completes, the Router runs an external `xcodebuild build` check before spawning the Reviewer. If compilation fails, the project is sent straight back to the Builder for revision without wasting Reviewer tokens. This is a hard gate: no Reviewer session is created until the project compiles cleanly.

### Template Usage

The Builder doesn't start from scratch. It uses these templates:

```
templates/swiftui/
├── AppTemplate/          # Base project with build settings, Info.plist, Asset catalog
├── Onboarding.swift      # Configurable PageTabView-based onboarding
├── Paywall.swift          # StoreKit 2 SubscriptionStoreView + entitlement check
├── Settings.swift         # Version, privacy policy, support email, restore purchases
└── GeminiWrapper.swift    # Gemini Flash API client with streaming response
```

These templates handle the boilerplate. The Builder focuses on the unique features that make each app valuable.

---

## Phase 5: Lint

**Agent:** Linter (Haiku)
**Duration:** 2-5 minutes
**Input:** `src/` (complete Xcode project)
**Output:** `lint_report.json`
**Gate:** Hard fail if >5 issues or any critical flag; soft fail proceeds with lint report attached

### What Happens

After the Builder completes and Gate 0 (compilation) passes, the Router spawns a lightweight Haiku-based lint pass before engaging the Reviewer. This catches obvious issues cheaply, before burning expensive Codex tokens on review.

The Linter scans for:

1. **Force unwraps** (`!`): Any use of `!` outside of IBOutlet patterns is flagged.
2. **Missing permissions**: Cross-references Info.plist `NS*UsageDescription` keys against actual framework imports (HealthKit, Camera, Location, etc.). Flags permissions that are used but undeclared, or declared but unused.
3. **Dead code**: Detects unused imports, unreachable functions, and commented-out code blocks.
4. **Placeholder text**: Scans all `.swift` files and asset catalogs for "Lorem ipsum", "TODO", "FIXME", "placeholder", "test", and similar strings in user-facing contexts.

### Failure Modes

- **Hard fail (>5 issues OR any critical flag)**: The project skips Review entirely and goes straight back to the Builder with the lint report. No Reviewer tokens are spent. Critical flags include: force unwraps in data persistence code, missing privacy permissions for HealthKit/Location/Camera, or placeholder text in onboarding screens.
- **Soft fail (1-5 non-critical issues)**: The project proceeds to Review, but the lint report is attached to the Reviewer's input. The Reviewer can factor lint issues into its scoring.
- **Clean pass (0 issues)**: Proceed to Review without the lint report.

### Why Haiku

Haiku is fast and cheap. A lint pass costs ~$0.01-0.03 and takes 2-5 minutes. Catching a force unwrap before Codex runs saves $0.50-1.00 in Reviewer tokens and 10-20 minutes of review time. Over hundreds of builds, this adds up significantly.

---

## Phase 6: Review

**Agent:** Reviewer (GPT-5.3-Codex)
**Duration:** 10-20 minutes
**Input:** `src/` (complete Xcode project), `onepager.json` (for feature cross-reference), quality rubric
**Output:** `quality.json`
**Gate:** Score >= 8.0/10. Three failures = flagged for human review.

### What Happens

The Reviewer reads every file in the project and runs 8 independent quality gates. See [Quality Gates](06_QUALITY_GATES.md) for the full specification.

**Key principle:** The Reviewer uses a different model (GPT-5.3-Codex) than the Builder (Opus 4.6). This is intentional. If the same model builds and reviews, it's likely to miss its own patterns of errors. Cross-model review catches more issues.

### Incremental Review

On attempt 2 and beyond, the Reviewer operates in **incremental mode**. Instead of re-checking all 8 gates from scratch, it only re-evaluates the gates that previously failed or had issues flagged. Gates that scored 8 or above with no issues are carried forward from the previous review.

This cuts revision review time by 50-60% and reduces Reviewer token costs proportionally. The Router signals incremental mode by passing `mode: "incremental"` and the previous `quality.json` to the Reviewer. The Reviewer merges carried-forward gate scores with freshly evaluated gates to produce the updated report.

### Failure Handling

- **Score >= 8.0**: Pass. Proceed to monetization.
- **Score < 8.0, attempt < 3**: Fail. Send quality report back to Builder for revision. The Builder receives the specific issues and fixes them without breaking existing functionality.
- **Score < 8.0, attempt >= 3**: Flagged. The project is marked for human review. You receive a notification with the quality report and the three sets of feedback. You decide whether to intervene manually or kill the project.

---

## Phase 7: Monetize

**Agent:** Shipper (Sonnet 4.6, monetize mode)
**Duration:** 10-15 minutes
**Input:** `onepager.json`, `src/`, App Store Connect credentials
**Output:** Configured products in App Store Connect, updated `state.json`

### What Happens

1. **Create app record** in App Store Connect (via API)
2. **Create subscription group** for the app
3. **Configure pricing**: Set the subscription price based on the onepager's monetization strategy
4. **Configure free trial**: 7-day introductory offer (default)
5. **Set up RevenueCat**: Register the app in RevenueCat, configure webhook for server-side notifications
6. **Verify StoreKit configuration**: Ensure product IDs in the code match the App Store Connect configuration

> **Monetize and Package run in parallel.** The Router spawns both Shipper modes simultaneously. Icon generation, screenshot scripts, and ASO copy (Package) don't depend on App Store Connect product configuration (Monetize). Both phases write to separate output paths and the Router waits for both to complete before transitioning to Ship. This shaves 20-40 minutes off the pipeline.

---

## Phase 8: Package

**Agent:** Shipper (Sonnet 4.6, package mode)
**Duration:** 20-40 minutes
**Input:** `src/`, `onepager.json`, `quality.json`, Fastlane templates, Nano Banana Pro API
**Output:** `assets/` (icon, screenshots, promo video), `listing.json`

### What Happens

1. **Icon generation**: Call Nano Banana Pro API with a prompt describing the app's concept, color scheme, and category. Generate a 1024x1024 icon. Ensure it's distinct from all other AppFactory app icons.

2. **Screenshot capture**:
   - Write XCUITest scripts that navigate to each key screen
   - Run `fastlane snapshot` on simulators: iPhone 16 Pro Max, iPhone 16, iPad Pro 13" (M4)
   - Capture 5-8 screenshots per device showing the app's key features
   - Frame with `fastlane frameit` — add device bezels and marketing headlines

3. **Promo video**: Use Remotion to generate a 30-second app preview video showing the core user flow. Add background music, text overlays, and transitions.

4. **App Store listing**: Generate:
   - **Title** (30 char max): App name + primary keyword
   - **Subtitle** (30 char max): Compelling one-liner
   - **Description** (4000 char max): Problem → Solution → Features → Social proof framework
   - **Keywords** (100 char max): Comma-separated, from research data
   - **What's New**: First version release notes
   - **Privacy URL**: Link to a generated privacy policy
   - **Support URL**: Link to support email/form

5. **Privacy policy generation**: Generate a privacy policy based on the app's actual data collection (from the Privacy manifest). Host on a static page.

---

## Phase 9: Ship

**Agent:** Shipper (Sonnet 4.6, submit mode)
**Duration:** 10-20 minutes (upload time)
**Input:** `src/`, `assets/`, `listing.json`, Fastlane config
**Output:** Submission to App Store Connect
**Gate:** Human presses "Submit for Review" (or auto-submit if configured)

### What Happens

1. **Build IPA**: `fastlane build_app` with the correct signing identity and provisioning profile (managed by `fastlane match`)
2. **Upload**: `fastlane deliver` uploads the IPA, metadata, screenshots, and promo video to App Store Connect
3. **Pre-check**: `fastlane precheck` validates metadata before submission to catch common rejection reasons
4. **Queue**: If fewer than 5 apps are currently in Apple review, add to the submission queue. Otherwise, hold.
5. **Notify human**: Alert via the Control Panel dashboard that an app is ready for final approval
6. **On approval**: Human presses submit in App Store Connect (or the Shipper calls `submit_for_review: true` if auto-submit is enabled)

### The Submit Button

The only mandatory manual step in the entire pipeline is pressing "Submit for Review" in App Store Connect. This is intentional:
- Apple limits 5 concurrent reviews per developer account
- Submissions should be batched strategically
- You maintain final control over what goes to Apple

In the Control Panel, you'll see a "Ready to Ship" queue. Tap an app to see its full one-pager, quality score, and all assets. If everything looks good, press "Approve for Submission."

---

## Phase 10: Market

**Agent:** Marketer (Opus 4.6 + Larry skill)
**Duration:** Ongoing (runs on its own schedule)
**Input:** `onepager.json`, audience analytics, content templates, post performance data
**Output:** Social media posts, promo videos, analytics updates

### What Happens

1. **Launch content**: When an app goes live, generate launch content:
   - TikTok slideshow (problem → solution → app demo)
   - Instagram carousel (feature highlights)
   - YouTube Short (15-second app walkthrough)

2. **Category content** (ongoing): The Marketer maintains content schedules for each category account. Content is valuable on its own — sleep tips for the Health & Fitness account, focus techniques for the Productivity account. App mentions are woven in naturally.

3. **Performance tracking**: After each post, track engagement metrics. Identify patterns: what hooks work, what posting times get the most views, what content formats drive the most app installs.

4. **Iteration**: The Larry skill handles the self-learning loop. It analyzes performance data, adjusts content strategy, and evolves its approach over time. This runs autonomously.

5. **Cross-promotion**: When a user of one AppFactory app would benefit from another, the Marketer creates cross-promotion content. "If you loved [App A], you'll love [App B] for [related use case]."

---

## Phase Context

Before spawning any agent, the Router writes a `phase_context` object to the project's `state.json`. This gives every agent immediate awareness of why it was invoked and what happened before it, without needing to parse previous conversation history.

The `phase_context` object contains:

| Field | Description |
|-------|-------------|
| `reason` | Why this agent is being invoked (e.g., "initial_build", "revision_after_review_failure", "lint_hard_fail") |
| `priority` | Priority level: `normal`, `high` (revision), or `critical` (3rd attempt) |
| `flags` | Relevant flags for this invocation (e.g., `["incremental_review", "lint_soft_fail"]`) |
| `previous_session_id` | Session ID of the last agent that touched this project, for traceability |

Example:
```json
{
  "phase_context": {
    "reason": "revision_after_review_failure",
    "priority": "high",
    "flags": ["incremental_review", "lint_clean"],
    "previous_session_id": "review-abc123-20260302"
  }
}
```

This ensures agents never operate blind. A Builder in revision mode immediately knows it was triggered by a review failure, what the priority level is, and which Reviewer session produced the feedback.

---

## Pipeline Timing

For a single app, assuming no review failures:

| Phase | Duration | Cumulative |
|-------|----------|------------|
| Research | 15-30 min | 30 min |
| Validate | 10-20 min | 50 min |
| Spec | 20-40 min | 1.5 hrs |
| Build | 30-90 min | 3 hrs |
| Lint | 2-5 min | 3.1 hrs |
| Review | 10-20 min | 3.5 hrs |
| Monetize + Package | 20-40 min (parallel) | 4 hrs |
| Ship | 10-20 min | 4.5 hrs |
| Apple Review | 24-72 hrs | 3-4 days |

> **Note:** Monetize and Package run in parallel, so their durations are not additive. The cumulative time reflects the longer of the two (Package). The Lint phase adds minimal overhead (2-5 minutes) but saves significant time by catching issues before the expensive Review phase.

**With 5 concurrent projects**, the factory is always working on something. While one app is in Apple review, others are building, reviewing, or researching.

---

## Error Recovery

### Build fails to compile
The Builder retries with error messages. If it can't fix compilation errors in the same session, it writes the errors to `state.json` and the Router flags it for human review.

### Review fails 3 times
Project is flagged for human review. The one-pager, source code, and all three quality reports are presented in the Control Panel. You decide: manual fix, kill, or restart from spec.

### Apple rejects the app
The Shipper reads the rejection reason from App Store Connect. If it's a metadata issue (common), it fixes and resubmits. If it's a code issue, it sends the rejection back to the Builder via the Router.

### API rate limits
All agents implement exponential backoff. The Router tracks rate limit state and avoids spawning sessions that would hit limits.

### Human suggestion override
At any point, you can manually move a project to any phase, kill it, or restart it from the Control Panel. Your manual actions always override the automated pipeline.
