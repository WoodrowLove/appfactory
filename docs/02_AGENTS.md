# Agents

## Agent Philosophy

Every agent in the factory is a specialist. It has one job, it does that job well, and it writes its results to files. No agent holds the full system context. No agent relies on conversation history from a previous session.

**Rules for all agents:**
1. Read your inputs from files. Write your outputs to files.
2. Never assume context from a previous session.
3. If you need information that isn't in your input files, say so in your output and exit.
4. Update `state.json` before exiting. Always.
5. Log your reasoning. Every decision should be traceable.

## Agent Roster

### 1. Router (Orchestrator)

| Property | Value |
|----------|-------|
| **Model** | Sonnet 4.6 |
| **Trigger** | Adaptive polling (1 min active / 15 min idle / 5 min default) |
| **Context budget** | < 5% of context window |
| **Input** | `projects/*/state.json`, `suggestions/*.json`, `config/openclaw.yaml` |
| **Output** | Spawned agent sessions, `factory.db` log entries |

**Responsibilities:**
- Read all project state files
- Count active projects (enforce max 5)
- For each active project, determine the current phase
- Spawn the correct agent with the correct prompt and input files
- Check the submission queue (max 5 in Apple review)
- Pull from the suggestion queue or research queue when slots open
- Log every routing decision with timestamp and reasoning

**Decision logic (pseudocode):**
```
every poll cycle (adaptive: 1 min active / 15 min idle / 5 min default):
  active_projects = read all projects/*/state.json where status != "killed" and status != "live"

  if len(active_projects) < 5:
    if suggestions_queue is not empty:
      start_new_project(next_suggestion)
    elif research_queue has validated ideas:
      start_new_project(next_validated_idea)
    else:
      trigger_scout_for_new_research()

  for project in active_projects:
    if project.status == "queued":
      spawn(scout, project)
    elif project.status == "researching" and project.research_complete:
      transition(project, "validating")
      spawn(scout, project, mode="validate")
    elif project.status == "validating" and project.validation_complete:
      if project.validation_score >= 21:
        transition(project, "speccing")
        spawn(architect, project)
      else:
        transition(project, "killed", reason="validation_score_too_low")
    elif project.status == "speccing" and project.spec_complete:
      transition(project, "building")
      spawn(builder, project)
    elif project.status == "building" and project.build_complete:
      transition(project, "linting")
      spawn(linter, project)
    elif project.status == "linting" and project.lint_complete:
      if project.lint_result == "hard_fail":
        transition(project, "revising")
        spawn(builder, project, mode="revise", feedback=project.lint_report)
      else:  # pass or soft_fail
        transition(project, "reviewing")
        spawn(reviewer, project)
    elif project.status == "reviewing" and project.review_complete:
      if project.quality_score >= 8:
        transition(project, "monetizing_and_packaging")
        spawn(shipper, project, mode="monetize")
        spawn(shipper, project, mode="package")
      elif project.review_attempts < 3:
        transition(project, "revising")
        spawn(builder, project, mode="revise", feedback=project.quality_report)
      else:
        transition(project, "flagged", reason="3_review_failures")
        notify_human(project)
    elif project.status == "monetizing_and_packaging" and project.monetization_complete and project.packaging_complete:
      transition(project, "ready_to_ship")
      notify_human(project, action="approve_submission")
    elif project.status == "ready_to_ship" and project.human_approved:
      if apps_in_review < 5:
        transition(project, "in_review")
        spawn(shipper, project, mode="submit")
    elif project.status == "approved":
      transition(project, "live")
      spawn(marketer, project)
```

**What the Router does NOT do:**
- Read source code
- Make creative decisions
- Judge quality
- Generate content
- Hold multi-turn conversations

---

### 2. Scout (Research & Validation)

| Property | Value |
|----------|-------|
| **Model** | Sonnet 4.6 |
| **Trigger** | Spawned by Router |
| **Context budget** | Medium (research data can be large, but it summarizes) |
| **Input** | Keywords, subreddit list, category focus, existing projects list |
| **Output** | `research.json`, updated `state.json` |

**Responsibilities (Research mode):**
- Crawl specified subreddits for pain points, broken workflows, "I wish there was an app for..."
- Scan X/Twitter for complaints in target categories
- Check Product Hunt for trending categories
- Analyze App Store reviews for high-demand, low-competition niches
- Cross-reference: Is this technically feasible for a single SwiftUI app?
- Output a ranked list of ideas with scores

**Responsibilities (Validation mode):**
- For a specific idea, deep-dive into competition
- Search App Store for direct competitors (name, rating, review count, last updated)
- Check Apple Search Ads for keyword difficulty and search volume
- Compare against all existing AppFactory projects for Guideline 4.3 risk
- Score on three axes: demand (1-10), competition (1-10), feasibility (1-10)
- Gate: combined score must be >= 21/30 to proceed

**Research output schema** (see `schemas/research.schema.json`):
```json
{
  "idea_slug": "sleep-routine-tracker",
  "category": "health-fitness",
  "pain_points": [
    {
      "source": "reddit",
      "subreddit": "r/sleep",
      "post_url": "...",
      "summary": "Users frustrated that sleep apps track sleep but don't help build routines",
      "upvotes": 342,
      "comment_sentiment": "negative"
    }
  ],
  "competitors": [
    {
      "name": "Sleep Cycle",
      "rating": 4.6,
      "review_count": 189000,
      "last_updated": "2026-02-15",
      "weakness": "No routine builder, only tracking"
    }
  ],
  "keywords": {
    "primary": "sleep routine",
    "search_volume_estimate": "medium",
    "difficulty_estimate": "low"
  },
  "scores": {
    "demand": 8,
    "competition": 7,
    "feasibility": 9,
    "total": 24
  },
  "differentiation_from_existing": "No overlap with any current AppFactory project. Nearest is focus-timer (productivity), which serves a different use case.",
  "technical_notes": "SwiftUI + HealthKit integration. No server needed. Local notifications for routine reminders.",
  "monetization_hypothesis": "Freemium: basic routines free, AI-powered personalized routines behind paywall. $4.99/month."
}
```

**Tools the Scout uses:**
- Reddit API (read-only, OAuth2)
- X API v2 (read-only, bearer token)
- App Store Search API (unofficial, via `searchads.apple.com` or scraping)
- Apple Search Ads API (keyword insights)
- Web search (for Product Hunt, review sites)

---

### 3. Architect (Specification)

| Property | Value |
|----------|-------|
| **Model** | Opus 4.6 |
| **Trigger** | Spawned by Router |
| **Context budget** | High (needs to reason deeply about product decisions) |
| **Input** | `research.json`, spec templates, existing app catalog |
| **Output** | `spec.md`, `onepager.json`, updated `state.json` |

**Responsibilities:**
- Transform research data into a complete product specification
- Define the target user persona
- Map pain points to features
- Design screen-by-screen layout (wireframe descriptions, not images)
- Choose the monetization strategy
- Define the differentiation angle (what makes this app unique)
- Write 3-5 onboarding screens (copy and flow)
- Specify any third-party integrations (HealthKit, CloudKit, Gemini Flash, etc.)
- Evaluate ICP integration needs (audit trail, trust scoring)
- Produce both human-readable spec.md and machine-readable onepager.json

**The one-pager is the contract.** Once approved, the Builder builds exactly what the spec says. No more, no less. If the spec is wrong, the app will be wrong. This is why Opus handles this phase — it requires the deepest reasoning.

**One-pager structure** (see `schemas/onepager.schema.json`):
```json
{
  "app_name": "RoutineRest",
  "tagline": "Build the sleep routine your body needs",
  "category": "Health & Fitness",
  "target_user": {
    "persona": "Health-conscious adults 25-45 who want better sleep but don't know where to start",
    "pain_points": ["..."],
    "current_solutions": ["..."],
    "why_they_fail": ["..."]
  },
  "features": {
    "core": ["Routine builder with templates", "Bedtime/wake reminders", "HealthKit sleep data import"],
    "premium": ["AI-powered routine personalization via Gemini Flash", "Sleep quality scoring", "Weekly insights"],
    "deferred": ["Apple Watch app", "Social sharing"]
  },
  "screens": [
    {"name": "Onboarding 1", "purpose": "Welcome + value prop", "copy": "..."},
    {"name": "Onboarding 2", "purpose": "Sleep goal selection", "copy": "..."},
    {"name": "Onboarding 3", "purpose": "Current routine assessment", "copy": "..."},
    {"name": "Home", "purpose": "Tonight's routine + progress ring", "elements": ["..."]},
    {"name": "Routine Editor", "purpose": "Build/modify routine steps", "elements": ["..."]},
    {"name": "Insights", "purpose": "Weekly sleep quality trends", "elements": ["..."]},
    {"name": "Settings", "purpose": "Notifications, theme, account", "elements": ["..."]},
    {"name": "Paywall", "purpose": "Premium upgrade", "elements": ["..."]}
  ],
  "monetization": {
    "model": "freemium_subscription",
    "free_tier": "3 built-in routines, basic tracking",
    "premium_tier": "Unlimited custom routines, AI personalization, insights",
    "price": "$4.99/month",
    "trial": "7 days free",
    "rationale": "Free trials convert 6x better than hard paywalls. Core value is locked behind personalization."
  },
  "differentiation": {
    "vs_competitors": "Sleep Cycle tracks but doesn't build. We build routines. Completely different value prop.",
    "vs_our_catalog": "No overlap with existing AppFactory apps.",
    "unique_angle": "First app focused on sleep routine construction rather than sleep tracking."
  },
  "technical": {
    "frameworks": ["SwiftUI", "HealthKit", "UserNotifications", "StoreKit 2"],
    "ai_integration": "Gemini Flash for routine personalization (premium feature)",
    "data_storage": "Local-first (SwiftData). No server required.",
    "min_ios": "17.0",
    "estimated_complexity": "medium"
  },
  "app_store": {
    "primary_category": "Health & Fitness",
    "secondary_category": "Lifestyle",
    "keywords": ["sleep routine", "bedtime routine", "sleep habit", "sleep schedule", "sleep better"],
    "age_rating": "4+"
  }
}
```

---

### 4. Builder

| Property | Value |
|----------|-------|
| **Model** | Opus 4.6 |
| **Trigger** | Spawned by Router |
| **Context budget** | High (full app generation requires deep context) |
| **Input** | `spec.md`, `onepager.json`, SwiftUI templates, coding standards |
| **Output** | `src/` (complete Xcode project), updated `state.json` |

**Responsibilities:**
- Generate a complete, compilable Xcode project
- Implement every screen defined in the spec
- Integrate StoreKit 2 using the paywall template
- Implement 3-5 onboarding screens per spec
- Add Gemini Flash wrapper if AI features are specified
- Handle all permissions (Info.plist entries + runtime prompts)
- Implement analytics hooks (RevenueCat events)
- Follow SwiftUI best practices (MVVM, @Observable, proper navigation)
- Handle accessibility (VoiceOver labels, Dynamic Type)
- Support dark mode
- Use Swift's `Decimal` type for any financial calculations (no `Double` or `Float` where precision matters)

**Revision mode:**
When called with quality feedback from the Reviewer, the Builder receives:
- The original spec
- The current source code
- The quality report with specific issues
- Instructions to fix only the flagged issues without breaking existing functionality

**Templates provided to the Builder:**
- `templates/swiftui/AppTemplate/` — Base Xcode project structure
- `templates/swiftui/Onboarding.swift` — Configurable onboarding flow
- `templates/swiftui/Paywall.swift` — StoreKit 2 subscription paywall
- `templates/swiftui/Settings.swift` — Standard settings with version, support, privacy
- `templates/swiftui/GeminiWrapper.swift` — Gemini Flash API integration

**Coding standards the Builder follows:**
1. No force unwraps (`!`) — use `guard let` or `if let`
2. All strings user-facing must be localized (`String(localized:)`)
3. All network calls must handle errors gracefully with user-facing messages
4. Navigation via `NavigationStack` with typed destinations
5. State management via `@Observable` macro (iOS 17+)
6. StoreKit 2 for all purchases — no StoreKit 1
7. Minimum iOS 17.0 deployment target
8. No third-party dependencies unless specified in the spec (keep the binary lean)

---

### 5. Linter

| Property | Value |
|----------|-------|
| **Model** | Claude Haiku 4 |
| **Trigger** | Spawned by Router (after Gate 0 compilation passes) |
| **Context budget** | Minimal (fast, cheap lint pass) |
| **Input** | `src/`, `onepager.json` |
| **Output** | `lint_report.json`, updated `state.json` |

**Responsibilities:**
Run a fast, automated lint pass before the full Reviewer engages. Checks 7 categories:

1. **Force unwraps** — Any use of `!` on optionals (use `guard let` / `if let` instead)
2. **Missing permissions** — Cross-reference Info.plist permission keys against actual API usage in code
3. **Dead code** — Unused imports, unreachable functions, commented-out blocks
4. **Force casts** — Any use of `as!` (use `as?` with proper handling instead)
5. **Missing restore purchases** — Verify a "Restore Purchases" button exists and is wired up
6. **Missing privacy manifest** — Verify `PrivacyInfo.xcprivacy` exists and covers all declared API categories
7. **Placeholder content** — Detect Lorem ipsum, "TODO", "FIXME", placeholder images, or dummy URLs

**Outcome logic:**
- **hard_fail**: Any force unwrap, missing restore purchases, or missing privacy manifest found. Project returns to REVISING.
- **soft_fail**: Minor issues (dead code, placeholder comments) found. Issues are forwarded to the Reviewer as advisory notes. Project proceeds to REVIEWING.
- **pass**: No issues found. Project proceeds to REVIEWING.

**Why Haiku?** This is a fast, pattern-matching task. Haiku is cheap and fast enough to run as a gate without meaningful cost or latency impact. It catches the obvious issues before the expensive Codex review.

---

### 6. Reviewer

| Property | Value |
|----------|-------|
| **Model** | GPT-5.3-Codex |
| **Trigger** | Spawned by Router |
| **Context budget** | High (reads every file in the project) |
| **Input** | `src/` (complete Xcode project), quality rubric |
| **Output** | `quality.json`, updated `state.json` |

**Responsibilities:**
Run 8 independent quality gates, each scored 0-10. The final score is the average, rounded down. An app needs >= 8.0 to proceed.

**The 8 Quality Gates:**

#### Gate 1: Crash Safety (0-10)
- No force unwraps
- All optionals handled
- No array index out of bounds risks
- No retain cycles (check for `[weak self]` in closures)
- Proper error handling in async/await blocks
- No implicit main-thread violations

#### Gate 2: Permission Handling (0-10)
- Every Info.plist permission has a clear, specific usage description
- Every permission is requested at the point of use (not at launch)
- App functions gracefully if permission is denied
- No unnecessary permissions requested
- Privacy manifest (PrivacyInfo.xcprivacy) is complete and accurate

#### Gate 3: Feature Completeness (0-10)
- Cross-reference every feature in `onepager.json` against the implementation
- Every screen described in the spec exists and is reachable
- All navigation flows work (no dead ends)
- Onboarding screens match the spec
- Paywall is implemented and functional

#### Gate 4: UI/UX Quality (0-10)
- Consistent spacing, typography, and color usage
- Dark mode support across all screens
- Dynamic Type support
- VoiceOver accessibility labels on interactive elements
- Loading states for async operations
- Empty states for lists with no data
- Pull-to-refresh where appropriate
- Proper keyboard handling (dismiss, avoidance)

#### Gate 5: App Store Compliance (0-10)
- No private API usage
- No references to competing platforms (Android, Google Play)
- All data collection disclosed in Privacy manifest
- Subscription terms clearly displayed before purchase
- "Restore Purchases" button present and functional
- Age rating appropriate for content
- No placeholder text or Lorem ipsum
- Differentiation analysis: does this app look/feel distinct from our other apps?

#### Gate 6: Code Quality (0-10)
- No dead code or unused imports
- Consistent naming conventions (camelCase for variables, PascalCase for types)
- Proper separation of concerns (View / ViewModel / Model)
- No business logic in views
- Proper use of Swift concurrency (async/await, not completion handlers)
- File organization matches feature modules
- Comments on non-obvious logic

#### Gate 7: Test Coverage (0-10)
- >= 60% line coverage on model and ViewModel layers
- Unit tests exist for all public model methods
- ViewModel state transitions are tested
- Edge cases (empty data, error states) are covered
- Tests compile and pass without warnings

#### Gate 8: Monetization UX (0-10)
- Hybrid paywall implemented (monthly + annual options)
- Annual savings percentage clearly displayed
- Subscription management link present (links to iOS subscription settings)
- Free trial terms displayed before purchase
- Price formatting is locale-aware
- Restore purchases is discoverable (not buried)
- Paywall copy matches the value proposition from the spec

**Quality report output** (see `schemas/quality.schema.json`):
```json
{
  "project_slug": "routine-rest",
  "review_attempt": 1,
  "timestamp": "2026-03-02T16:30:00Z",
  "gates": {
    "crash_safety": {
      "score": 9,
      "issues": [
        {"severity": "minor", "file": "InsightsView.swift", "line": 42, "description": "Optional binding should use guard let for early return"}
      ]
    },
    "permission_handling": { "score": 10, "issues": [] },
    "feature_completeness": {
      "score": 8,
      "issues": [
        {"severity": "medium", "file": "RoutineEditorView.swift", "description": "Missing reorder functionality for routine steps as specified in onepager"}
      ]
    },
    "ui_ux_quality": { "score": 9, "issues": [] },
    "app_store_compliance": { "score": 10, "issues": [] },
    "code_quality": { "score": 8, "issues": [] }
  },
  "overall_score": 9,
  "pass": true,
  "summary": "High quality build. Minor issues in crash safety and feature completeness. Recommend fixing the routine reorder before shipping.",
  "blocking_issues": [],
  "recommendations": [
    "Add guard let in InsightsView.swift:42",
    "Implement drag-to-reorder in RoutineEditorView"
  ]
}
```

---

### 7. Shipper

| Property | Value |
|----------|-------|
| **Model** | Sonnet 4.6 |
| **Trigger** | Spawned by Router |
| **Context budget** | Medium (deterministic pipeline steps) |
| **Input** | `src/`, `onepager.json`, `quality.json`, Fastlane templates |
| **Output** | `assets/`, `listing.json`, IPA binary, updated `state.json` |

**Operates in three modes:**

#### Monetize Mode
- Configure StoreKit 2 products in App Store Connect (via API)
- Create subscription group
- Set pricing tiers
- Configure 7-day free trial offer
- Verify RevenueCat webhook integration

#### Package Mode
- Generate app icon via Nano Banana Pro API (1024x1024, matching app's brand)
- Set up Fastlane configuration from templates
- Run Fastlane snapshot (XCUITest-based screenshot capture across devices)
- Frame screenshots with device bezels and marketing text (Fastlane frameit)
- Generate promo video via Remotion (30-second app walkthrough)
- Write App Store listing: title, subtitle, description, keywords, what's new
- Compile all assets into `assets/` directory

#### Submit Mode
- Build IPA via Fastlane (`fastlane build_app`)
- Upload to App Store Connect (`fastlane deliver`)
- Set metadata, screenshots, and promo video
- Submit for review
- Update state to `in_review`

**The Shipper generates screenshots by:**
1. Writing XCUITest scripts that navigate to each key screen
2. Running `fastlane snapshot` against multiple simulators (iPhone 16 Pro Max, iPhone 16, iPad Pro 13" (M4))
3. Each simulator captures screenshots in the app's primary language
4. `fastlane frameit` adds device frames and marketing headlines
5. Screenshots are organized per device size as required by App Store Connect

---

### 8. Marketer

| Property | Value |
|----------|-------|
| **Model** | Opus 4.6 (creative) + gpt-image-1.5 (visuals) |
| **Trigger** | Spawned by Router on app launch; also runs on its own content schedule |
| **Context budget** | Medium (content generation + analytics) |
| **Input** | `onepager.json`, audience analytics, content templates, performance data |
| **Output** | Social media posts, promo videos, updated analytics |

**Responsibilities:**

#### Content Generation
- Generate native content for category-specific social accounts
- Health & Fitness account: sleep tips, routine advice, wellness content
- Productivity account: focus techniques, workflow hacks, tool comparisons
- Each piece of content provides genuine value — it's not just ads
- When a relevant app ships, weave a natural mention into the content

#### Channel Management
- Post via Postiz API (authorized, API-compliant posting)
- Maintain posting schedules per platform per category
- Track per-post engagement (views, likes, comments, saves, shares)
- Identify high-performing content patterns and double down
- Kill underperforming content angles

#### Promo Videos
- Generate 15-30 second app walkthrough videos via Remotion
- Create TikTok-style hook videos (problem → solution → app demo)
- Output as MP4 for upload to social platforms

#### Self-Learning Loop
- After each content cycle, analyze what performed well
- Adjust tone, hooks, posting times based on data
- Write performance insights to `marketing/<category>/analytics.json`
- The Larry skill (OpenClaw) handles the autonomous iteration loop

**Content rules:**
1. Every post must provide value even without the app mention
2. No fake urgency ("LAST CHANCE" or "LIMITED TIME")
3. No misleading claims about app capabilities
4. All content must comply with platform ToS
5. One account per category (not multiple accounts per category)
6. Use official platform APIs only (no bots or automation that violates ToS)

---

## Model Selection Rationale

| Model | Role | Why |
|-------|------|-----|
| **Opus 4.6** | Builder, Architect, Marketer | Deepest reasoning. Building a full app from a spec, designing product specifications, and creating compelling creative content all require the highest-quality model available. The cost per session is justified by the output quality. |
| **Sonnet 4.6** | Router, Scout, Shipper | Fast, cheap, excellent at structured tasks. The Router needs speed (adaptive polling cycles). The Scout processes lots of data but makes simple decisions. The Shipper runs deterministic pipelines. Sonnet handles all of these well at a fraction of Opus cost. |
| **Haiku 4** | Linter | Cheapest and fastest Anthropic model. The lint pass is pattern-matching (force unwraps, missing manifests, placeholder text) — it doesn't need deep reasoning. Running Haiku as a gate before Codex catches obvious issues at near-zero cost. |
| **GPT-5.3-Codex** | Reviewer | Independent verification requires a different model family. If the same model builds and reviews, it's likely to miss its own blind spots. Codex is specifically optimized for code analysis. Using a different vendor also prevents correlated failures. |
| **Gemini Flash** | In-app AI (user-facing) | Used inside the apps themselves as the AI backend. Fast, cheap, good enough for user-facing features like personalized recommendations. Not used in the factory pipeline. |
| **Nano Banana Pro** | Icon generation | Google's latest image model with excellent design aesthetics. Specifically good at generating clean, icon-style imagery at high resolution. |
| **gpt-image-1.5** | Marketing visuals | OpenAI's image model for social media content. Used by the Marketer for post imagery. |

## Agent Communication

Agents do NOT communicate directly. All communication happens through files:

```
Scout writes research.json → Router reads it → Router spawns Architect with research.json
Architect writes spec.md → Router reads it → Router spawns Builder with spec.md
Builder writes src/ → Router reads it → Router spawns Linter with src/
Linter writes lint_report.json → Router reads it → Router spawns Reviewer (or back to Builder on hard_fail)
Reviewer writes quality.json → Router reads it → Router spawns Builder or Shipper
```

This is intentional. It means:
- Any agent can be replaced without affecting others
- Any agent can be re-run on the same inputs for debugging
- The full history of every project is captured in files
- Context bloat is impossible because each agent starts fresh
