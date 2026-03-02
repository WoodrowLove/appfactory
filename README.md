# AppFactory

**Autonomous iOS App Factory — From market research to App Store submission, powered by coordinated AI agents.**

---

## What This Is

AppFactory is a production-grade system that autonomously researches app ideas, builds iOS apps in Swift/SwiftUI, tests them, adds monetization, generates marketing assets, and submits them to the Apple App Store — with minimal human intervention.

It is not a toy. It is not a prototype. It is a money-generation tool that also happens to produce genuinely useful, differentiated apps that solve real problems for real people.

## Philosophy

**Ship fast. Validate with real keywords. Double down on what works. Kill what doesn't.**

Every app that leaves this factory must:
1. Solve a **real pain point** identified from actual user complaints
2. Be **genuinely differentiated** from every other app we've shipped and every competitor on the App Store
3. Pass **rigorous quality gates** that mirror Apple's own review standards
4. Have a **clear monetization path** validated by market data

We do not spam the App Store. We do not ship template clones. We ship meaningful tools that people need, and we ship them fast.

## How It Works

```
RESEARCH → VALIDATE → SPEC → BUILD → REVIEW → MONETIZE → PACKAGE → SHIP → MARKET
    │          │        │       │        │         │          │        │       │
  Scout     Scout   Architect Builder Reviewer  Shipper   Shipper  Shipper Marketer
 (Sonnet)  (Sonnet)  (Opus)   (Opus)  (Codex)  (Sonnet)  (Sonnet) (Sonnet) (Opus)
```

An **OpenClaw orchestrator** runs on a 5-minute cron cycle. It reads the state of every active project (max 5 concurrent), determines what phase each is in, and spawns the right Claude Code session with the right prompt for that phase. The session does the work, updates the state file, and exits. No wasted compute. No context bloat.

Each agent is a specialist. Each handles its own context. The orchestrator uses less than 5% of its context window to direct traffic.

## System Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Orchestrator** | Cron-based router, state manager | OpenClaw + Sonnet 4.6 |
| **Scout Agent** | Market research, pain point discovery | Sonnet 4.6 + Reddit/X/App Store APIs |
| **Architect Agent** | One-pager generation, feature specs | Opus 4.6 |
| **Builder Agent** | Full SwiftUI app generation | Opus 4.6 |
| **Reviewer Agent** | Independent code review, quality scoring | GPT-5.3-Codex |
| **Shipper Agent** | Fastlane pipeline, ASO, screenshots | Sonnet 4.6 |
| **Marketer Agent** | Content generation, social posting | Opus 4.6 + Larry skill |
| **Control Panel** | Dashboard for monitoring and approvals | SvelteKit + TailwindCSS |
| **State Store** | Persistent project state | JSON files + SQLite |
| **Audit Layer** | Trust and provenance tracking | ICP canisters |

## Control Panel

The SvelteKit dashboard is your window into the factory:

- **Home**: Active projects, shipped apps, queue length, items needing your attention
- **Projects**: Each app with progress bar and quality score (0-10). Tap to expand the one-pager showing every AI decision from idea to execution
- **Live Feed**: Real-time agent activity stream (what's building, what's reviewing, what's waiting)
- **Revenue**: MRR, trial conversions, churn, per-app breakdown
- **Marketing**: Content calendar, post performance, audience growth
- **Suggestions**: Submit your own app ideas into the pipeline

## Revenue Target

$5,000-$10,000/month within 6 months of first app launch.

At $4.99/month with 7-day free trial and 15% trial-to-paid conversion:
- 10 shipped apps
- 13-20 paying subscribers per app
- Total: 130-200 subscribers = $650-$1,000/month per pricing tier

Scale by shipping more apps and increasing conversion through iteration.

## Documentation

| Document | What It Covers |
|----------|---------------|
| [Architecture](docs/01_ARCHITECTURE.md) | System design, data flow, component interactions |
| [Agents](docs/02_AGENTS.md) | Every agent's role, model, tools, and constraints |
| [Pipeline](docs/03_PIPELINE.md) | End-to-end flow from idea to App Store |
| [Research Engine](docs/04_RESEARCH_ENGINE.md) | How ideas are discovered and validated |
| [Build System](docs/05_BUILD_SYSTEM.md) | SwiftUI generation, templates, ICP bridge |
| [Quality Gates](docs/06_QUALITY_GATES.md) | 6 automated checks, scoring methodology |
| [Monetization](docs/07_MONETIZATION.md) | StoreKit 2, free trials, RevenueCat analytics |
| [Packaging & Shipping](docs/08_PACKAGING_SHIPPING.md) | Fastlane, ASO, screenshots, icons, submission |
| [Marketing Engine](docs/09_MARKETING_ENGINE.md) | Social media automation, content, Larry skill |
| [Control Panel](docs/10_CONTROL_PANEL.md) | SvelteKit dashboard specification |
| [State Management](docs/11_STATE_MANAGEMENT.md) | State files, persistence, data flow |
| [Orchestrator](docs/12_ORCHESTRATOR.md) | OpenClaw cron configuration, agent spawning |
| [Apple Compliance](docs/13_APPLE_COMPLIANCE.md) | Guideline 4.3 strategy, differentiation |
| [Human Interface](docs/14_HUMAN_INTERFACE.md) | How you interact, suggest apps, approve submissions |
| [Deployment](docs/15_DEPLOYMENT.md) | Setup, installation, configuration |
| [Revenue Targets](docs/16_REVENUE_TARGETS.md) | Unit economics, financial projections |

## Agent Prompts

Production-ready system prompts for each agent live in [`prompts/`](prompts/). These are what the orchestrator feeds to Claude Code sessions.

## Quick Start

See [Deployment Guide](docs/15_DEPLOYMENT.md) for full setup instructions.

```bash
# Prerequisites: macOS with Xcode, OpenClaw, Claude Code, Node.js 20+
# 1. Clone the repo
git clone git@github.com:WoodrowLove/appfactory.git
cd appfactory

# 2. Install dependencies
npm install

# 3. Configure API keys
cp .env.example .env
# Edit .env with your keys (Anthropic, OpenAI, Apple Developer, etc.)

# 4. Start the control panel
npm run dev

# 5. Start the orchestrator
openclaw cron start --config config/openclaw.yaml
```

## Part of the Axia Ecosystem

AppFactory is a standalone revenue engine, but it connects to the broader ecosystem:

- **ICP Canisters**: Audit trails for every AI decision, trust scores for app quality
- **Namora AI**: Policy governance layer for agent behavior and quality enforcement
- **AxiaSystem**: Identity and subscription infrastructure (optional integration)

## License

Proprietary. All rights reserved.

---

*Built by Aaron Wright. Powered by coordinated AI agents.*
