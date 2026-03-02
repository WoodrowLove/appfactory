# Terminal Agent Prompt

Use this prompt to continue setup in a fresh Claude Code terminal session. Copy-paste the entire prompt below into a new Claude Code session running on macOS.

---

## Prompt

```
You are helping me set up AppFactory, an autonomous iOS app factory. The repository is already created at ~/appfactory with full documentation, agent prompts, schemas, and configuration files.

Read the following files first to understand the system:
- README.md (overview)
- docs/01_ARCHITECTURE.md (system design)
- docs/15_DEPLOYMENT.md (setup instructions)
- .claude/CLAUDE.md (project instructions)
- config/openclaw.yaml (orchestrator config with adaptive polling, event-driven webhook, budget enforcement)
- config/budget.json (cost budget and per-model pricing)
- config/models.yaml (model assignments — single source of truth for model IDs)

Then complete the following setup tasks in order:

### Phase 1: SvelteKit Control Panel

1. Initialize a SvelteKit project in the repo root:
   - Use SvelteKit 2 with Svelte 5
   - Add TailwindCSS v4
   - Add better-sqlite3 for database access
   - Add chokidar for file watching
   - Add ws (WebSocket library) for live updates

2. Build the Control Panel with these pages:
   - Dashboard (home): metric cards (active projects, shipped apps, MRR, queue length), live activity feed, needs-attention section (items awaiting human approval)
   - Projects: list of projects with progress bars and quality scores, expandable one-pager view, lint report badge (pass/soft_fail/hard_fail), phase context display
   - Revenue: MRR charts, per-app breakdown, conversion funnel, contextual paywall vs onboarding paywall conversion comparison, RevenueCat experiment results
   - Marketing: content calendar, performance metrics, engagement stats, win-back campaign status
   - Suggestions: form to submit app ideas (name, category, pain point, priority), list of existing suggestions with status
   - Settings: configuration management, budget dashboard (current spend vs limit, cost per app), model assignments viewer
   - Activity Log: searchable log of all agent sessions with phase context, duration, cost, outcome

3. Implement the backend:
   - API routes to read projects/*/state.json files (including new fields: phase_context, fixing, human_approved_submission, auto_approved)
   - API routes to read/write factory.db (SQLite)
   - API route to write suggestion files to suggestions/
   - API route to approve/reject specs and submissions (PATCH /api/projects/:slug/approve)
   - WebSocket endpoint for live activity feed
   - File watcher on projects/ directory for real-time updates
   - Budget endpoint reading config/budget.json and calculating current month spend from agent_sessions table

4. Initialize factory.db with the schema from docs/11_STATE_MANAGEMENT.md
   - Include all tables: events, quality_scores, revenue, ideas, content_performance, submission_queue, agent_sessions
   - Include the new columns: gate_0_pass, gate_7_score, gate_8_score, lint_pass, incremental_review (from updated Quality Gates doc)

### Phase 2: Orchestrator Scripts

5. Create these Node.js scripts in a scripts/ directory:
   - scripts/router.js — Main orchestrator logic:
     * Reads state files, makes routing decisions, spawns agents
     * Implements adaptive polling (1 min active / 15 min idle / 5 min default) per config/openclaw.yaml
     * Validates all JSON outputs against schemas/ before phase transitions
     * Runs Gate 0 (compilation check via xcodebuild) after Builder completes
     * Spawns Linter (Haiku) after Gate 0 passes, before Reviewer
     * Supports incremental review mode (on attempt 2+, only re-check failed gates)
     * Runs Monetize and Package phases in parallel
     * Implements auto-approval for specs (4hr timeout, score >= 24/30)
     * Never auto-approves submissions — always requires human
     * Writes phase_context to state.json before every agent spawn
     * Budget pre-check before spawning any agent (blocks non-critical at 100%, alerts at 80%)
   - scripts/webhook-server.js — Event-driven transition webhook:
     * Express server on port 3002
     * POST /agent-complete endpoint receives completion events
     * Triggers immediate routing instead of waiting for next cron tick
     * Auth token verification per config/openclaw.yaml
   - scripts/session-complete.js — Callback handler when an agent session completes
   - scripts/marketing-scheduler.js — Checks marketing content schedule
   - scripts/revenue-sync.js — Pulls RevenueCat data including experiment results and win-back campaign metrics
   - scripts/review-status.js — Checks App Store Connect review status, monitors phased release rollout
   - scripts/backup.js — Daily backup of factory.db and state files
   - scripts/init-db.js — Database initialization from schema
   - scripts/verify-keys.js — API key verification for all required services
   - scripts/schema-validator.js — Shared module that validates JSON against schemas/*.schema.json using Ajv

### Phase 3: SwiftUI Templates

6. Create the SwiftUI template files in templates/swiftui/:
   - AppTemplate/ — Base Xcode project structure that compiles with iOS 17.0 deployment target
   - Onboarding.swift — Configurable TabView-based onboarding with @AppStorage for "hasSeenOnboarding"
   - Paywall.swift — StoreKit 2 SubscriptionStoreView paywall with .storeButton(.visible, for: .restorePurchases)
   - ContextualPaywall.swift — PremiumGateModifier for hybrid paywall strategy (feature-gated + onboarding)
   - Settings.swift — Standard settings with privacy, restore, version, and "More Apps" cross-promotion section
   - GeminiWrapper.swift — Gemini Flash API integration using actor pattern
   - CrossPromo.swift — "More from AppFactory" module reading from more_apps.json
   - PrivacyInfo.xcprivacy — Pre-configured privacy manifest (required by Apple)
   - Info.plist template — Including SKAdNetworkItems with Apple's network ID (cstr6suwn9.skadnetwork)
   - Localizable.xcstrings — String catalog configured for EN, ES, DE, FR, JA

### Phase 4: Fastlane Templates

7. Create Fastlane template files in templates/fastlane/:
   - Fastfile — Build, screenshot, and deploy lanes including:
     * test lane (runs xcodebuild test with code coverage)
     * screenshot lane (using iPhone 16 Pro Max, iPhone 16, iPad Pro 13" M4)
     * beta lane (TestFlight upload with changelog)
     * release lane (App Store submission with phased release enabled)
     * regional_pricing lane (configures pricing tiers per region from onepager.json)
   - Appfile — App identity template
   - Snapfile — Screenshot configuration for all three device types
   - Framefile — frameit configuration for screenshot framing

### Phase 5: Lint Script

8. Create the lint runner:
   - scripts/lint-runner.js — Executes the 7 lint checks from prompts/linter.md:
     * Force unwraps (scan .swift files for ! outside strings/comments)
     * Missing permissions (cross-reference Info.plist NS*UsageDescription vs imports)
     * Dead code (unused functions, unused imports)
     * Force casts (as! patterns)
     * Missing restore purchases (AppStore.sync() or .storeButton(.visible, for: .restorePurchases))
     * Missing privacy manifest (PrivacyInfo.xcprivacy exists and is non-empty)
     * Placeholder content (Lorem ipsum, TODO, FIXME, placeholder, test data in user-facing strings)
   - Outputs lint_report.json per the schema in prompts/linter.md

### Phase 6: Integration

9. Set up the OpenClaw cron configuration:
   - Read config/openclaw.yaml and config/cron.json
   - Create a startup script (scripts/start.sh) that:
     * Starts the webhook server (scripts/webhook-server.js) on port 3002
     * Starts the SvelteKit control panel
     * Registers cron jobs
   - Test that the router script runs without errors (in dry-run mode)

10. Verify the complete system:
    - Control Panel starts and shows the dashboard
    - Database is initialized with the correct schema (all tables, all columns)
    - File watcher detects changes in projects/
    - Template files are valid Swift that compiles
    - Schema validator correctly validates sample state.json against schemas/state.schema.json
    - Webhook server responds to POST /agent-complete
    - Budget calculations work correctly

### Important Notes:
- This project runs on macOS (required for Xcode). You are running in a terminal on that machine.
- Read the existing documentation before implementing — the docs are comprehensive and recently audited.
- Follow the schemas in schemas/ for all data structures. The Router MUST validate all agent outputs against these schemas.
- The agent prompts in prompts/ define each agent's behavior (8 agents: Router, Scout, Architect, Builder, Linter, Reviewer, Shipper, Marketer).
- Model assignments live in config/models.yaml — this is the single source of truth for model IDs.
- Budget limits live in config/budget.json — enforce the $500/month cap.
- Device references: iPhone 16 Pro Max, iPhone 16, iPad Pro 13" (M4). Never reference iPhone 15.
- Do NOT install OpenClaw or Claude Code if not already installed — just set up the project code.
- Ask me before making any decisions that aren't covered in the documentation.
- If you encounter an error, fix it. Don't skip steps.
- Commit your work as you go with clear commit messages.
```

---

## What This Prompt Does

This prompt instructs a terminal Claude Code agent to build all the executable code for the AppFactory system. The documentation (which is already complete in the repo) serves as the specification. The agent reads the docs and implements the system accordingly.

### Expected Timeline
- Phase 1 (Control Panel): 45-75 minutes
- Phase 2 (Orchestrator Scripts): 30-45 minutes
- Phase 3 (SwiftUI Templates): 15-25 minutes
- Phase 4 (Fastlane Templates): 10-15 minutes
- Phase 5 (Lint Script): 15-20 minutes
- Phase 6 (Integration): 10-15 minutes

### What You Need to Do After
1. Copy your API keys into `.env`
2. Set up Fastlane match for code signing: `fastlane match init && fastlane match appstore`
3. Install OpenClaw: `npm install -g openclaw@latest`
4. Register the cron jobs: `openclaw cron create --config config/openclaw.yaml`
5. Start everything: `./scripts/start.sh`

### What Requires a Mac
The following must be done on macOS:
- Xcode installation and configuration
- Fastlane match setup (code signing)
- Building and testing iOS apps (including Gate 0 compilation checks)
- Running simulators for screenshots (iPhone 16 Pro Max, iPhone 16, iPad Pro 13")
- Uploading to App Store Connect
- Running the lint script against real Xcode projects

If you're currently on WSL (Windows), you'll need to transfer the project to a Mac for the iOS-specific steps. The Control Panel, scripts, webhook server, and documentation all work on any platform.

### System Architecture Reference

```
┌─────────────────────────────────────────────────────┐
│                  OpenClaw Orchestrator               │
│         (Adaptive Polling + Event-Driven)            │
│    ┌──────────────────────────────────────────┐      │
│    │     Router (Sonnet) — routes projects     │     │
│    │     Budget check → Schema validate →      │     │
│    │     Spawn agent → Write phase_context     │     │
│    └──────────────────────────────────────────┘      │
├─────────────────────────────────────────────────────┤
│  Scout → Architect → Builder → Linter → Reviewer    │
│  (Sonnet) (Opus)    (Opus)   (Haiku)  (Codex)      │
│           ↓ auto-approve after 4hrs if score≥24     │
│                                   ↓                  │
│  Monetize ═══════╗  ← parallel →  Package           │
│  (Shipper)       ║                (Shipper)          │
│                  ╚══════════════════╗                 │
│                                Ship → Market         │
│                              (Shipper) (Marketer)    │
├─────────────────────────────────────────────────────┤
│  Webhook Server (:3002)  │  Control Panel (:5173)   │
│  Event-driven triggers   │  SvelteKit dashboard     │
├─────────────────────────────────────────────────────┤
│  factory.db (SQLite)  │  state.json (per project)   │
│  config/budget.json   │  schemas/*.schema.json      │
└─────────────────────────────────────────────────────┘
```
