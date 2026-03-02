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
  cron: "*/5 * * * *"  # Default interval (adaptive override below)
  timezone: "America/New_York"

  # Adaptive polling — overrides cron interval based on factory state
  adaptive_polling:
    active_interval_min: 1    # When agents are currently running
    default_interval_min: 5   # Fallback
    idle_interval_min: 15     # When nothing is happening

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

### Adaptive Polling

Instead of a fixed 5-minute interval, the Router uses adaptive polling to reduce wasted invocations by approximately 70%:

| Mode | Interval | Trigger |
|------|----------|---------|
| **Active** | 1 min | At least one agent session is currently running |
| **Default** | 5 min | Fallback when no specific condition applies |
| **Idle** | 15 min | No agents running, no projects in actionable phases |

The Router determines its polling mode at the end of each tick by checking running sessions and pending work. This means during a build phase (where an Opus agent runs for up to 2 hours), the Router checks every minute to catch completion quickly. When the factory is idle overnight, it only wakes every 15 minutes to scan for new suggestions or review status changes.

### What Happens Each Tick

```
┌─────────────────────────────────────────┐
│     CRON TICK (adaptive: 1/5/15 min)    │
└──────────────────┬──────────────────────┘
                   │
         ┌─────────▼──────────┐
         │ Determine polling   │
         │ mode (active/       │
         │ default/idle)       │
         └─────────┬──────────┘
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
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │ Set next cron       │
         │ interval based on   │
         │ current state       │
         └─────────────────────┘
```

## Event-Driven Transitions

Adaptive polling reduces waste, but the real latency killer is event-driven transitions. Without them, a 9-phase pipeline where each phase completes mid-interval wastes an average of 2.5 minutes per transition (half the default interval). Across 9 phases, that is up to 22 minutes of dead time per app, or 45 minutes at idle intervals.

### Agent Completion Webhook

When an agent session completes, the session completion handler POSTs to the Router's webhook instead of waiting for the next cron tick:

```
POST http://localhost:3002/agent-complete
Content-Type: application/json

{
  "slug": "routine-rest",
  "phase": "build",
  "status": "success",
  "session_id": "sess_abc123",
  "duration_minutes": 87,
  "tokens_used": 142000,
  "cost_estimate_usd": 4.12
}
```

The Router processes this immediately: it validates the phase output, updates `state.json`, and spawns the next agent in the same request cycle. The project moves from build-complete to review-started in under 5 seconds instead of waiting up to 15 minutes.

### Webhook Server

The Router runs a lightweight HTTP server alongside the cron scheduler:

```yaml
# Added to config/openclaw.yaml
orchestrator:
  webhook:
    enabled: true
    port: 3002
    path: "/agent-complete"
    auth_token: "${ROUTER_WEBHOOK_SECRET}"  # Prevents unauthorized triggers
```

### Cron as Fallback

The cron tick still runs on its adaptive schedule as a safety net. If the webhook fails (process crash, network issue), the next cron tick catches the completed phase and routes normally. This means the system is eventually consistent even if the event-driven path fails.

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

    # 6. Set adaptive polling interval for next tick
    if len(running_sessions) > 0:
        set_cron_interval(minutes=1)   # Active mode
    elif len(projects) == 0:
        set_cron_interval(minutes=15)  # Idle mode
    else:
        set_cron_interval(minutes=5)   # Default


def route_project(project):
    phase = project["phase"]
    slug = project["slug"]

    if phase == "research" and is_phase_complete(project, "research"):
        # Validate output schema before transitioning
        if not validate_schema(slug, "research"):
            handle_schema_failure(slug, "research")
            return
        write_phase_context(slug, "validation",
            reason="Research complete. Validate market viability before investing in spec.")
        transition(slug, "validation")
        spawn_agent("scout", slug, mode="validate")

    elif phase == "validation" and is_phase_complete(project, "validation"):
        if not validate_schema(slug, "validation"):
            handle_schema_failure(slug, "validation")
            return
        score = project["phases_completed"]["validation"]["validation_score"]
        if score >= 21:
            write_phase_context(slug, "spec",
                reason=f"Validation passed (score {score}/30). Create spec for Builder.")
            transition(slug, "spec")
            spawn_agent("architect", slug)
        else:
            transition(slug, "killed", reason=f"Validation score {score} < 21")

    elif phase == "spec" and is_phase_complete(project, "spec"):
        if not validate_schema(slug, "spec"):
            handle_schema_failure(slug, "spec")
            return
        # Auto-approval: specs auto-approve after 4 hours if validation score >= 24/30
        spec_completed_at = project["phases_completed"]["spec"]["completed_at"]
        validation_score = project["phases_completed"]["validation"]["validation_score"]
        hours_waiting = hours_since(spec_completed_at)

        if project["phases_completed"]["spec"].get("human_approved"):
            # Explicitly approved by human
            pass
        elif validation_score >= 24 and hours_waiting >= 4:
            # Auto-approve: high-confidence spec, waited long enough
            project["phases_completed"]["spec"]["human_approved"] = True
            project["phases_completed"]["spec"]["approved_by"] = "auto (score >= 24, timeout 4h)"
            update_state(slug, project)
        elif requires_human_approval("spec"):
            add_to_attention_queue(slug, "Spec ready for review")
            return

        write_phase_context(slug, "building",
            reason="Spec approved. Build the app exactly as specified.")
        transition(slug, "building")
        spawn_agent("builder", slug)

    elif phase == "building" and is_phase_complete(project, "build"):
        if not validate_schema(slug, "build"):
            handle_schema_failure(slug, "build")
            return
        # Gate 0: Lint before expensive Codex review
        write_phase_context(slug, "linting",
            reason="Build complete. Run compilation check + Haiku lint before Codex review.")
        transition(slug, "linting")
        spawn_agent("linter", slug)

    elif phase == "linting" and is_phase_complete(project, "lint"):
        if not validate_schema(slug, "lint"):
            handle_schema_failure(slug, "lint")
            return
        lint_report = read_json(f"projects/{slug}/lint_report.json")
        if lint_report["compilation_passed"] and lint_report["error_count"] == 0:
            # Clean lint — proceed to full Codex review
            write_phase_context(slug, "reviewing",
                reason="Lint passed (0 errors). Run full quality review.")
            transition(slug, "reviewing")
            spawn_agent("reviewer", slug)
        else:
            # Lint failed — send back to Builder with lint errors
            write_phase_context(slug, "revising",
                reason=f"Lint failed: {lint_report['error_count']} errors, "
                       f"{lint_report['warning_count']} warnings. Fix before review.")
            transition(slug, "revising")
            spawn_agent("builder", slug, mode="revise")

    elif phase == "reviewing" and is_phase_complete(project, "review"):
        if not validate_schema(slug, "review"):
            handle_schema_failure(slug, "review")
            return
        review = project["review_history"][-1]
        if review["passed"]:
            # Monetize and Package can run in parallel — they are independent
            write_phase_context(slug, "monetizing",
                reason="Review passed. Add StoreKit 2 monetization.")
            write_phase_context(slug, "packaging",
                reason="Review passed. Generate Fastlane config, screenshots, metadata.")
            transition(slug, "monetizing_and_packaging")
            spawn_agent("shipper", slug, mode="monetize")
            spawn_agent("shipper", slug, mode="package")
        elif len(project["review_history"]) < 3:
            write_phase_context(slug, "revising",
                reason=f"Review failed ({review['score']}/10). Fix issues: {review['summary']}")
            transition(slug, "revising")
            spawn_agent("builder", slug, mode="revise")
        else:
            transition(slug, "flagged")
            add_to_attention_queue(slug, "Failed review 3 times")

    elif phase == "revising" and is_phase_complete(project, "revision"):
        # After revision, go through lint gate again
        write_phase_context(slug, "linting",
            reason="Revision complete. Re-run lint gate before review.")
        transition(slug, "linting")
        spawn_agent("linter", slug)

    elif phase == "monetizing_and_packaging":
        # Both must complete before proceeding
        monetize_done = is_phase_complete(project, "monetize")
        package_done = is_phase_complete(project, "package")
        if monetize_done and package_done:
            if not (validate_schema(slug, "monetize") and validate_schema(slug, "package")):
                handle_schema_failure(slug, "monetize_or_package")
                return
            transition(slug, "ready_to_ship")
            add_to_attention_queue(slug, "Ready for submission approval")
        # else: wait for both to finish

    elif phase == "ready_to_ship" and project.get("human_approved_submission"):
        in_review_count = count_in_review()
        if in_review_count < 5:
            write_phase_context(slug, "in_review",
                reason="Submission approved. Submit to App Store Connect.")
            transition(slug, "in_review")
            spawn_agent("shipper", slug, mode="submit")
        # else: wait for a review slot to open

    elif phase == "in_review":
        # Check App Store Connect for review status
        check_review_status(slug)

    elif phase == "approved":
        write_phase_context(slug, "live",
            reason="App approved by Apple. Launch marketing campaign.")
        transition(slug, "live")
        spawn_agent("marketer", slug, mode="launch")


def write_phase_context(slug, next_phase, reason):
    """Write phase_context to state.json so the spawned agent knows why it was invoked."""
    state = read_json(f"projects/{slug}/state.json")
    state["phase_context"] = {
        "next_phase": next_phase,
        "reason": reason,
        "routed_at": now_iso(),
        "routed_by": "orchestrator"
    }
    write_json(f"projects/{slug}/state.json", state)


def handle_schema_failure(slug, phase):
    """Handle JSON schema validation failure for agent output."""
    project = read_json(f"projects/{slug}/state.json")
    retries = project.get("schema_retries", {}).get(phase, 0)
    max_retries = get_agent_config(phase)["max_retries"]

    if retries < max_retries:
        project.setdefault("schema_retries", {})[phase] = retries + 1
        update_state(slug, project)
        errors = get_last_validation_errors(slug, phase)
        write_phase_context(slug, phase,
            reason=f"Schema validation failed. Errors: {errors}. Retry {retries+1}/{max_retries}.")
        spawn_agent(phase_to_agent(phase), slug, mode="retry")
    else:
        transition(slug, "flagged")
        add_to_attention_queue(slug,
            f"Schema validation failed after {max_retries} retries in {phase}")


def fill_slots(available_slots):
    # Priority: human suggestions first, then research queue
    suggestions = read_suggestions(status="queued", sort_by="priority")
    for suggestion in suggestions[:available_slots]:
        slug = create_project_from_suggestion(suggestion)
        write_phase_context(slug, "research",
            reason=f"New project from suggestion: {suggestion['title']}")
        transition(slug, "research")
        spawn_agent("scout", slug)
        available_slots -= 1

    # If still slots available, trigger new research
    if available_slots > 0:
        spawn_agent("scout", None, mode="discover")
```

## Spawning Agent Sessions

### Budget Pre-Check

Before spawning ANY agent, the Router performs a budget pre-check against `config/budget.json`:

```python
def budget_pre_check(agent_type, slug):
    budget = read_json("config/budget.json")
    estimated_cost = estimate_session_cost(agent_type)

    if budget["current_month_spend"] + estimated_cost >= budget["monthly_budget_usd"]:
        # At 100% — only critical agents proceed
        if agent_type in ["shipper", "marketer"]:
            # Critical: these finish already-built apps to avoid wasting sunk costs
            log.warn(f"Budget exceeded but spawning critical agent {agent_type} for {slug}")
            return True
        else:
            # Non-critical: scout, architect, builder, reviewer — pause
            log.error(f"Budget exceeded. Blocking {agent_type} for {slug}")
            add_to_attention_queue(slug, "Monthly budget exceeded — agent paused")
            return False

    elif budget["current_month_spend"] + estimated_cost >= budget["monthly_budget_usd"] * budget["alert_threshold"]:
        # At 80% — alert but proceed
        log.warn(f"Budget at {pct}% — approaching limit")
        notify("budget_warning", f"Spend at {budget['current_month_spend']:.2f} / {budget['monthly_budget_usd']:.2f}")

    return True
```

The logic preserves sunk costs: if an app has already been built (costing ~$8-15 in Opus tokens), it would be wasteful to block the $2 Shipper and Marketer sessions that turn it into revenue. Non-critical agents (Scout, Architect, Builder, Reviewer) are paused because they represent new spend on apps that have not yet generated any value.

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
5. POSTs to `http://localhost:3002/agent-complete` to trigger immediate routing (see Event-Driven Transitions above)
6. Falls back to the next cron tick if the webhook is unreachable

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

## Schema Validation

Before any phase transition, the Router validates the agent's output against the appropriate JSON schema from `schemas/`. This prevents malformed or incomplete data from cascading through the pipeline and corrupting downstream phases.

### Validation Flow

```
Agent completes phase
        │
        ▼
┌───────────────────┐
│ Load schema from   │
│ schemas/{phase}.json│
└────────┬──────────┘
         │
         ▼
┌───────────────────┐     ┌──────────────────────┐
│ Validate agent     │────▶│ PASS: Update          │
│ output against     │     │ state.json, proceed   │
│ schema (ajv)       │     │ to next phase         │
└────────┬──────────┘     └──────────────────────┘
         │ FAIL
         ▼
┌───────────────────┐
│ Retry count < max? │
│ Yes: Re-spawn agent│
│ with error context │
│ No: Flag for human │
│ review              │
└───────────────────┘
```

### Schema Mapping

| Phase | Schema File | Validates |
|-------|------------|-----------|
| research | `schemas/research.json` | research.json output |
| validation | `schemas/validation.json` | validation scores and reasoning |
| spec | `schemas/spec.json` | spec structure and completeness |
| build | `schemas/build.json` | build manifest, file list, compilation status |
| lint | `schemas/lint_report.json` | lint results, error/warning counts |
| review | `schemas/quality.json` | quality.json scores and verdicts |
| monetize | `schemas/monetization.json` | StoreKit config, pricing |
| package | `schemas/package.json` | Fastlane config, screenshots, metadata |

### On Validation Failure

When an agent produces malformed JSON:

1. The Router logs the validation errors (missing fields, wrong types, extra properties)
2. If the agent has retries remaining, it re-spawns the agent with the validation errors included in the prompt context so the agent can self-correct
3. If retries are exhausted, the project is flagged for human review with the raw output and validation errors attached to the attention queue entry

This is especially important for the Builder and Architect phases, where a missing field in `spec.md` or `onepager.json` can cause the Builder to produce incomplete apps or the Reviewer to score against the wrong criteria.

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
