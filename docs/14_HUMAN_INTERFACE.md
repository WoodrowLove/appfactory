# Human Interface

## Philosophy

The factory runs autonomously, but you maintain strategic control. The system handles execution; you handle judgment. Every critical decision point has a human gate, and you can intervene at any time.

## Your Roles

### 1. Strategic Director
- Decide which categories to focus on
- Suggest specific app ideas
- Set revenue targets and quality thresholds
- Choose when to scale up or slow down

### 2. Quality Gatekeeper
- Review one-pagers before build (optional, can auto-approve)
- Review flagged projects (3 review failures)
- Approve apps for App Store submission
- Handle Apple rejections that require judgment

### 3. Final Approver
- Press "Submit for Review" in App Store Connect (or approve via Control Panel)
- Approve batch submissions (max 5 at a time)
- Decide whether to appeal rejections or kill projects

## Interaction Methods

### 1. Control Panel (Primary)

The SvelteKit dashboard is your main interface. See [Control Panel](10_CONTROL_PANEL.md) for full spec.

**Daily workflow:**
1. Open the Control Panel (http://localhost:5173)
2. Check "Needs Attention" section on Dashboard
3. Review any items requiring approval
4. Check Revenue tab for performance data
5. Optionally suggest new app ideas

### 2. Claude Code Remote Control (Mobile)

When you're away from your laptop:

```bash
# Start Remote Control
claude --remote-control
# Scan QR code with Claude iOS app
```

From your phone, you can:
- View project status
- Approve submissions
- Read one-pagers
- Kill projects
- Monitor agent activity

Your laptop continues running the factory in the background.

### 3. File-Based Interaction

For power users or when the Control Panel isn't running:

**Suggest an app:**
```bash
# Create a suggestion file
cat > suggestions/$(date +%Y-%m-%d)-my-idea.json << 'EOF'
{
  "id": "2026-03-02-my-idea",
  "submitted": "2026-03-02T12:00:00Z",
  "idea": "A pomodoro timer that syncs with your calendar",
  "category": "productivity",
  "priority": "high",
  "notes": "Saw this need repeatedly in r/productivity",
  "status": "queued",
  "source": "human"
}
EOF
```

**Approve a submission:**
```bash
# Edit the project's state.json
jq '.human_approved_submission = true' projects/routine-rest/state.json > tmp.json
mv tmp.json projects/routine-rest/state.json
```

**Kill a project:**
```bash
jq '.status = "killed" | .phase = "killed" | .flags += ["killed_by_human"]' \
  projects/bad-idea/state.json > tmp.json
mv tmp.json projects/bad-idea/state.json
```

**Pause the factory:**
```bash
openclaw cron stop
```

**Resume the factory:**
```bash
openclaw cron start --config config/openclaw.yaml
```

## Approval Workflows

### Spec Approval (Optional)

By default, specs auto-approve after the Architect completes them. You can require manual approval:

```yaml
# config/openclaw.yaml
approval_gates:
  spec: "manual"      # Requires human approval
  # spec: "auto"      # Auto-approve (default)
  submission: "manual" # Always requires human approval (cannot be changed)
```

When spec approval is manual:
1. Architect completes the spec
2. Control Panel shows "Spec ready for review" in Needs Attention
3. You open the one-pager, read it, and either:
   - **Approve**: The project proceeds to build
   - **Edit**: You modify the spec and approve the edited version
   - **Kill**: You reject the idea with a reason

### Submission Approval (Always Manual)

This gate is always manual. You cannot auto-approve App Store submissions.

1. Shipper completes packaging
2. Control Panel shows "Ready for submission" with the full asset package
3. You review: icon, screenshots, listing copy, promo video, quality score
4. You approve → Shipper submits to App Store Connect
5. You optionally batch-approve multiple apps at once

### Flagged Project Review

When a project fails review 3 times:

1. Control Panel shows the project as "Flagged"
2. You see all 3 quality reports side by side
3. You decide:
   - **Fix it**: Edit the code yourself or add specific instructions for the Builder
   - **Restart from spec**: Tell the Architect to reduce scope or change approach
   - **Kill**: The idea isn't working. Free up the slot.

## Suggesting App Ideas

### From the Control Panel

Navigate to the **Suggestions** page and fill in:
- **App idea**: Describe the problem to solve (can be rough)
- **Category**: Pick from the dropdown
- **Priority**: Normal, High, or Rush
- **Notes**: Any context (source URL, your reasoning, target audience)

### From a File

Drop a JSON file in `suggestions/`:

```json
{
  "id": "2026-03-02-garden-planner",
  "submitted": "2026-03-02T09:00:00Z",
  "idea": "An app that tells you exactly when to plant, water, and harvest based on your location and the specific plants in your garden",
  "category": "lifestyle",
  "priority": "high",
  "notes": "r/gardening has 5M members. No good iOS app for this. Most are web-based with terrible UX. Could use location + weather API + plant database.",
  "status": "queued",
  "source": "human"
}
```

### Priority Levels

| Priority | Behavior |
|----------|----------|
| **Normal** | Enters the queue behind existing ideas. Processed in FIFO order. |
| **High** | Jumps to the front of the queue. Next available slot goes to this idea. |
| **Rush** | Immediately starts research, even if it means queuing another project. Use sparingly. |

## Notification System

### What Triggers a Notification

| Event | Notification | Urgency |
|-------|-------------|---------|
| Spec ready for review (if manual approval) | "Review one-pager for [app]" | Medium |
| App ready for submission | "Approve [app] for App Store" | High |
| Project flagged (3 review failures) | "Review flagged project [app]" | High |
| Apple rejection received | "Apple rejected [app]: [reason]" | High |
| App approved by Apple | "Congratulations! [app] is live" | Info |
| Revenue milestone | "[app] hit $X MRR" | Info |
| Budget alert (80% spent) | "Monthly API budget at 80%" | Medium |
| Agent error | "[agent] failed on [project]" | Medium |

### Notification Channels

Configure in `config/openclaw.yaml`:

1. **Control Panel** (always): Notifications appear in the "Needs Attention" section
2. **Email** (optional): Send to your email for high-urgency items
3. **Telegram** (optional): Instant push notifications on your phone
4. **File** (default): Written to `attention_queue.json` for other tools to read

## Batch Operations

### Batch Submission

When multiple apps are ready to ship:

1. Navigate to Projects → filter by "Ready to Ship"
2. Select up to 5 apps (Apple's concurrent review limit)
3. Review each briefly (quality score, listing preview)
4. Press "Submit Batch"
5. Shipper submits all selected apps to App Store Connect

### Batch Kill

For housekeeping:

1. Navigate to Projects → filter by "Flagged" or "Killed"
2. Select multiple projects
3. Press "Archive" to move them out of the active view
4. Projects and their files remain on disk for reference but don't consume pipeline slots

## Emergency Controls

### Pause Everything
```bash
openclaw cron stop
```
All agent spawning stops. Projects in progress will complete their current session but no new sessions will start.

### Kill a Running Session
```bash
openclaw sessions kill <session-id>
```
Force-stops a running agent. The project returns to its pre-session state.

### Reset a Project
```bash
# Reset to a specific phase
jq '.phase = "spec" | del(.phases_completed.build, .phases_completed.review) | .review_history = []' \
  projects/routine-rest/state.json > tmp.json
mv tmp.json projects/routine-rest/state.json
```

### Full Factory Reset
```bash
# WARNING: This kills all projects and clears all state
openclaw cron stop
rm -rf projects/*/
rm factory.db
# Restart
openclaw cron start --config config/openclaw.yaml
```

## Time Commitment

Expected time commitment per week:

| Activity | Time | Frequency |
|----------|------|-----------|
| Review one-pagers | 10-15 min | 2-3 per week |
| Approve submissions | 5-10 min | 1-2 per week |
| Review flagged projects | 15-20 min | 0-1 per week |
| Check revenue dashboard | 5 min | Daily |
| Suggest new ideas | 10-15 min | Weekly |

**Total: 1-2 hours per week** for a fully running factory.

The entire point of this system is that your time goes to judgment calls, not execution. The AI handles the work. You handle the decisions.
