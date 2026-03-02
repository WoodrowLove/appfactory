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
3. Pass **rigorous quality gates** (8 automated gates + pre-review lint) that mirror Apple's own review standards
4. Have a **clear monetization path** validated by market data with hybrid contextual paywalls

We do not spam the App Store. We do not ship template clones. We ship meaningful tools that people need, and we ship them fast.

## How It Works

```
RESEARCH → VALIDATE → SPEC → BUILD → LINT → REVIEW → MONETIZE ═╗ → SHIP → MARKET
    │          │        │       │       │       │         │      ║      │       │
  Scout     Scout   Architect Builder Linter Reviewer  Shipper  ║   Shipper Marketer
 (Sonnet)  (Sonnet)  (Opus)   (Opus)  (Haiku) (Codex) (Sonnet) ║  (Sonnet)  (Opus)
                                                                 ║
                                                    PACKAGE ═════╝
                                                    (Shipper, parallel)
```

An **OpenClaw orchestrator** runs with adaptive polling (1 min when active, 15 min when idle, 5 min default). It reads the state of every active project (max 5 concurrent), determines what phase each is in, and spawns the right Claude Code session with the right prompt for that phase. An event-driven webhook on port 3002 eliminates dead time between phases.

Each agent is a specialist. Each handles its own context. The orchestrator uses less than 5% of its context window to direct traffic. Every agent output is schema-validated before phase transitions.

## 8 Agents

| Agent | Model | Purpose |
|-------|-------|---------|
| **Router** | Sonnet 4.6 | Orchestrator — routes projects, validates schemas, enforces budget |
| **Scout** | Sonnet 4.6 | Market research, pain point discovery, validation |
| **Architect** | Opus 4.6 | One-pager generation, feature specs, differentiation |
| **Builder** | Opus 4.6 | Full SwiftUI app generation with contextual paywalls, localization, cross-promo |
| **Linter** | Haiku 4 | Pre-review lint pass — catches 40% of common failures before expensive Codex review |
| **Reviewer** | GPT-5.3-Codex | Independent code review, 8 quality gates, incremental review on revisions |
| **Shipper** | Sonnet 4.6 | Fastlane pipeline, ASO, screenshots, regional pricing, phased release |
| **Marketer** | Opus 4.6 | Content generation, social posting, win-back campaigns, Larry skill |

## System Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Orchestrator** | Adaptive polling + event-driven routing | OpenClaw + Sonnet 4.6 |
| **8 Agents** | Specialized AI workers | Claude (Sonnet/Opus/Haiku) + GPT Codex |
| **Control Panel** | Dashboard for monitoring and approvals | SvelteKit + TailwindCSS |
| **State Store** | Persistent project state | JSON files + SQLite |
| **Schema Validator** | Validates all agent outputs | JSON Schema (Ajv) |
| **Budget Enforcer** | $500/month cap with per-agent cost tracking | config/budget.json |
| **Webhook Server** | Event-driven phase transitions | Express on port 3002 |
| **Audit Layer** | Trust and provenance tracking | ICP canisters |

## Pipeline Features

- **Gate 0 (Compilation)**: Router runs `xcodebuild build` after Builder completes — catches broken builds before Reviewer
- **Pre-Review Lint**: Haiku-based fast pass catches force unwraps, missing permissions, dead code, placeholder content
- **8 Quality Gates**: Crash Safety, Permission Handling, Feature Completeness, UI/UX, App Store Compliance, Code Quality, Test Coverage (60% min), Monetization UX
- **Incremental Review**: On revision attempt 2+, only re-check failed gates (50-60% time savings)
- **Parallel Phases**: Monetize and Package run simultaneously (saves 20-40 min)
- **Auto-Approval**: Specs auto-approve after 4 hours if validation score >= 24/30. Submissions never auto-approve.
- **Contextual Paywalls**: Hybrid strategy — onboarding + feature-gated (2-3x better conversion)
- **Schema Validation**: Every agent output validated against schemas/ before phase transition
- **Budget Enforcement**: Pre-spawn check against $500/month cap, alerts at 80%, hard stop at 100%

## Control Panel

The SvelteKit dashboard is your window into the factory:

- **Home**: Active projects, shipped apps, queue length, items needing your attention
- **Projects**: Each app with progress bar, quality score (0-10), lint status, and phase context. Tap to expand the one-pager showing every AI decision from idea to execution
- **Live Feed**: Real-time agent activity stream (what's building, what's reviewing, what's waiting)
- **Revenue**: MRR, trial conversions, churn, per-app breakdown, paywall conversion comparison, experiment results
- **Marketing**: Content calendar, post performance, audience growth, win-back campaign status
- **Suggestions**: Submit your own app ideas into the pipeline
- **Settings**: Budget dashboard, model assignments, configuration

## Revenue Target

$5,000-$10,000/month within 6 months of first app launch.

At $4.99/month with 7-day free trial, contextual paywalls, and 15% trial-to-paid conversion:
- 10 shipped apps
- 13-20 paying subscribers per app
- Total: 130-200 subscribers = $650-$1,000/month per pricing tier
- Plus: lifetime purchases ($79.99), regional pricing, RevenueCat price experiments, win-back campaigns

Scale by shipping more apps and increasing conversion through iteration.

## Documentation

| Document | What It Covers |
|----------|---------------|
| [Architecture](docs/01_ARCHITECTURE.md) | System design, data flow, component interactions |
| [Agents](docs/02_AGENTS.md) | All 8 agents — roles, models, tools, constraints |
| [Pipeline](docs/03_PIPELINE.md) | End-to-end flow with lint phase, parallel stages, phase context |
| [Research Engine](docs/04_RESEARCH_ENGINE.md) | How ideas are discovered and validated |
| [Build System](docs/05_BUILD_SYSTEM.md) | SwiftUI generation, templates, ICP bridge |
| [Quality Gates](docs/06_QUALITY_GATES.md) | 8 automated gates + Gate 0 compilation + pre-review lint |
| [Monetization](docs/07_MONETIZATION.md) | StoreKit 2, contextual paywalls, lifetime purchase, regional pricing, experiments |
| [Packaging & Shipping](docs/08_PACKAGING_SHIPPING.md) | Fastlane, ASO, screenshots, icons, phased release, TestFlight soak |
| [Marketing Engine](docs/09_MARKETING_ENGINE.md) | Social media automation, content, Larry skill, win-back campaigns |
| [Control Panel](docs/10_CONTROL_PANEL.md) | SvelteKit dashboard specification |
| [State Management](docs/11_STATE_MANAGEMENT.md) | State files, persistence, data flow |
| [Orchestrator](docs/12_ORCHESTRATOR.md) | Adaptive polling, event-driven webhook, budget enforcement, schema validation |
| [Apple Compliance](docs/13_APPLE_COMPLIANCE.md) | Guideline 4.3 strategy, differentiation, multi-account strategy |
| [Human Interface](docs/14_HUMAN_INTERFACE.md) | How you interact, suggest apps, approve submissions |
| [Deployment](docs/15_DEPLOYMENT.md) | Setup, installation, configuration |
| [Revenue Targets](docs/16_REVENUE_TARGETS.md) | Unit economics, financial projections |

## Configuration

| File | Purpose |
|------|---------|
| `config/openclaw.yaml` | Orchestrator config — adaptive polling, webhook, schema validation, auto-approval, agent definitions |
| `config/budget.json` | $500/month budget, per-model cost tables, enforcement rules |
| `config/models.yaml` | Model assignments — single source of truth for model IDs |
| `config/cron.json` | Cron job definitions for all scheduled tasks |

## Agent Prompts

Production-ready system prompts for each agent live in [`prompts/`](prompts/). These are what the orchestrator feeds to Claude Code sessions.

| Prompt | Agent |
|--------|-------|
| `prompts/router.md` | Router — orchestration and routing logic |
| `prompts/scout.md` | Scout — research and validation modes |
| `prompts/architect.md` | Architect — one-pager generation |
| `prompts/builder.md` | Builder — app generation with contextual paywalls, localization, cross-promo |
| `prompts/linter.md` | Linter — 7-category pre-review lint pass |
| `prompts/reviewer.md` | Reviewer — 8 quality gates with incremental review mode |
| `prompts/shipper.md` | Shipper — monetize, package, and submit modes |
| `prompts/marketer.md` | Marketer — content generation and social posting |

## Schemas

All agent outputs are validated against JSON schemas in [`schemas/`](schemas/):

| Schema | Validates |
|--------|-----------|
| `schemas/state.schema.json` | Project state (phase, context, fixing, approvals, revenue) |
| `schemas/onepager.schema.json` | App spec (features, monetization, ICP integration, localization) |
| `schemas/research.schema.json` | Research output (sources, trends, scoring, failures) |
| `schemas/quality.schema.json` | Quality review (8 gates, scores, issues) |
| `schemas/listing.schema.json` | App Store listing (ASO, localized listings, phased release) |

## Quick Start

See [Deployment Guide](docs/15_DEPLOYMENT.md) for full setup instructions.

```bash
# Prerequisites: macOS with Xcode 16+, OpenClaw, Claude Code, Node.js 20+
# 1. Clone the repo
git clone git@github.com:WoodrowLove/appfactory.git
cd appfactory

# 2. Install dependencies
npm install

# 3. Configure API keys
cp .env.example .env
# Edit .env with your keys (Anthropic, OpenAI, Apple Developer, etc.)

# 4. Start everything
./scripts/start.sh
# This starts: webhook server (:3002), control panel (:5173), cron jobs
```

Or use the [Terminal Agent Prompt](TERMINAL_AGENT_PROMPT.md) to have Claude Code build all the executable code from the documentation.

## Part of the Axia Ecosystem

AppFactory is a standalone revenue engine, but it connects to the broader ecosystem:

- **ICP Canisters**: Audit trails for every AI decision, trust scores for app quality
- **Namora AI**: Policy governance layer for agent behavior and quality enforcement
- **AxiaSystem**: Identity and subscription infrastructure (optional integration)

## License

Proprietary. All rights reserved.

---

*Built by Aaron Wright. Powered by coordinated AI agents.*
