# Architecture

## System Overview

AppFactory is a **multi-agent, cron-driven pipeline** that transforms market research into shipped iOS applications. The system is designed around three principles:

1. **Context isolation**: Each agent operates in its own session with only the context it needs. No single agent holds the full system state.
2. **State-driven execution**: All important state lives in files, never in conversation history. The orchestrator reads state files and decides what happens next.
3. **Human-in-the-loop at gates**: The system runs autonomously but pauses at critical decision points for human approval.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HUMAN LAYER                                  │
│                                                                     │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│   │ SvelteKit   │  │ Claude iOS   │  │ App Store Connect        │  │
│   │ Dashboard   │  │ Remote Ctrl  │  │ (Submit Button)          │  │
│   └──────┬──────┘  └──────┬───────┘  └──────────┬───────────────┘  │
│          │                │                      │                  │
└──────────┼────────────────┼──────────────────────┼──────────────────┘
           │                │                      │
┌──────────┼────────────────┼──────────────────────┼──────────────────┐
│          ▼                ▼                      ▼                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    ORCHESTRATOR (Router)                      │   │
│  │              OpenClaw + Sonnet 4.6                            │   │
│  │              Adaptive polling (1 min active / 15 min idle /    │   │
│  │              5 min default) + event-driven webhooks on :3002  │   │
│  │              Reads: projects/*/state.json                     │   │
│  │              Spawns: Claude Code sessions per phase           │   │
│  └──────┬──────────┬──────────┬──────────┬──────────┬───────────┘   │
│         │          │          │          │          │                │
│  ┌──────▼───┐ ┌────▼────┐ ┌──▼───┐ ┌──▼────┐ ┌────▼────┐ ┌──▼──────┐ │
│  │  Scout   │ │Architect│ │Builder│ │Linter │ │Reviewer │ │ Shipper │ │
│  │ (Sonnet) │ │ (Opus)  │ │(Opus) │ │(Haiku)│ │(Codex)  │ │(Sonnet) │ │
│  └──────┬───┘ └────┬────┘ └──┬───┘ └──┬────┘ └────┬────┘ └──┬──────┘ │
│         │          │          │          │          │    ┌─────────┐ │
│         │          │          │          │          │    │Marketer │ │
│         │          │          │          │          │    │ (Opus)  │ │
│         │          │          │          │          │    └────┬────┘ │
│         │          │          │          │          │         │      │
│  ┌──────▼──────────▼──────────▼──────────▼──────────▼─────────▼──┐  │
│  │                     STATE LAYER                                │  │
│  │  projects/<slug>/state.json     (project lifecycle)            │  │
│  │  projects/<slug>/research.json  (market data)                  │  │
│  │  projects/<slug>/spec.md        (one-pager)                    │  │
│  │  projects/<slug>/src/           (Xcode project)                │  │
│  │  projects/<slug>/quality.json   (review scores)                │  │
│  │  projects/<slug>/assets/        (icons, screenshots, video)    │  │
│  │  projects/<slug>/listing.json   (ASO metadata)                 │  │
│  │  factory.db                     (SQLite: history, analytics)   │  │
│  └──────────────────────────┬────────────────────────────────────┘  │
│                             │                                       │
│  ┌──────────────────────────▼────────────────────────────────────┐  │
│  │                     EXTERNAL SERVICES                          │  │
│  │                                                                │  │
│  │  Reddit API    X/Twitter API    App Store Connect API          │  │
│  │  Apple Search Ads API    RevenueCat    Fastlane                │  │
│  │  Nano Banana Pro (icons)    Remotion (video)                   │  │
│  │  Postiz (social posting)    ICP Canisters (audit)              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│                         AGENT LAYER                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Orchestrator (Router)

The brain of the system. Runs on adaptive polling (1 min active / 15 min idle / 5 min default) with event-driven webhook transitions on port 3002.

**Responsibilities:**
- Read all `projects/*/state.json` files
- Count active projects (enforce max 5 concurrent)
- Determine what phase each project is in
- Spawn the correct Claude Code session with the correct prompt
- Track the submission queue (max 5 in Apple review simultaneously)
- Pull new ideas from the research queue when slots open
- Log all decisions to `factory.db`

**What it does NOT do:**
- Build code
- Review code
- Make creative decisions
- Hold conversation history

The orchestrator's context window usage should be under 5%. It reads JSON, makes routing decisions, and spawns sessions. That's it.

### 2. Scout Agent

**Model**: Sonnet 4.6
**Phase**: Research + Validate
**Context**: Market data, competitor analysis, App Store metadata

Crawls Reddit, X, Product Hunt, and App Store reviews. Identifies pain points, validates demand, checks competition, and scores ideas. Outputs structured JSON that the Architect can consume without needing the Scout's full context.

### 3. Architect Agent

**Model**: Opus 4.6
**Phase**: Spec
**Context**: Research JSON + spec templates

Takes validated research and produces a comprehensive one-pager: target user, pain points, feature list, screen-by-screen layout, monetization strategy, differentiation analysis. This document is the contract between research and build.

### 4. Builder Agent

**Model**: Opus 4.6
**Phase**: Build
**Context**: Spec + SwiftUI templates + StoreKit templates

Generates the full Xcode project. Receives the spec, templates for common patterns (payments, onboarding, AI wrapper), and coding standards. Produces a complete, compilable SwiftUI application.

### 5. Linter Agent

**Model**: Claude Haiku 4
**Phase**: Lint (between Build and Review)
**Context**: Source code + one-pager spec

Runs a fast, cheap lint pass after the Builder produces code and Gate 0 (compilation) passes. Checks 7 categories: force unwraps, missing permissions, dead code, force casts, missing restore purchases, missing privacy manifest, and placeholder content. Produces a structured lint report. Hard failures send the project back to REVISING; soft failures are forwarded to the Reviewer as advisory notes.

### 6. Reviewer Agent

**Model**: GPT-5.3-Codex
**Phase**: Review
**Context**: Full source code + quality rubric

Independently reads every file the Builder produced. Runs 8 scored quality gates + Gate 0 compilation check. Produces a structured quality report with a score out of 10. This agent never sees the Builder's reasoning — it evaluates the output cold.

### 7. Shipper Agent

**Model**: Sonnet 4.6
**Phase**: Monetize + Package + Ship
**Context**: Source code + quality report + ASO templates

Handles the entire post-build pipeline: StoreKit configuration, Fastlane setup, screenshot generation, icon generation (Nano Banana Pro), App Store listing copy, and submission.

### 8. Marketer Agent

**Model**: Opus 4.6 (creative) + gpt-image-1.5 (visuals)
**Phase**: Market (runs in parallel with other phases)
**Context**: App specs + audience analytics + content templates

Generates social media content, manages posting schedules via Postiz, creates promo videos via Remotion, and tracks per-post performance. Operates somewhat independently — the marketing flywheel runs continuously, not just when an app ships.

## Data Flow

```
                    ┌─────────────────┐
                    │   Reddit / X /  │
                    │  App Store Data  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   SCOUT AGENT   │
                    │                 │
                    │ Output:         │
                    │ research.json   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ ARCHITECT AGENT │
                    │                 │
                    │ Output:         │
                    │ spec.md         │
                    │ onepager.json   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  BUILDER AGENT  │
                    │                 │
                    │ Output:         │
                    │ src/ (Xcode)    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  LINTER AGENT   │
                    │                 │
                    │ Output:         │
                    │ lint_report.json│
                    └────────┬────────┘
                             │
                      Pass / soft_fail?
                      ┌──────┴──────┐
                      │ YES         │ hard_fail
                      ▼             ▼
                      │        Back to BUILD
                      │
                    ┌─────────▼───────┐
                    │ REVIEWER AGENT  │
                    │                 │
                    │ Output:         │
                    │ quality.json    │
                    └────────┬────────┘
                             │
                      Pass (≥8/10)?
                      ┌──────┴──────┐
                      │ YES         │ NO (retry or flag)
                      ▼             ▼
             ┌──────────────┐  ┌──────────────┐
             │SHIPPER AGENT │  │ Back to BUILD │
             │              │  │ (max 3 tries) │
             │ Output:      │  └──────────────┘
             │ assets/      │
             │ listing.json │
             │ IPA binary   │
             └──────┬───────┘
                    │
                    ▼
           ┌──────────────┐
           │ SUBMISSION    │
           │ QUEUE         │
           │ (max 5 in    │
           │  review)      │
           └──────┬───────┘
                  │
                  ▼
           HUMAN PRESSES SUBMIT
                  │
                  ▼
           Apple Review
```

## State Machine

Every project follows this state machine:

```
                    ┌──────────┐
        ┌──────────│  QUEUED   │
        │          └─────┬─────┘
        │                │ (slot available)
        │          ┌─────▼──────┐
        │          │ RESEARCHING │
        │          └─────┬──────┘
        │                │ (research complete)
        │          ┌─────▼──────┐
        │          │ VALIDATING  │
        │          └─────┬──────┘
        │                │ (score ≥ 21/30)
        │          ┌─────▼──────┐
        │          │  SPECCING   │
        │          └─────┬──────┘
        │                │ (spec approved)
        │          ┌─────▼──────┐
        │          │  BUILDING   │
        │          └─────┬──────┘
        │                │ (build complete + Gate 0 passes)
        │          ┌─────▼──────┐
        │          │  LINTING    │
        │          └─────┬──────┘
        │                │ (lint pass or soft_fail)
        │          ┌─────▼──────┐
        │     ┌────│ REVIEWING   │────┐
        │     │    └─────────────┘    │
        │     │ (score ≥ 8)     (score < 8, attempts < 3)
        │     │                       │
        │     │    ┌─────────────┐    │
        │     │    │  REVISING   │◄───┘
        │     │    └──────┬──────┘
        │     │           │ (back to BUILDING)
        │     │           │
        │     ▼
        │  ┌────────────┐
        │  │ MONETIZING  │
        │  └─────┬──────┘
        │        │
        │  ┌─────▼──────┐
        │  │ PACKAGING   │
        │  └─────┬──────┘
        │        │
        │  ┌─────▼──────┐
        │  │READY_TO_SHIP│ ◄── Human approves
        │  └─────┬──────┘
        │        │
        │  ┌─────▼──────┐
        │  │ IN_REVIEW   │ ◄── Apple reviewing
        │  └─────┬──────┘
        │        │
        │  ┌─────▼──────┐        ┌──────────┐
        │  │  APPROVED   │        │ REJECTED │
        │  └─────┬──────┘        └─────┬────┘
        │        │                     │
        │  ┌─────▼──────┐        ┌─────▼────┐
        │  │   LIVE      │        │ FIXING   │
        │  └─────┬──────┘        └──────────┘
        │        │
        │  ┌─────▼──────┐
        │  │ MARKETING   │ (ongoing)
        │  └─────────────┘
        │
        │  ┌─────────────┐
        └──│   KILLED     │ ◄── Validation failed or 3 review failures
           └─────────────┘
```

## Directory Structure

```
appfactory/
├── .claude/
│   └── CLAUDE.md                     # Claude Code project instructions
├── config/
│   ├── openclaw.yaml                 # Orchestrator configuration
│   ├── cron.json                     # Cron schedule
│   ├── models.yaml                   # Model assignments per agent
│   └── budget.json                   # Per-agent budget limits and tracking
├── docs/                             # This documentation
├── prompts/
│   ├── router.md                     # Orchestrator system prompt
│   ├── scout.md                      # Research agent prompt
│   ├── architect.md                  # Spec agent prompt
│   ├── builder.md                    # Build agent prompt
│   ├── linter.md                     # Lint agent prompt
│   ├── reviewer.md                   # Review agent prompt
│   ├── shipper.md                    # Ship agent prompt
│   └── marketer.md                   # Marketing agent prompt
├── templates/
│   ├── swiftui/                      # SwiftUI code templates
│   │   ├── AppTemplate/              # Base Xcode project structure
│   │   ├── Onboarding.swift          # 3-5 screen onboarding flow
│   │   ├── Paywall.swift             # StoreKit 2 paywall template
│   │   ├── Settings.swift            # Standard settings screen
│   │   └── GeminiWrapper.swift       # Gemini Flash AI integration
│   ├── fastlane/                     # Fastlane configuration templates
│   │   ├── Fastfile                  # Build + deploy lanes
│   │   ├── Appfile                   # App identity
│   │   └── Snapfile                  # Screenshot configuration
│   └── marketing/                    # Content templates
│       ├── tiktok_hook.md            # TikTok script template
│       └── app_promo.md              # App promo video script
├── schemas/
│   ├── state.schema.json             # Project state file schema
│   ├── research.schema.json          # Research output schema
│   ├── onepager.schema.json          # One-pager document schema
│   ├── quality.schema.json           # Quality report schema
│   └── listing.schema.json           # App Store listing schema
├── projects/                         # Active and completed projects
│   └── <app-slug>/
│       ├── state.json                # Current state + metadata
│       ├── research.json             # Scout output
│       ├── spec.md                   # Architect output (one-pager)
│       ├── onepager.json             # Structured one-pager data
│       ├── src/                      # Builder output (Xcode project)
│       ├── lint_report.json          # Linter output
│       ├── quality.json              # Reviewer output
│       ├── assets/
│       │   ├── icon.png              # Generated app icon
│       │   ├── screenshots/          # Generated screenshots
│       │   └── promo.mp4             # Generated promo video
│       └── listing.json              # App Store listing metadata
├── suggestions/
│   ├── README.md                     # How to suggest apps
│   └── *.json                        # User-submitted app suggestions
├── factory.db                        # SQLite: analytics, history, logs
├── package.json                      # SvelteKit control panel deps
└── src/                              # SvelteKit control panel source
    └── ...
```

## Security Model

### API Key Management
- All API keys stored in environment variables, never in state files
- `.env` file is gitignored
- Each agent session receives only the keys it needs
- Apple Developer credentials are scoped to the Shipper agent only

### Agent Isolation
- Agents cannot access each other's sessions
- The Builder cannot see Reviewer feedback until the next cycle
- The Marketer operates on a separate schedule from the build pipeline
- No agent has access to the full factory.db — they write to their project's state files

### ICP Audit Trail
- Every state transition is logged to an ICP canister
- Quality scores are immutable once written
- Submission decisions and their reasoning are permanently recorded
- This creates a tamper-proof record of every decision the factory makes

## Scaling Considerations

### Current Design (v1)
- Single machine (your Mac)
- Max 5 concurrent projects
- OpenClaw cron on 5-minute intervals
- SQLite for analytics

### Future Scaling (v2)
- Multiple machines via Claude Code Remote Control
- Increase concurrent project limit
- PostgreSQL for analytics (shared across machines)
- Dedicated CI/CD runners for Fastlane builds
- Kubernetes for the control panel

The v1 design is intentionally simple. Ship the pipeline first, optimize later.

## Integration Points

### Apple Ecosystem
- **Xcode**: Build and compile (requires macOS)
- **App Store Connect API**: Metadata, screenshots, submission
- **TestFlight**: Beta testing (optional)
- **StoreKit 2**: In-app purchases and subscriptions
- **Apple Search Ads API**: Keyword research and competition data

### AI Models
- **Anthropic API**: Claude Opus 4.6 (build, architect, marketing), Sonnet 4.6 (routing, shipping, research), Haiku 4 (linting)
- **OpenAI API**: GPT-5.3-Codex (code review)
- **Google AI**: Gemini Flash (in-app AI wrapper), Nano Banana Pro (icon generation)

### Social & Marketing
- **Postiz**: Social media scheduling and posting
- **Reddit API**: Pain point research (read-only)
- **X/Twitter API**: Trend monitoring (read-only)
- **Remotion**: Programmatic video generation

### Infrastructure
- **OpenClaw**: Agent orchestration and cron scheduling
- **ICP**: Audit trail canisters
- **RevenueCat**: Subscription analytics and management
- **Fastlane**: iOS build, test, and deploy automation
