# Deployment

## Prerequisites

### Hardware
- **Mac** (required for Xcode and iOS builds)
- macOS 14 Sonoma or later
- 16GB RAM minimum (32GB recommended for concurrent Xcode builds)
- 100GB+ free disk space (Xcode projects, simulators, and build artifacts)

### Software

| Tool | Version | Purpose | Install |
|------|---------|---------|---------|
| Xcode | 16+ | iOS builds, simulators | Mac App Store |
| Node.js | 20+ | Control Panel, scripts | `nvm install 20` |
| Ruby | 3.0+ | Fastlane | `brew install ruby` |
| Python | 3.11+ | Research scripts, tooling | `brew install python` |
| Git | Latest | Version control | Pre-installed on macOS |
| Homebrew | Latest | Package manager | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` |

### CLI Tools

| Tool | Purpose | Install |
|------|---------|---------|
| Claude Code | AI agent sessions | `npm install -g @anthropic-ai/claude-code` |
| OpenClaw | Orchestrator, cron, agent spawning | `npm install -g openclaw@latest` |
| Fastlane | iOS build automation | `gem install fastlane` |
| gh | GitHub CLI | `brew install gh` |
| jq | JSON processing | `brew install jq` |
| DFX | ICP canister deployment (optional) | `sh -ci "$(curl -fsSL https://internetcomputer.org/install.sh)"` |

### Accounts & API Keys

| Account | Purpose | Setup |
|---------|---------|-------|
| **Apple Developer** ($99/year) | App Store submission, code signing | [developer.apple.com](https://developer.apple.com) |
| **Anthropic API** | Claude Opus + Sonnet agents | [console.anthropic.com](https://console.anthropic.com) |
| **OpenAI API** | GPT-5.3-Codex reviewer | [platform.openai.com](https://platform.openai.com) |
| **Google AI Studio** | Gemini Flash (in-app AI), Nano Banana Pro (icons) | [aistudio.google.com](https://aistudio.google.com) |
| **RevenueCat** (free tier) | Subscription analytics | [revenuecat.com](https://www.revenuecat.com) |
| **Reddit API** (free) | Research crawling | [reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) |
| **X API** ($100/month) | Trend monitoring | [developer.x.com](https://developer.x.com) |
| **Postiz** (self-hosted or cloud) | Social media posting | [postiz.com](https://postiz.com) |

## Installation

### Step 1: Clone the Repository

```bash
git clone git@github.com:WoodrowLove/appfactory.git
cd appfactory
```

### Step 2: Install Node Dependencies

```bash
npm install
```

### Step 3: Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env` with your API keys:

```bash
# .env
# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# OpenAI (for Codex reviewer)
OPENAI_API_KEY=sk-...

# Google AI
GOOGLE_AI_API_KEY=AIza...

# Apple Developer
APPLE_TEAM_ID=XXXXXXXXXX
APPLE_DEVELOPER_EMAIL=woodrow227@icloud.com
MATCH_GIT_URL=git@github.com:WoodrowLove/appfactory-certs.git
MATCH_PASSWORD=...

# RevenueCat
REVENUECAT_API_KEY=appl_...
REVENUECAT_WEBHOOK_SECRET=...

# Reddit
REDDIT_CLIENT_ID=...
REDDIT_CLIENT_SECRET=...

# X / Twitter
X_BEARER_TOKEN=...

# Postiz
POSTIZ_API_URL=http://localhost:4200
POSTIZ_API_KEY=...

# ICP (optional)
DFX_NETWORK=ic
CANISTER_ID_AUDIT=...
```

### Step 4: Set Up Code Signing

Fastlane `match` manages certificates and provisioning profiles:

```bash
# Initialize match (first time only)
fastlane match init

# Generate App Store certificates
fastlane match appstore
```

This creates encrypted copies of your signing certificates in a private git repo.

### Step 5: Initialize the Database

```bash
node scripts/init-db.js
```

This creates `factory.db` with the schema from [State Management](11_STATE_MANAGEMENT.md).

### Step 6: Configure OpenClaw

```bash
# Verify OpenClaw is installed
openclaw --version

# Set up the cron job
openclaw cron create \
  --name "appfactory-router" \
  --schedule "*/5 * * * *" \
  --command "node scripts/router.js" \
  --config config/openclaw.yaml
```

### Step 7: Enable Claude Code Remote Control

```bash
# Configure Remote Control for all sessions
claude /config
# Set remote_control: "always"
```

### Step 8: Start the Control Panel

```bash
npm run dev
# Opens at http://localhost:5173
```

### Step 9: Start the Orchestrator

```bash
openclaw cron start
```

The factory is now running.

## Directory Setup After Installation

```
appfactory/
├── .env                    # API keys (gitignored)
├── factory.db              # SQLite database (gitignored)
├── node_modules/           # Dependencies (gitignored)
├── projects/               # Active and completed projects
├── suggestions/            # User suggestion queue
├── logs/                   # Agent and orchestrator logs (gitignored)
└── ... (everything else from the repo)
```

## Verification

### Check Everything Is Working

```bash
# 1. Verify Claude Code
claude --version

# 2. Verify OpenClaw
openclaw --version
openclaw cron list

# 3. Verify Fastlane
fastlane --version

# 4. Verify Xcode
xcodebuild -version

# 5. Verify API keys
node scripts/verify-keys.js

# 6. Verify code signing
fastlane match appstore --readonly

# 7. Check the Control Panel
open http://localhost:5173
```

### Run a Test Pipeline

To verify the full pipeline works:

```bash
# Create a test suggestion
cat > suggestions/test-app.json << 'EOF'
{
  "id": "test-app",
  "submitted": "2026-03-02T00:00:00Z",
  "idea": "A simple unit converter with a clean UI",
  "category": "utilities",
  "priority": "rush",
  "notes": "Test app to verify the pipeline",
  "status": "queued",
  "source": "human"
}
EOF

# Trigger a manual orchestrator run
node scripts/router.js

# Watch the logs
tail -f logs/orchestrator.log
```

## Updating

### Update Dependencies

```bash
# Node packages
npm update

# Ruby gems (Fastlane)
bundle update

# OpenClaw
npm update -g openclaw@latest

# Claude Code
npm update -g @anthropic-ai/claude-code
```

### Update Models

When new model versions become available, update `config/openclaw.yaml`:

```yaml
agents:
  builder:
    model: "opus-4.7"  # Updated from 4.6
  reviewer:
    model: "codex-5.4"  # Updated from 5.3
```

### Update Xcode

When Apple releases a new Xcode version:
1. Update via Mac App Store
2. Accept the license: `sudo xcodebuild -license accept`
3. Install additional simulators: Xcode → Settings → Platforms
4. Update `templates/swiftui/` if new APIs are available
5. Update minimum iOS target in templates if appropriate

## Troubleshooting

### Xcode Build Fails

```bash
# Clean derived data
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# Reset simulators
xcrun simctl shutdown all
xcrun simctl erase all

# Check Xcode command line tools
xcode-select --install
```

### Code Signing Issues

```bash
# Reset match certificates
fastlane match nuke appstore
fastlane match appstore

# Or manually revoke and regenerate
# Visit developer.apple.com → Certificates → Revoke
```

### OpenClaw Cron Not Running

```bash
# Check cron status
openclaw cron list
openclaw cron logs appfactory-router

# Restart
openclaw cron stop
openclaw cron start --config config/openclaw.yaml
```

### API Rate Limits

If agents are hitting rate limits:
1. Reduce `max_concurrent_agents` in `config/openclaw.yaml`
2. Increase cron interval from 5 minutes to 10 minutes
3. Check Anthropic usage dashboard for quota status

## Cost Estimation

### Monthly Costs (at full operation)

| Item | Estimated Cost | Notes |
|------|---------------|-------|
| Anthropic API (Opus + Sonnet) | $100-300 | Depends on number of builds |
| OpenAI API (Codex) | $30-80 | Code reviews are relatively cheap |
| Google AI (Gemini + Nano Banana) | $20-50 | Icons and in-app AI |
| Apple Developer Program | $8.25 | ($99/year ÷ 12) |
| X API | $100 | Basic tier for research |
| RevenueCat | $0 | Free under $2,500 MRR |
| Hosting (Postiz, etc.) | $20-50 | Self-hosted or minimal cloud |
| **Total** | **$280-590/month** | |

### Break-Even Analysis

At $280-590/month in costs:
- Need 56-118 subscribers at $4.99/month (after Apple's 30% cut)
- With 10 apps, that's 6-12 subscribers per app
- This is achievable within 2-3 months of first app launch

After break-even, the cost structure has very high operating leverage — costs stay roughly flat while revenue scales with number of apps and subscribers.

## Scaling

### Multi-Machine Architecture

Not every agent requires macOS. By splitting workloads across machines, you can 2-3x throughput without buying additional Macs.

**Machine roles:**

| Machine | OS | Agents / Services | Why This Machine |
|---------|----|--------------------|-----------------|
| Primary Mac | macOS 14+ | Builder, Shipper, Fastlane | Xcode and iOS simulators require macOS |
| Linux Server (or WSL) | Ubuntu 22+ | Router, Scout, Marketer, Control Panel | These agents only need Node.js, Python, and API access |

**What requires macOS (and why):**
- **Builder**: Needs `xcodebuild`, iOS simulators, and Swift compiler
- **Shipper**: Needs Fastlane (which wraps `xcodebuild` and `altool`/`xcrun`)
- **Code signing**: `fastlane match` requires macOS Keychain

**What does NOT require macOS:**
- **Router**: Reads `state.json` files and spawns agents. Pure Node.js.
- **Scout**: Calls Reddit API, X API, and LLM APIs. Pure API calls.
- **Architect**: Generates specs via LLM. No platform dependency.
- **Reviewer/Linter**: Reads source code and calls LLM APIs. No compilation needed.
- **Marketer**: Generates content and calls Postiz API. Pure API calls.
- **Control Panel**: SvelteKit app. Runs anywhere Node.js runs.

**Setup:**
```
┌──────────────────────────┐     ┌──────────────────────────┐
│      Linux Server        │     │       Primary Mac         │
│                          │     │                           │
│  Router (orchestrator)   │────▶│  Builder (Xcode builds)   │
│  Scout (research)        │     │  Shipper (Fastlane)       │
│  Architect (specs)       │     │  Code signing (Keychain)  │
│  Reviewer (code review)  │     │                           │
│  Linter (fast checks)    │     └──────────────────────────┘
│  Marketer (content)      │
│  Control Panel (UI)      │
│  factory.db (SQLite)     │
└──────────────────────────┘
```

Communication between machines uses the shared git repository and file system (via NFS, SSHFS, or synced working directories). The Router on the Linux server writes `state.json` transitions, and the Mac-based agents poll or receive webhooks when it is their turn.

**Alternatively**, use SSH-based remote execution: the Router on Linux triggers Builder/Shipper jobs on the Mac via `ssh mac-host "cd /path/to/appfactory && fastlane ios build"`.

This architecture means the Mac is only busy during actual compilation and submission — freeing it from the overhead of running the orchestrator, research, and marketing workloads.

## Future Considerations

### Apple Search Ads (Phase 3 — Paid Acquisition)

Once 5-10 apps are live and generating organic revenue, consider allocating a paid acquisition budget through Apple Search Ads.

**Strategy:**
- **Budget**: $50-100 per app per month (start conservative, scale what works)
- **Targeting**: High-intent keywords identified during the Scout's research phase
- **Campaign type**: Search Results campaigns (users see ads when searching relevant terms)
- **Attribution**: SKAdNetwork integration (already included in app templates) provides conversion tracking without compromising user privacy

**Typical performance for utility apps:**
| Metric | Expected Range |
|--------|---------------|
| Cost per tap (CPT) | $0.50-2.00 |
| Tap-through rate (TTR) | 5-10% |
| Conversion rate (installs) | 40-60% |
| Cost per install (CPI) | $1.00-4.00 |
| Return on ad spend (ROAS) | 3-5x over 12 months |

**Implementation:**
1. The Marketer extracts the top 10-20 keywords from `research.json` for each app
2. Keywords are grouped into campaigns by intent level (brand, category, competitor)
3. Apple Search Ads API is used to create and manage campaigns programmatically
4. Budget pacing is automated: increase spend on keywords with ROAS > 3x, pause keywords with ROAS < 1x
5. Weekly reports are generated and stored in `projects/<slug>/marketing/ads/`

**Prerequisites:**
- SKAdNetwork attribution configured in each app (already in templates)
- Apple Search Ads account linked to the developer account
- Minimum 30 days of organic data per app before enabling paid campaigns (establishes baseline metrics)

Paid acquisition is not needed for initial traction — organic marketing and ASO should be the primary growth drivers for the first 6 months. Apple Search Ads becomes valuable when you have proven product-market fit and want to accelerate growth for your best-performing apps.
