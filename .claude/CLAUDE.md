# AppFactory - Claude Code Project Instructions

## Project Overview
AppFactory is an autonomous iOS app factory. AI agents research ideas, build SwiftUI apps, test them, add monetization, generate marketing assets, and submit to the App Store.

## Architecture
- **Orchestrator**: OpenClaw with adaptive polling (1 min active / 15 min idle) + event-driven webhook transitions
- **8 Agents**: Router (Sonnet), Scout (Sonnet), Architect (Opus), Builder (Opus), Linter (Haiku), Reviewer (Codex), Shipper (Sonnet), Marketer (Opus)
- **State**: All state in files (`state.json`, `research.json`, `spec.md`, `onepager.json`, `lint_report.json`, `quality.json`)
- **Database**: SQLite (`factory.db`) for analytics and history
- **Dashboard**: SvelteKit + TailwindCSS control panel

## Key Directories
- `docs/` — 16 documentation files covering every aspect of the system
- `prompts/` — System prompts for each agent (8 prompts including linter)
- `schemas/` — JSON schemas for all data formats
- `config/` — OpenClaw yaml, cron schedule, model assignments, budget tracking
- `templates/` — SwiftUI code templates, state file templates
- `projects/` — Active and completed app projects (each in its own subdirectory)
- `suggestions/` — User-submitted app ideas (JSON files)

## Rules
1. **State in files, never in memory**. Read from files, write to files.
2. **Max 5 concurrent projects**. The Router enforces this.
3. **Quality score >= 8.0/10 to ship**. 8 scored gates + Gate 0 compilation check. 3 failures = flagged for human review.
4. **Apple Guideline 4.3 compliance is critical**. Every app must be genuinely different.
5. **The spec is the contract**. Builder implements exactly what Architect specifies.
6. **Different model for review**. Codex reviews what Opus builds. Haiku lints before Codex runs.
7. **Schema validation on every transition**. Router validates JSON outputs against schemas/ before phase changes.
8. **Budget pre-check before spawning**. Router checks config/budget.json before spawning any agent.
9. **Model IDs live in config only**. Docs use friendly names (Opus, Sonnet, Codex). Config files use API model IDs. Change models in config/models.yaml and config/openclaw.yaml only.

## Pipeline (10 Phases)
```
RESEARCH → VALIDATE → SPEC → BUILD → LINT → REVIEW → MONETIZE ─┐
                                                                 ├→ (parallel) → SHIP → MARKET
                                                      PACKAGE ──┘
```

## Tech Stack
- iOS apps: Swift 6 / SwiftUI / iOS 17+ / StoreKit 2
- Dashboard: SvelteKit 2 / Svelte 5 / TailwindCSS v4
- Orchestration: OpenClaw (adaptive poll + event-driven webhook)
- Build automation: Fastlane + optional Xcode Cloud
- Databases: SQLite (factory), SwiftData (in-app)
- AI: Anthropic (Opus/Sonnet/Haiku), OpenAI (Codex), Google (Gemini Flash, Nano Banana Pro)
- Marketing: Postiz API, Remotion (video), Larry skill (content loop)
- Monetization: StoreKit 2 + RevenueCat (experiments, analytics, win-back)
- Audit: ICP canisters (optional)
- Attribution: SKAdNetwork (for future Apple Search Ads)

## When Working on This Project
- Read the relevant `docs/` file before making changes to any system component
- Follow the schemas in `schemas/` for all data files
- Agent prompts in `prompts/` are the source of truth for agent behavior
- Keep the README.md documentation table up to date if you add new docs
- Validate JSON output against schemas before writing (use ajv or equivalent)
- All device references should use iPhone 16 Pro Max / iPhone 16 / iPad Pro 13" (M4)
