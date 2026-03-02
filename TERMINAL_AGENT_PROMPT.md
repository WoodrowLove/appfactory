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

Then complete the following setup tasks in order:

### Phase 1: SvelteKit Control Panel

1. Initialize a SvelteKit project in the repo root:
   - Use SvelteKit 2 with Svelte 5
   - Add TailwindCSS v4
   - Add better-sqlite3 for database access
   - Add chokidar for file watching

2. Build the Control Panel with these pages:
   - Dashboard (home): metric cards, live activity feed, needs-attention section
   - Projects: list of projects with progress bars and quality scores, expandable one-pager view
   - Revenue: MRR charts, per-app breakdown, conversion funnel
   - Marketing: content calendar, performance metrics
   - Suggestions: form to submit app ideas, list of existing suggestions
   - Settings: configuration management

3. Implement the backend:
   - API routes to read projects/*/state.json files
   - API routes to read/write factory.db (SQLite)
   - API route to write suggestion files to suggestions/
   - WebSocket endpoint for live activity feed
   - File watcher on projects/ directory for real-time updates

4. Initialize factory.db with the schema from docs/11_STATE_MANAGEMENT.md

### Phase 2: OpenClaw Orchestrator Scripts

5. Create these Node.js scripts in a scripts/ directory:
   - scripts/router.js — Main orchestrator logic (reads state files, makes routing decisions, spawns agents)
   - scripts/session-complete.js — Callback handler when an agent session completes
   - scripts/marketing-scheduler.js — Checks marketing content schedule
   - scripts/revenue-sync.js — Pulls RevenueCat data
   - scripts/review-status.js — Checks App Store Connect review status
   - scripts/backup.js — Daily backup of factory.db and state files
   - scripts/init-db.js — Database initialization
   - scripts/verify-keys.js — API key verification

### Phase 3: SwiftUI Templates

6. Create the SwiftUI template files in templates/swiftui/:
   - AppTemplate/ — Base Xcode project structure that compiles
   - Onboarding.swift — Configurable TabView-based onboarding
   - Paywall.swift — StoreKit 2 SubscriptionStoreView paywall
   - Settings.swift — Standard settings with privacy, restore, version
   - GeminiWrapper.swift — Gemini Flash API integration

### Phase 4: Fastlane Templates

7. Create Fastlane template files in templates/fastlane/:
   - Fastfile — Build, screenshot, and deploy lanes
   - Appfile — App identity template
   - Snapfile — Screenshot configuration

### Phase 5: Integration

8. Set up the OpenClaw cron configuration:
   - Register the cron jobs from config/cron.json
   - Test that the router script runs without errors (in dry-run mode)

9. Verify the complete system:
   - Control Panel starts and shows the dashboard
   - Database is initialized with the correct schema
   - File watcher detects changes in projects/
   - Template files are valid

### Important Notes:
- This project runs on macOS (required for Xcode). You are running in a terminal on that machine.
- Read the existing documentation before implementing — the docs are comprehensive.
- Follow the schemas in schemas/ for all data structures.
- The agent prompts in prompts/ define each agent's behavior.
- Do NOT install OpenClaw or Claude Code if not already installed — just set up the project code.
- Ask me before making any decisions that aren't covered in the documentation.
- If you encounter an error, fix it. Don't skip steps.
- Commit your work as you go with clear commit messages.
```

---

## What This Prompt Does

This prompt instructs a terminal Claude Code agent to build all the executable code for the AppFactory system. The documentation (which is already complete in the repo) serves as the specification. The agent reads the docs and implements the system accordingly.

### Expected Timeline
- Phase 1 (Control Panel): 30-60 minutes
- Phase 2 (Scripts): 20-30 minutes
- Phase 3 (SwiftUI Templates): 15-25 minutes
- Phase 4 (Fastlane Templates): 10-15 minutes
- Phase 5 (Integration): 10-15 minutes

### What You Need to Do After
1. Copy your API keys into `.env`
2. Set up Fastlane match for code signing: `fastlane match init && fastlane match appstore`
3. Install OpenClaw: `npm install -g openclaw@latest`
4. Register the cron jobs: `openclaw cron create --config config/openclaw.yaml`
5. Start the Control Panel: `npm run dev`
6. Start the orchestrator: `openclaw cron start`

### What Requires a Mac
The following must be done on macOS:
- Xcode installation and configuration
- Fastlane match setup (code signing)
- Building and testing iOS apps
- Running simulators for screenshots
- Uploading to App Store Connect

If you're currently on WSL (Windows), you'll need to transfer the project to a Mac for the iOS-specific steps. The Control Panel, scripts, and documentation all work on any platform.
