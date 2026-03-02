# AppFactory - Claude Code Project Instructions

## Project Overview
AppFactory is an autonomous iOS app factory. AI agents research ideas, build SwiftUI apps, test them, add monetization, generate marketing assets, and submit to the App Store.

## Architecture
- **Orchestrator**: OpenClaw cron job (every 5 min) reads `projects/*/state.json` and spawns agents
- **7 Agents**: Router (Sonnet), Scout (Sonnet), Architect (Opus), Builder (Opus), Reviewer (Codex), Shipper (Sonnet), Marketer (Opus)
- **State**: All state in files (`state.json`, `research.json`, `spec.md`, `onepager.json`, `quality.json`)
- **Database**: SQLite (`factory.db`) for analytics and history
- **Dashboard**: SvelteKit + TailwindCSS control panel

## Key Directories
- `docs/` — 16 documentation files covering every aspect of the system
- `prompts/` — System prompts for each agent
- `schemas/` — JSON schemas for all data formats
- `config/` — OpenClaw, cron, and model configuration
- `templates/` — SwiftUI code templates, state file templates
- `projects/` — Active and completed app projects (each in its own subdirectory)
- `suggestions/` — User-submitted app ideas (JSON files)

## Rules
1. **State in files, never in memory**. Read from files, write to files.
2. **Max 5 concurrent projects**. The Router enforces this.
3. **Quality score >= 8.0/10 to ship**. 3 failures = flagged for human review.
4. **Apple Guideline 4.3 compliance is critical**. Every app must be genuinely different.
5. **The spec is the contract**. Builder implements exactly what Architect specifies.
6. **Different model for review**. Codex reviews what Opus builds.

## Tech Stack
- iOS apps: Swift 6 / SwiftUI / iOS 17+ / StoreKit 2
- Dashboard: SvelteKit 2 / Svelte 5 / TailwindCSS v4
- Orchestration: OpenClaw + cron
- Build automation: Fastlane
- Databases: SQLite (factory), SwiftData (in-app)
- AI: Anthropic (Opus/Sonnet), OpenAI (Codex), Google (Gemini Flash, Nano Banana Pro)
- Marketing: Postiz API, Remotion (video), Larry skill (content loop)
- Audit: ICP canisters (optional)

## When Working on This Project
- Read the relevant `docs/` file before making changes to any system component
- Follow the schemas in `schemas/` for all data files
- Agent prompts in `prompts/` are the source of truth for agent behavior
- Keep the README.md documentation table up to date if you add new docs
