# Orchestrator

## Overview

The Orchestrator is an OpenClaw cron job that runs every 5 minutes. It reads the state of every project, makes routing decisions, and spawns the right Claude Code session for each project's current phase. It is the brain of the factory but does none of the work itself.

## Why OpenClaw

OpenClaw was selected over Google Antigravity for the orchestrator role because:

| Feature | OpenClaw | Antigravity |
|---------|----------|-------------|
| Cron scheduling | Native (`cron` command) | Not supported (IDE-first) |
| Headless execution | Yes (CLI) | No (requires VS Code window) |
| State file management | Built-in persistent state | Not applicable |
| Claude Code session spawning | `sessions_spawn` command | Not applicable |
| Heartbeat/monitoring | Native | Manager View (visual only) |
| Agent communication | ACP bridge | Multi-agent within IDE only |
| Cost | Free (open source) | Free (open source) |

**Antigravity remains useful** as an optional IDE when you want to manually observe or debug agents in real-time. But the autonomous cron-based factory runs on OpenClaw.

## OpenClaw Configuration

### `config/openclaw.yaml`

```yaml
# AppFactory Orchestrator Configuration
version: "1.0"

orchestrator:
  name: "appfactory-router"
  model: "sonnet-4.6"
  cron: "*/5 * * * *"  # Every 5 minutes
  timezone: "America/New_York"

  # Context budget control
  max_context_tokens: 4000  # Keep under 5% of context window
  state_summary_only: true  # Only read state.json, not full project data

  # Concurrency limits
  max_active_projects: 5
  max_in_review: 5
  max_concurrent_agents: 3  # Don't overwhelm the machine

  # Paths
  projects_dir: "./projects"
  suggestions_dir: "./suggestions"
  prompts_dir: "./prompts"
  database: "./factory.db"

agents:
  scout:
    model: "sonnet-4.6"
    prompt_file: "./prompts/scout.md"
    timeout_minutes: 45
    max_retries: 2

  architect:
    model: "opus-4.6"
    prompt_file: "./prompts/architect.md"
    timeout_minutes: 60
    max_retries: 1

  builder:
    model: "opus-4.6"
    prompt_file: "./prompts/builder.md"
    timeout_minutes: 120
    max_retries: 2

  reviewer:
    model: "codex-5.3"
    prompt_file: "./prompts/reviewer.md"
    timeout_minutes: 30
    max_retries: 1
    provider: "openai"

  shipper:
    model: "sonnet-4.6"
    prompt_file: "./prompts/shipper.md"
    timeout_minutes: 60
    max_retries: 2

  marketer:
    model: "opus-4.6"
    prompt_file: "./prompts/marketer.md"
    timeout_minutes: 45
    max_retries: 1

notifications:
  # How to notify when items need human attention
  method: "file"  # Options: "file", "email", "telegram"
  attention_file: "./attention_queue.json"
  # email:
  #   to: "woodrow227@icloud.com"
  #   smtp_server: "..."
  # telegram:
  #   bot_token: "..."
  #   chat_id: "..."

logging:
  level: "info"  # debug, info, warn, error
  file: "./logs/orchestrator.log"
  max_size_mb: 50
  rotation: "daily"
```

## Cron Logic

### What Happens Every 5 Minutes

```
┌─────────────────────────────────────────┐
│           CRON TICK (every 5 min)        │
└──────────────────┬──────────────────────┘
                   │
         ┌─────────▼──────────┐
         │ Read all state.json │
         │ files in projects/  │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │ Count active        │
         │ projects             │
         │ (not killed/live)   │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │ For each project:   │
         │ Check if phase      │
         │ complete, route     │
         │ to next agent       │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │ If slots available: │
         │ Pull from queue or  │
         │ trigger new research│
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │ Log all decisions   │
         │ to factory.db       │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │ Update attention    │
         │ queue if items need │
         │ human review        │
         └─────────────────────┘
```

### Router Decision Logic (Pseudocode)

```python
def orchestrator_tick():
    # 1. Read all project states
    projects = []
    for dir in glob("projects/*/state.json"):
        state = read_json(dir)
        if state["status"] not in ["killed", "live"]:
            projects.append(state)

    # 2. Check for running agents (don't double-spawn)
    running_sessions = get_running_sessions()

    # 3. Process each active project
    for project in projects:
        slug = project["slug"]

        # Skip if an agent is already working on this project
        if slug in running_sessions:
            continue

        # Route based on current phase
        route_project(project)

    # 4. Fill empty slots
    active_count = len([p for p in projects if p["status"] == "active"])
    if active_count < MAX_ACTIVE:
        fill_slots(MAX_ACTIVE - active_count)

    # 5. Check marketing schedule
    check_marketing_schedule()


def route_project(project):
    phase = project["phase"]
    slug = project["slug"]

    if phase == "research" and is_phase_complete(project, "research"):
        transition(slug, "validation")
        spawn_agent("scout", slug, mode="validate")

    elif phase == "validation" and is_phase_complete(project, "validation"):
        score = project["phases_completed"]["validation"]["validation_score"]
        if score >= 21:
            transition(slug, "spec")
            spawn_agent("architect", slug)
        else:
            transition(slug, "killed", reason=f"Validation score {score} < 21")

    elif phase == "spec" and is_phase_complete(project, "spec"):
        # Check if human approval is required
        if requires_human_approval("spec") and not project["phases_completed"]["spec"].get("human_approved"):
            add_to_attention_queue(slug, "Spec ready for review")
        else:
            transition(slug, "building")
            spawn_agent("builder", slug)

    elif phase == "building" and is_phase_complete(project, "build"):
        transition(slug, "reviewing")
        spawn_agent("reviewer", slug)

    elif phase == "reviewing" and is_phase_complete(project, "review"):
        review = project["review_history"][-1]
        if review["passed"]:
            transition(slug, "monetizing")
            spawn_agent("shipper", slug, mode="monetize")
        elif len(project["review_history"]) < 3:
            transition(slug, "revising")
            spawn_agent("builder", slug, mode="revise")
        else:
            transition(slug, "flagged")
            add_to_attention_queue(slug, "Failed review 3 times")

    elif phase == "revising" and is_phase_complete(project, "revision"):
        transition(slug, "reviewing")
        spawn_agent("reviewer", slug)

    elif phase == "monetizing" and is_phase_complete(project, "monetize"):
        transition(slug, "packaging")
        spawn_agent("shipper", slug, mode="package")

    elif phase == "packaging" and is_phase_complete(project, "package"):
        transition(slug, "ready_to_ship")
        add_to_attention_queue(slug, "Ready for submission approval")

    elif phase == "ready_to_ship" and project.get("human_approved_submission"):
        in_review_count = count_in_review()
        if in_review_count < 5:
            transition(slug, "in_review")
            spawn_agent("shipper", slug, mode="submit")
        # else: wait for a review slot to open

    elif phase == "in_review":
        # Check App Store Connect for review status
        check_review_status(slug)

    elif phase == "approved":
        transition(slug, "live")
        spawn_agent("marketer", slug, mode="launch")


def fill_slots(available_slots):
    # Priority: human suggestions first, then research queue
    suggestions = read_suggestions(status="queued", sort_by="priority")
    for suggestion in suggestions[:available_slots]:
        slug = create_project_from_suggestion(suggestion)
        transition(slug, "research")
        spawn_agent("scout", slug)
        available_slots -= 1

    # If still slots available, trigger new research
    if available_slots > 0:
        spawn_agent("scout", None, mode="discover")
```

## Spawning Agent Sessions

The Router spawns Claude Code sessions using OpenClaw's `sessions_spawn`:

```bash
# Example: Spawn Builder for routine-rest
openclaw sessions spawn \
  --model opus-4.6 \
  --prompt "$(cat prompts/builder.md)" \
  --context "projects/routine-rest/spec.md,projects/routine-rest/onepager.json" \
  --working-dir "projects/routine-rest/" \
  --timeout 120m \
  --on-complete "node scripts/session-complete.js routine-rest build"
```

### Session Parameters

Each agent type gets specific parameters:

| Agent | Model | Timeout | Context Files |
|-------|-------|---------|--------------|
| Scout (research) | sonnet-4.6 | 45 min | Category config, existing app catalog |
| Scout (validate) | sonnet-4.6 | 30 min | research.json, app catalog |
| Architect | opus-4.6 | 60 min | research.json, spec templates |
| Builder | opus-4.6 | 120 min | spec.md, onepager.json, templates/ |
| Builder (revise) | opus-4.6 | 90 min | spec.md, src/, quality.json |
| Reviewer | codex-5.3 | 30 min | src/, onepager.json, quality rubric |
| Shipper | sonnet-4.6 | 60 min | src/, onepager.json, Fastlane templates |
| Marketer | opus-4.6 | 45 min | onepager.json, audience analytics |

### Session Completion Handler

When a session completes, a callback script:
1. Reads the session output
2. Verifies the expected output files were created/updated
3. Updates `state.json` to mark the phase as complete
4. Logs the session to `factory.db` (duration, tokens used, cost estimate)
5. Triggers the next cron cycle to pick up the transition

## Concurrency Control

### Max Concurrent Agents: 3

Running too many agents simultaneously will:
- Overwhelm the machine (CPU, memory for Xcode builds)
- Hit API rate limits (Anthropic, OpenAI)
- Cause file contention

The Router maintains a running session count and only spawns new sessions when below the limit.

### Priority Order

When multiple projects need attention and we're at the concurrency limit:

1. **Flagged projects** (human requested action) — highest priority
2. **Projects closest to shipping** (packaging, ready_to_ship) — near the finish line
3. **Projects in review cycle** (reviewing, revising) — avoid stale reviews
4. **Projects in build** (building) — the longest phase
5. **Projects in spec** (speccing) — requires Opus, can wait
6. **Projects in research** (researching, validating) — lowest priority

## Monitoring

### Health Checks

The Router performs self-checks every cycle:

1. **Stale projects**: Any project stuck in the same phase for > 4 hours? Flag it.
2. **Failed sessions**: Any agent session that timed out or errored? Log and retry.
3. **API health**: Can we reach Anthropic, OpenAI, App Store Connect? If not, pause spawning.
4. **Disk space**: Is the machine running low? (Xcode projects can be large.)
5. **Cost tracking**: Has monthly API spend exceeded the budget? Alert.

### Cost Budget

Monthly budget tracking:

```json
// config/budget.json
{
  "monthly_budget_usd": 500,
  "current_month_spend": 187.42,
  "alert_threshold": 0.8,
  "hard_stop_threshold": 1.0,
  "per_model_costs": {
    "opus-4.6": {"input_per_1k": 0.015, "output_per_1k": 0.075},
    "sonnet-4.6": {"input_per_1k": 0.003, "output_per_1k": 0.015},
    "codex-5.3": {"input_per_1k": 0.006, "output_per_1k": 0.024}
  }
}
```

At 80% of budget: Alert in Control Panel.
At 100% of budget: Pause all non-critical agent spawning (only allow shipping and marketing for already-built apps).

## Claude Code Remote Control

For monitoring and manual intervention from your phone:

```bash
# Start a Claude Code session with Remote Control enabled
claude --remote-control

# Or enable for all sessions
claude /config
# → Set remote_control: "always"
```

This lets you:
- View the Control Panel from your phone's browser
- Approve submissions
- Kill projects
- Review one-pagers
- Monitor agent activity

All while your laptop runs the factory autonomously.

## Failsafe: Manual Override

At any point, you can:

1. **Pause the factory**: `openclaw cron stop`
2. **Resume the factory**: `openclaw cron start`
3. **Kill a project**: Edit `state.json` → `"status": "killed"`
4. **Skip a phase**: Edit `state.json` → change `"phase"` to desired phase
5. **Force re-run**: Delete the phase from `phases_completed` and the Router will re-trigger it
6. **Change models**: Edit `config/openclaw.yaml` → update model assignments
7. **Adjust concurrency**: Edit `config/openclaw.yaml` → change `max_concurrent_agents`

The factory is designed to be fully controllable through file edits and CLI commands. No magic, no hidden state.
