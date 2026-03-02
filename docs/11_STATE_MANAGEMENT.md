# State Management

## Core Principle

**Never rely on conversation history for important state. Always write it to files.**

Every agent starts fresh. Every agent reads its inputs from files. Every agent writes its outputs to files. The state files are the source of truth, not any agent's memory.

## State File: `state.json`

Every project has a `state.json` that tracks its lifecycle:

```json
{
  "slug": "routine-rest",
  "app_name": "RoutineRest",
  "category": "health-fitness",
  "phase": "packaging",
  "status": "active",
  "created_at": "2026-03-01T10:00:00Z",
  "updated_at": "2026-03-02T14:30:00Z",
  "source": "research",

  "phases_completed": {
    "research": {
      "completed_at": "2026-03-01T10:24:00Z",
      "duration_seconds": 1440,
      "agent_session_id": "sess_abc123"
    },
    "validation": {
      "completed_at": "2026-03-01T10:39:00Z",
      "duration_seconds": 900,
      "validation_score": 24,
      "scores": {"demand": 8, "competition": 7, "feasibility": 9}
    },
    "spec": {
      "completed_at": "2026-03-01T11:14:00Z",
      "duration_seconds": 2100,
      "human_approved": true,
      "approved_at": "2026-03-01T11:20:00Z"
    },
    "build": {
      "completed_at": "2026-03-01T12:26:00Z",
      "duration_seconds": 4320,
      "compilation_success": true
    },
    "review": {
      "completed_at": "2026-03-01T12:44:00Z",
      "duration_seconds": 1080,
      "attempt": 1,
      "quality_score": 9.0,
      "passed": true
    },
    "monetize": {
      "completed_at": "2026-03-01T12:56:00Z",
      "duration_seconds": 720,
      "product_ids": [
        "com.appfactory.routinerest.premium.monthly",
        "com.appfactory.routinerest.premium.yearly"
      ]
    }
  },

  "current_phase_started_at": "2026-03-01T13:00:00Z",

  "review_history": [
    {
      "attempt": 1,
      "score": 9.0,
      "passed": true,
      "timestamp": "2026-03-01T12:44:00Z"
    }
  ],

  "submission": {
    "queued_at": null,
    "submitted_at": null,
    "review_result": null,
    "live_at": null,
    "rejection_count": 0,
    "rejection_reasons": []
  },

  "revenue": {
    "mrr": 0,
    "total_revenue": 0,
    "subscribers": 0,
    "trial_starts": 0,
    "last_updated": null
  },

  "flags": [],
  "notes": []
}
```

## State Transitions

Every phase transition updates `state.json`:

```
phase: "research" → "validation" → "spec" → "building" → "reviewing" →
       "monetizing" → "packaging" → "ready_to_ship" → "in_review" →
       "approved" → "live"

Special states:
  "revising"    — Failed review, Builder is fixing
  "flagged"     — 3 review failures or other issue requiring human attention
  "killed"      — Idea killed (failed validation or human decision)
  "rejected"    — Apple rejected (under remediation)
```

## Project Files

Each project produces files at different phases:

```
projects/<slug>/
├── state.json              # Lifecycle state (written by every agent)
├── research.json           # Scout output (research data)
├── spec.md                 # Architect output (human-readable one-pager)
├── onepager.json           # Architect output (machine-readable spec)
├── src/                    # Builder output (complete Xcode project)
│   └── ...
├── quality.json            # Reviewer output (quality report)
├── quality_history/        # All review attempts
│   ├── attempt_1.json
│   ├── attempt_2.json
│   └── attempt_3.json
├── listing.json            # Shipper output (App Store metadata)
├── assets/
│   ├── icon.png            # Shipper output (generated icon)
│   ├── screenshots/        # Shipper output (device screenshots)
│   │   ├── iPhone_16_Pro_Max/
│   │   ├── iPhone_16/
│   │   └── iPad_Pro_13/
│   ├── framed/             # Shipper output (framed screenshots)
│   └── promo.mp4           # Shipper output (promo video)
├── marketing/
│   ├── launch_content/     # Marketer output (launch posts)
│   └── ongoing/            # Marketer output (scheduled content)
└── logs/
    └── agent_sessions.log  # Session IDs and timestamps for audit
```

## SQLite Database: `factory.db`

For analytics, history, and cross-project queries that aren't practical with individual JSON files.

### Schema

```sql
-- Project lifecycle events
CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_slug TEXT NOT NULL,
    event_type TEXT NOT NULL,       -- 'phase_transition', 'quality_score', 'submission', 'error', 'human_action'
    from_phase TEXT,
    to_phase TEXT,
    data TEXT,                      -- JSON payload
    agent TEXT,                     -- Which agent triggered this
    session_id TEXT,
    timestamp TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Quality scores over time
CREATE TABLE quality_scores (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_slug TEXT NOT NULL,
    attempt INTEGER NOT NULL,
    gate_1 REAL, gate_2 REAL, gate_3 REAL,
    gate_4 REAL, gate_5 REAL, gate_6 REAL,
    overall REAL NOT NULL,
    passed BOOLEAN NOT NULL,
    timestamp TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Revenue data (from RevenueCat webhooks)
CREATE TABLE revenue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_slug TEXT NOT NULL,
    event_type TEXT NOT NULL,       -- 'trial_start', 'conversion', 'renewal', 'cancellation', 'refund'
    amount REAL,
    currency TEXT DEFAULT 'USD',
    subscriber_id TEXT,
    timestamp TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Research ideas (validated and killed)
CREATE TABLE ideas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    slug TEXT NOT NULL UNIQUE,
    category TEXT NOT NULL,
    demand_score INTEGER,
    competition_score INTEGER,
    feasibility_score INTEGER,
    total_score INTEGER,
    status TEXT NOT NULL,           -- 'queued', 'researching', 'validated', 'killed', 'in_pipeline'
    source TEXT,                    -- 'research', 'suggestion'
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    killed_reason TEXT
);

-- Marketing content performance
CREATE TABLE content_performance (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    category TEXT NOT NULL,
    platform TEXT NOT NULL,         -- 'tiktok', 'instagram', 'youtube'
    post_id TEXT,
    content_type TEXT,              -- 'value', 'app_mention', 'launch'
    views INTEGER DEFAULT 0,
    likes INTEGER DEFAULT 0,
    comments INTEGER DEFAULT 0,
    shares INTEGER DEFAULT 0,
    link_clicks INTEGER DEFAULT 0,
    posted_at TEXT NOT NULL,
    metrics_updated_at TEXT
);

-- Submission queue
CREATE TABLE submission_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_slug TEXT NOT NULL,
    queued_at TEXT NOT NULL DEFAULT (datetime('now')),
    submitted_at TEXT,
    review_result TEXT,             -- 'approved', 'rejected', null
    result_at TEXT,
    priority INTEGER DEFAULT 0
);

-- Agent session log
CREATE TABLE agent_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,
    agent_type TEXT NOT NULL,
    project_slug TEXT,
    model TEXT NOT NULL,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    tokens_used INTEGER,
    cost_estimate REAL,
    status TEXT DEFAULT 'running'   -- 'running', 'completed', 'failed'
);
```

### Indexes

```sql
CREATE INDEX idx_events_project ON events(project_slug);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_quality_project ON quality_scores(project_slug);
CREATE INDEX idx_revenue_project ON revenue(project_slug);
CREATE INDEX idx_revenue_type ON revenue(event_type);
CREATE INDEX idx_ideas_status ON ideas(status);
CREATE INDEX idx_content_category ON content_performance(category);
CREATE INDEX idx_sessions_project ON agent_sessions(project_slug);
```

## Suggestion Files

User suggestions live in `suggestions/`:

```json
// suggestions/2026-03-02-family-habit-tracker.json
{
  "id": "2026-03-02-family-habit-tracker",
  "submitted": "2026-03-02T16:45:00Z",
  "idea": "A habit tracker for parents to model good habits alongside their kids",
  "category": "health-fitness",
  "priority": "normal",
  "notes": "Saw on r/parenting. Multiple parents wanting this.",
  "status": "queued",
  "source": "human",
  "picked_up_at": null,
  "project_slug": null
}
```

When the Router picks up a suggestion:
1. Creates a new project directory `projects/<slug>/`
2. Copies relevant suggestion data into the initial `state.json`
3. Updates the suggestion file: `status: "in_pipeline"`, `project_slug: "<slug>"`
4. Spawns the Scout in research mode

## File Locking

Since the Router and agents may write to the same files, we use simple file locking:

```javascript
// OpenClaw state manager
import { lockSync, unlockSync } from 'proper-lockfile';

function updateState(slug, updates) {
    const statePath = `projects/${slug}/state.json`;
    const release = lockSync(statePath);
    try {
        const state = JSON.parse(fs.readFileSync(statePath, 'utf8'));
        const updated = { ...state, ...updates, updated_at: new Date().toISOString() };
        fs.writeFileSync(statePath, JSON.stringify(updated, null, 2));
    } finally {
        release();
    }
}
```

## Backup Strategy

State files are critical. Losing them means losing project history.

1. **Git**: All state files are committed to the repo after every phase transition
2. **SQLite WAL mode**: Database uses Write-Ahead Logging for crash recovery
3. **ICP audit**: Critical events (quality scores, submissions) are also logged to the ICP canister as a tamper-proof backup
4. **Daily backup**: A cron job copies `factory.db` and all `state.json` files to a backup directory

## State Queries

Common queries the Control Panel makes:

```sql
-- Active projects
SELECT * FROM ideas WHERE status = 'in_pipeline'
  UNION
SELECT slug, category FROM (
  SELECT json_extract(readfile('projects/' || slug || '/state.json'), '$.phase') as phase
  FROM ideas WHERE status = 'in_pipeline'
);

-- Monthly revenue
SELECT SUM(amount) as mrr
FROM revenue
WHERE event_type IN ('conversion', 'renewal')
  AND timestamp > date('now', '-30 days');

-- Average quality score trend
SELECT date(timestamp) as day, AVG(overall) as avg_score
FROM quality_scores
GROUP BY date(timestamp)
ORDER BY day DESC
LIMIT 30;

-- Top performing content
SELECT * FROM content_performance
WHERE posted_at > date('now', '-7 days')
ORDER BY views DESC
LIMIT 10;

-- Agent compute costs
SELECT agent_type, SUM(cost_estimate) as total_cost, COUNT(*) as sessions
FROM agent_sessions
WHERE started_at > date('now', '-30 days')
GROUP BY agent_type;
```
