# App Suggestions

Drop JSON files here to suggest app ideas for the factory to research and build.

## File Format

```json
{
  "id": "YYYY-MM-DD-short-slug",
  "submitted": "ISO-8601 timestamp",
  "idea": "Describe the problem to solve or the app you want built",
  "category": "health-fitness | productivity | personal-finance | lifestyle | education | utilities",
  "priority": "normal | high | rush",
  "notes": "Any context: source URLs, your reasoning, target audience",
  "status": "queued",
  "source": "human"
}
```

## Priority Levels

- **normal**: Enters the queue in FIFO order
- **high**: Jumps to the front of the queue
- **rush**: Starts immediately, even if it means queuing another project

## What Happens Next

1. The Router picks up your suggestion on its next 5-minute cron cycle
2. The Scout researches and validates the idea
3. If it scores >= 21/30, it enters the build pipeline
4. You can track progress in the Control Panel (Suggestions page)

## Example

```json
{
  "id": "2026-03-02-garden-planner",
  "submitted": "2026-03-02T09:00:00Z",
  "idea": "An app that tells you exactly when to plant, water, and harvest based on your location and the specific plants in your garden",
  "category": "lifestyle",
  "priority": "high",
  "notes": "r/gardening has 5M members. No good iOS app for this.",
  "status": "queued",
  "source": "human"
}
```
