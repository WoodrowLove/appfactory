# Control Panel

## Overview

The Control Panel is a SvelteKit web application that serves as your window into the factory. It shows what's happening, what needs your attention, and how the business is performing. It runs locally on your machine and reads from the same state files and database that the agents write to.

## Technology

- **Framework**: SvelteKit 2 + Svelte 5
- **Styling**: TailwindCSS v4
- **Database**: SQLite (via better-sqlite3) for analytics and history
- **State**: Reads `projects/*/state.json` files directly from the filesystem
- **Real-time**: WebSocket connection to OpenClaw Gateway for live agent activity
- **Charts**: Chart.js or Svelte-specific charting library
- **Icons**: Lucide Icons

## Pages

### 1. Dashboard (Home)

The first thing you see when opening the Control Panel.

**Layout:**
```
┌────────────────────────────────────────────────────────┐
│  AppFactory Dashboard                    [Settings] [?] │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Active   │ │ Shipped  │ │ Queue    │ │ Needs    │  │
│  │ Projects │ │ Apps     │ │ Length   │ │ Attention│  │
│  │    3     │ │    7     │ │    2     │ │    1     │  │
│  │ /5 max   │ │ lifetime │ │ waiting  │ │ flagged  │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Live Agent Activity                              │   │
│  │                                                  │   │
│  │ 16:42 [Builder] routine-rest: Building views...  │   │
│  │ 16:41 [Scout] Researching productivity niche...  │   │
│  │ 16:40 [Reviewer] focus-timer: Gate 3 passed (9) │   │
│  │ 16:38 [Router] Spawned Builder for routine-rest  │   │
│  │ 16:35 [Shipper] budget-buddy: Screenshots done   │   │
│  │                                                  │   │
│  │ ◉ Live                              [View All →] │   │
│  └─────────────────────────────────────────────────┘   │
│                                                        │
│  ┌─────────────────────┐ ┌─────────────────────────┐   │
│  │ Monthly Revenue     │ │ Quality Trend           │   │
│  │                     │ │                         │   │
│  │  $2,847 MRR         │ │  Avg: 8.7/10           │   │
│  │  ▲ 23% from last    │ │  [sparkline chart]     │   │
│  │  [line chart]       │ │                         │   │
│  └─────────────────────┘ └─────────────────────────┘   │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Needs Your Attention                             │   │
│  │                                                  │   │
│  │ ⚠ focus-timer failed review 3 times     [View]  │   │
│  │ ✅ budget-buddy ready for submission    [Approve] │   │
│  │ 📋 meal-planner spec ready for review   [View]  │   │
│  │                                                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Key metrics cards:**
- **Active Projects**: Currently in the pipeline (0-5)
- **Shipped Apps**: Total apps live on the App Store
- **Queue Length**: Ideas validated but waiting for a build slot
- **Needs Attention**: Items requiring human intervention

**Live Agent Activity:**
- Real-time feed from OpenClaw Gateway
- Shows agent name, project, and current action
- Green dot indicates live connection
- Each entry is timestamped
- Clicking "View All" opens a full activity log

### 2. Projects

Shows all projects across all states.

**Layout:**
```
┌────────────────────────────────────────────────────────┐
│  Projects                    [+ Suggest App] [Filter ▾]│
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ RoutineRest                           Score: 9.0│   │
│  │ Health & Fitness · Sleep habits                  │   │
│  │                                                  │   │
│  │ ████████████████████░░░░░  PACKAGING  (7/10)    │   │
│  │                                                  │   │
│  │ Research ✓  Validate ✓  Spec ✓  Build ✓         │   │
│  │ Review ✓  Monetize ✓  Package ●  Ship ○         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ FocusTimer                            Score: 7.2│   │
│  │ Productivity · Pomodoro + deep work             │   │
│  │                                                  │   │
│  │ ██████████████░░░░░░░░░░  REVIEWING  (5/10)    │   │
│  │                                                  │   │
│  │ ⚠ Failed review 2/3 times            [View →]  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                        │
│  ┌─────────────────────────────────────────────────┐   │
│  │ BudgetBuddy                           Score: 8.4│   │
│  │ Personal Finance · Expense tracking              │   │
│  │                                                  │   │
│  │ █████████████████████████  READY TO SHIP (10/10)│   │
│  │                                                  │   │
│  │ [Approve for Submission]                         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                        │
│  Filter: Active ○ Shipped ○ Killed ○ All ●            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Project card elements:**
- App name and category
- Progress bar showing current phase (out of 10)
- Quality score (from Reviewer)
- Phase indicators (checkmark for complete, dot for current, circle for pending)
- Warning indicators for flagged items
- Quick actions (Approve, View, Kill)

### 3. One-Pager View (Project Detail)

Tapping a project card expands into the full one-pager.

**Layout:**
```
┌────────────────────────────────────────────────────────┐
│  ← Back to Projects                                    │
│                                                        │
│  RoutineRest                                           │
│  "Build the sleep routine your body needs"             │
│  Health & Fitness · Quality: 9.0/10 · Phase: Packaging │
├────────────────────────────────────────────────────────┤
│                                                        │
│  TARGET USER                                           │
│  Health-conscious adults 25-45 who want better sleep   │
│  but don't know where to start. They've tried Sleep    │
│  Cycle but found it only tracks, doesn't build habits. │
│                                                        │
│  PAIN POINTS IDENTIFIED                                │
│  • Reddit r/sleep: 342 upvotes on "apps track but      │
│    don't help build routines"                          │
│  • App Store: 142 Sleep Cycle reviews mention "no       │
│    routine builder"                                    │
│  • X: 89 tweets about wanting bedtime routine help     │
│                                                        │
│  FEATURES                                              │
│  Core (Free):                                          │
│  ✓ Routine builder with templates                      │
│  ✓ Bedtime/wake reminders                              │
│  ✓ HealthKit sleep data import                         │
│                                                        │
│  Premium ($4.99/month, 7-day trial):                   │
│  ✓ AI-powered routine personalization                  │
│  ✓ Sleep quality scoring                               │
│  ✓ Weekly insights                                     │
│                                                        │
│  DIFFERENTIATION                                       │
│  vs. Sleep Cycle: They track. We build.                │
│  vs. Our catalog: No overlap (nearest: FocusTimer,     │
│  which is productivity, not sleep)                     │
│                                                        │
│  MONETIZATION                                          │
│  Model: Freemium subscription                          │
│  Free: 3 built-in routines, basic tracking             │
│  Premium: Unlimited custom, AI, insights               │
│  Price: $4.99/month or $29.99/year                     │
│  Trial: 7 days free                                    │
│                                                        │
│  SCREENS                                               │
│  [Onboarding 1] [Onboarding 2] [Onboarding 3]         │
│  [Home] [Routine Editor] [Insights] [Settings]         │
│  [Paywall]                                             │
│                                                        │
│  QUALITY REPORT                                        │
│  Gate 1 (Crash Safety): 9/10                           │
│  Gate 2 (Permissions): 10/10                           │
│  Gate 3 (Features): 9/10                               │
│  Gate 4 (UI/UX): 9/10                                  │
│  Gate 5 (Compliance): 10/10                            │
│  Gate 6 (Code Quality): 8/10                           │
│  Gate 7 (Test Coverage): 9/10                           │
│  Gate 8 (Monetization UX): 10/10                        │
│                                                        │
│  TIMELINE                                              │
│  Research: Mar 1, 10:00 AM (24 min)                    │
│  Validate: Mar 1, 10:30 AM (15 min)                    │
│  Spec: Mar 1, 11:00 AM (35 min)                        │
│  Build: Mar 1, 12:00 PM (72 min)                       │
│  Review: Mar 1, 1:30 PM (18 min) — Passed first try   │
│  Monetize: Mar 1, 2:00 PM (12 min)                     │
│  Package: Mar 1, 2:30 PM — In progress                 │
│                                                        │
│  [Kill Project] [Restart from Phase...] [Edit Spec]    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

This is the full decision trail. Every choice the AI made, from pain point identification to feature scoping to monetization design, is visible here. If you disagree with a decision, you can edit the spec and restart from that phase.

### 4. Revenue

Financial dashboard powered by RevenueCat webhook data.

**Sections:**
- **MRR Overview**: Total MRR, growth rate, projected trajectory
- **Per-App Breakdown**: MRR, subscribers, churn, LTV for each app
- **Conversion Funnel**: Impressions → Installs → Trial Starts → Paid Conversions
- **Churn Analysis**: Monthly churn rate by app, reasons (when available)
- **Revenue by Category**: Which app categories generate the most revenue
- **Projections**: Based on current growth rate, when do we hit $5K, $10K MRR?

### 5. Marketing

Content performance dashboard.

**Sections:**
- **Content Calendar**: Upcoming scheduled posts across all accounts
- **Performance Metrics**: Views, engagement, clicks per post
- **Audience Growth**: Follower growth per account over time
- **Top Performers**: Best-performing content of the week/month
- **Attribution**: Which content led to the most app installs
- **Learnings Feed**: AI-generated insights from the self-learning loop

### 6. Suggestions

Where you submit app ideas into the pipeline.

**Layout:**
```
┌────────────────────────────────────────────────────────┐
│  Suggest an App                                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│  App idea or problem to solve:                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ A habit tracker specifically for parents who want │  │
│  │ to model good habits for their kids. Track your   │  │
│  │ habits alongside your children's habits.          │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  Category: [Health & Fitness ▾]                         │
│                                                        │
│  Priority: [Normal ▾]  (Normal / High / Rush)          │
│                                                        │
│  Notes (optional):                                     │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Saw this idea on a parenting subreddit. Multiple  │  │
│  │ parents saying existing habit apps don't account   │  │
│  │ for family dynamics.                              │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
│  [Submit to Research Queue]                            │
│                                                        │
│  ─────────────────────────────────────────────────     │
│                                                        │
│  Your Suggestions                                      │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 📋 Family Habit Tracker       Queued  (priority) │  │
│  │ 🔍 Garden Planner             Researching        │  │
│  │ ✅ Meal Prep Assistant        Building (phase 4) │  │
│  │ ❌ Meditation Timer           Killed (too many    │  │
│  │                               competitors)       │  │
│  └──────────────────────────────────────────────────┘  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Suggestion flow:**
1. You type an idea (can be rough — just a problem statement)
2. Choose a category and priority
3. Submit → creates a file in `suggestions/<timestamp>-<slug>.json`
4. Router picks it up on the next cron cycle
5. Scout researches and validates it through the normal pipeline
6. You see progress updates on the Suggestions page

**Suggestion file format:**
```json
{
  "id": "2026-03-02-family-habit-tracker",
  "submitted": "2026-03-02T16:45:00Z",
  "idea": "A habit tracker specifically for parents who want to model good habits for their kids",
  "category": "health-fitness",
  "priority": "normal",
  "notes": "Saw this on a parenting subreddit...",
  "status": "queued",
  "source": "human"
}
```

### 7. Activity Log

Full history of every action taken by every agent.

**Filterable by:**
- Agent (Router, Scout, Architect, Builder, Linter, Reviewer, Shipper, Marketer)
- Project
- Date range
- Event type (phase transition, quality score, error, human action)

### 8. Settings

- **API key management** (links to `.env` file)
- **Cron schedule** (view/modify the 5-minute interval)
- **Max concurrent projects** (default: 5)
- **Auto-approve settings** (which phases can auto-proceed vs. require human approval)
- **Notification preferences** (what triggers a "Needs Attention" alert)
- **Apple Developer account status**
- **Category priorities** (which categories to prioritize for research)

## Real-Time Updates

### WebSocket Connection

The Control Panel maintains a WebSocket connection to the OpenClaw Gateway for live updates:

```javascript
// SvelteKit: src/lib/stores/liveActivity.ts
import { writable } from 'svelte/store';

export const activityFeed = writable([]);

const ws = new WebSocket('ws://localhost:3001/activity');

ws.onmessage = (event) => {
    const entry = JSON.parse(event.data);
    activityFeed.update(feed => [entry, ...feed].slice(0, 100));
};
```

### File Watching

For state file changes, the Control Panel uses `chokidar` to watch the `projects/` directory:

```javascript
import chokidar from 'chokidar';

chokidar.watch('projects/*/state.json').on('change', (path) => {
    // Re-read the state file
    // Update the relevant project card in the UI
    // Push update via WebSocket to connected clients
});
```

## Mobile Access

The Control Panel is responsive and works on mobile browsers. When using Claude Code Remote Control from your phone, you can also access the dashboard on your laptop's browser (same machine, different device).

For push notifications (when items need attention), consider:
- Email notifications (simple, reliable)
- Telegram bot (instant, good for mobile)
- iOS shortcut that checks the API

## Design Principles

1. **Information density**: Show as much useful information as possible without clutter
2. **Action-oriented**: Every card/section has clear actions (Approve, View, Kill)
3. **Status at a glance**: Progress bars, scores, and color coding tell you the state without clicking
4. **Audit trail visible**: Every AI decision is traceable in the one-pager view
5. **Minimal clicks to act**: The most common action (approve submission) should be 2 clicks max
