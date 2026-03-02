# Marketing Engine

## Philosophy

The best marketing doesn't feel like marketing. Each category-specific social account builds an audience by providing genuine value — sleep tips, productivity hacks, finance strategies — and then naturally introduces relevant apps when they ship.

This is not a bot farm. This is a content operation that happens to be AI-powered.

## Channel Strategy

### One Account Per Category

| Category | TikTok Handle | Instagram Handle | Content Theme |
|----------|--------------|-----------------|---------------|
| Health & Fitness | @[brand]_wellness | @[brand]_wellness | Sleep routines, workout tips, nutrition, mindfulness |
| Productivity | @[brand]_focus | @[brand]_focus | Focus techniques, time management, habit building |
| Personal Finance | @[brand]_money | @[brand]_money | Budgeting tips, savings challenges, money mindset |
| Lifestyle | @[brand]_home | @[brand]_home | Home organization, cooking hacks, gardening, DIY |
| Education | @[brand]_learn | @[brand]_learn | Study techniques, learning hacks, memory tips |

**Why one account per category:**
- Builds a focused, engaged audience in each niche
- Algorithm favors consistent content themes
- When an app ships, the audience is already primed for that category
- Reduces platform ban risk (one account doing many things looks like a bot)

### Platform Priority

1. **TikTok**: Highest organic reach, best for discovery. Short-form video (15-60 seconds).
2. **Instagram Reels**: Cross-post TikTok content. Strong for carousel posts and stories.
3. **YouTube Shorts**: Cross-post short-form video. Growing discovery platform.

TikTok is the primary platform. Instagram and YouTube Shorts receive adapted versions of the same content.

## Content Types

### 1. Value Content (80% of posts)

Pure value, no app mention. Builds trust and audience.

**Formats:**
- **Hook + Tip**: "Most people make this mistake with their sleep schedule..." → practical tip
- **List**: "3 productivity systems that actually work" → carousel or slideshow
- **Before/After**: "My morning routine before vs. after [technique]"
- **Myth-busting**: "You don't actually need 8 hours of sleep. Here's why..."
- **Tool comparison**: "I tested 5 meditation techniques for 30 days"

**Content is generated from:**
- Research data from the Scout (Reddit pain points make great content)
- Category expertise (the Marketer is prompted with deep knowledge of each category)
- Trending topics (what's currently performing on TikTok in the category)

### 2. App Mentions (15% of posts)

Natural mentions of AppFactory apps within value content.

**Rules:**
- Never make the app the focus. The value/tip is the focus.
- Mention the app as a tool that helps, not the point of the post
- "I've been using [app] to track this" is good
- "DOWNLOAD [APP] NOW" is bad
- Include a subtle CTA in caption, not in the video
- Link in bio, not in every post

**Example script:**
```
[Hook]: "I couldn't stick to a bedtime routine until I tried this framework"
[Value]: Explain the 3-step wind-down method (genuine advice)
[Mention]: "I built the actual routine in RoutineRest - it sends me reminders and tracks my consistency"
[CTA]: "Link in bio if you want to try it"
```

### 3. Launch Content (5% of posts)

Dedicated app launch posts when a new app ships.

**Formats:**
- **Problem → Solution**: "Here's the app I wish existed when I was struggling with [problem]"
- **Demo**: 30-second screen recording showing the app in action
- **Testimonial-style**: "I've been beta testing this for 2 weeks and here's what changed"

Launch content gets a burst push: post on all platforms within 24 hours of going live.

## Content Generation Pipeline

### Larry Skill (OpenClaw)

The Larry skill handles the autonomous content loop:

1. **Generate**: Create content ideas based on category themes and trending topics
2. **Create visuals**: Use gpt-image-1.5 for thumbnails and static images
3. **Write hooks**: Generate 3 hook variants per piece of content
4. **Post**: Schedule via Postiz API
5. **Analyze**: After 48 hours, check engagement metrics
6. **Learn**: Adjust content strategy based on what performed well
7. **Iterate**: Next content cycle incorporates learnings

### Content Schedule

| Day | Content Type | Platform |
|-----|-------------|----------|
| Monday | Value tip (carousel) | Instagram → TikTok |
| Tuesday | Hook + tip (video) | TikTok → Instagram → YouTube |
| Wednesday | List/comparison | TikTok → Instagram |
| Thursday | Value tip (video) | TikTok → Instagram → YouTube |
| Friday | Before/after or myth-bust | TikTok → Instagram |
| Saturday | App mention (if relevant) | TikTok → Instagram |
| Sunday | Rest (or repurpose top performer) | Instagram story only |

**Posting frequency**: 5-6 posts per week per account. Enough to stay visible without burning out the audience.

**Posting times** (optimized per platform):
- TikTok: 7 PM - 9 PM ET (peak usage)
- Instagram: 11 AM - 1 PM ET (lunch break scrolling)
- YouTube Shorts: 2 PM - 4 PM ET

These are defaults. The Larry skill adjusts based on actual engagement data.

## Promo Video Generation

### Tool: Remotion

For each app launch, generate a promo video:

**Structure (30 seconds):**
1. **Hook** (0-3s): Text overlay stating the problem. Dark background, white text, dramatic.
2. **Demo** (3-20s): Screen recording of the app solving the problem. Smooth scrolls, taps highlighted.
3. **Features** (20-25s): 3 bullet points animated in, each with a screenshot.
4. **CTA** (25-30s): App icon, name, "Download Free" text, App Store badge.

**Production:**
```javascript
// Remotion composition structure
const PromoVideo = () => {
  return (
    <Composition
      id="AppPromo"
      component={AppPromo}
      durationInFrames={30 * 30} // 30 seconds at 30fps
      fps={30}
      width={1080}
      height={1920} // 9:16 vertical
    />
  );
};
```

### TikTok-Style Hook Videos

For ongoing marketing, shorter 15-second hook videos:

1. **Text hook** (0-2s): Bold text stating a surprising fact or question
2. **Quick demo** (2-12s): Fast-paced screen recording or slideshow
3. **CTA** (12-15s): "Link in bio" + app name

These are simpler to produce and can be generated in batch.

## Social Media Posting

### Tool: Postiz

Postiz is an open-source social media scheduling platform that supports TikTok, Instagram, YouTube, and more via official APIs.

**Integration:**
- Self-hosted Postiz instance (or cloud)
- API-based posting (no browser automation, no ToS violations)
- Supports: text, images, video, carousels
- Built-in analytics dashboard

**API flow:**
```
Marketer generates content
  → Saves to projects/<app-slug>/marketing/content/<date>/
  → Calls Postiz API to schedule post
  → Postiz publishes at scheduled time
  → Postiz webhook reports engagement metrics
  → Marketer reads metrics and adjusts strategy
```

### Platform API Compliance

| Platform | API Used | Compliance |
|----------|----------|------------|
| TikTok | TikTok Content Posting API | Authorized app, no automation scripts |
| Instagram | Instagram Graph API (via Meta) | Authorized business account |
| YouTube | YouTube Data API v3 | Authorized OAuth2 credentials |

**What we do NOT do:**
- No browser automation to post (violates ToS)
- No fake engagement (no buying likes/follows)
- No multiple accounts on the same platform for the same category
- No follow/unfollow schemes
- No comment bots or auto-DMs

## Analytics & Self-Learning

### Metrics Tracked Per Post

| Metric | Source | Purpose |
|--------|--------|---------|
| Views | Platform API | Reach measurement |
| Likes | Platform API | Content quality signal |
| Comments | Platform API | Engagement depth |
| Shares/Saves | Platform API | Viral potential signal |
| Profile visits | Platform API | Interest conversion |
| Link clicks | UTM tracking | App install intent |
| App installs | App Store Connect | Attribution |

### Self-Learning Loop

The Marketer maintains a learning file per category:

```json
// marketing/health-fitness/learnings.json
{
  "top_performing_hooks": [
    {"hook": "Most people get sleep wrong because...", "avg_views": 45000},
    {"hook": "I stopped doing this before bed and...", "avg_views": 38000}
  ],
  "underperforming_angles": [
    {"angle": "Scientific study citations", "avg_views": 2000}
  ],
  "best_posting_times": {
    "tiktok": "7:30 PM ET",
    "instagram": "12:15 PM ET"
  },
  "content_format_ranking": [
    {"format": "hook_tip_video", "avg_engagement_rate": 0.08},
    {"format": "list_carousel", "avg_engagement_rate": 0.06},
    {"format": "before_after", "avg_engagement_rate": 0.05}
  ],
  "audience_demographics": {
    "primary_age": "25-34",
    "primary_gender": "female",
    "top_locations": ["US", "UK", "CA"]
  }
}
```

Each content cycle, the Marketer reads this file and adjusts:
- Hook styles that work → generate more like them
- Formats that perform → prioritize them
- Posting times that get more views → shift schedule
- Angles that fail → avoid them

## Cross-Promotion

When an AppFactory app serves a user who might benefit from another AppFactory app:

**Example:** A user of the Sleep Routine app might also want the Focus Timer app. The Marketer creates content bridging the two:

"Your morning focus is 3x better when your sleep routine is dialed in. Here's the combo I use..."

Cross-promotion rules:
- Must be natural and provide genuine value
- Never more than one cross-promotion per week per account
- Only promote apps that genuinely complement each other
- Track cross-promotion conversion separately

## Budget

### Content Production Costs

| Item | Monthly Cost | Notes |
|------|-------------|-------|
| Postiz (self-hosted) | $0-20 | Open source, hosting costs |
| gpt-image-1.5 API | $50-100 | ~200-400 images per month across all accounts |
| Remotion | Free (open source) | Self-hosted rendering |
| Opus 4.6 for content writing | Included in Anthropic API | Part of existing API usage |
| TikTok API | Free | Content posting API |
| Instagram Graph API | Free | Business account required |
| YouTube Data API | Free | Quota-limited |

**Total marketing infrastructure cost: $50-$120/month**

### Growth Targets

| Month | Followers Per Account | Monthly Reach | App Installs (est.) |
|-------|----------------------|---------------|---------------------|
| 1 | 100-500 | 5,000-20,000 | 50-200 |
| 3 | 1,000-3,000 | 50,000-150,000 | 500-1,500 |
| 6 | 5,000-15,000 | 200,000-600,000 | 2,000-6,000 |
| 12 | 15,000-50,000 | 500,000-1,500,000 | 5,000-15,000 |

These are conservative estimates. A single viral video (100K+ views) can accelerate these timelines dramatically. The self-learning loop optimizes for this virality over time.

## Engagement Auto-Reply

When a post receives significant engagement, responding to comments boosts algorithmic reach. Most platforms reward posts where the creator actively participates in the conversation.

**Trigger:** When a post accumulates **10+ comments**, the Marketer queues a response task.

**Process:**
1. The Larry skill monitors comment counts via the Postiz analytics webhook
2. When a post crosses the 10-comment threshold, it triggers a `comment_response` task
3. The Marketer reads the top comments (sorted by likes/relevance) and generates thoughtful replies
4. Replies are queued for posting via Postiz API (not instant — stagger over 1-2 hours to appear natural)
5. Each reply provides additional value or answers a question (never generic "thanks!" replies)

**Rules:**
- Maximum 5 replies per post (more than that looks desperate)
- Prioritize replying to questions and genuine engagement over simple praise
- Never argue with negative comments — acknowledge, redirect, or ignore
- If a comment asks about a feature the app doesn't have, log it as a feature request in `projects/<slug>/feedback/`
- Space replies 15-30 minutes apart to avoid triggering spam detection

**Configuration:**
```yaml
# In openclaw.yaml
marketing:
  engagement_auto_reply:
    enabled: true
    comment_threshold: 10     # Minimum comments to trigger
    max_replies_per_post: 5
    reply_delay_minutes: 15   # Minimum delay between replies
    platforms: ["tiktok", "instagram"]  # YouTube comments handled differently
```

**Why this matters:** Platform algorithms interpret creator engagement as a signal that the content is worth promoting. A post where the creator replies to comments typically sees 20-40% more reach in the 48 hours following the engagement.

## In-App Events

Apple's **In-App Events** feature displays time-limited event cards directly in App Store search results and on the app's product page. These cards drive installs by creating urgency and surfacing the app in seasonal searches.

**What are In-App Events:**
- Promotional cards that appear in the App Store for a defined time period
- Show up in search results, editorial features, and the app's product page
- Support custom imagery, short description, and deep links into the app
- Free to create — no ad spend required

**Seasonal calendar (automate these annually):**

| Event | Timing | Applicable Categories | Example Event |
|-------|--------|----------------------|---------------|
| New Year / Fresh Start | Dec 28 - Jan 15 | Health, Productivity, Finance | "Start Your 2027 Habits" |
| Valentine's Day | Feb 7 - Feb 14 | Lifestyle, Health | "Self-Care Challenge" |
| Back to School | Aug 1 - Sep 7 | Education, Productivity | "Study Smarter This Semester" |
| Fall Reset | Sep 15 - Oct 1 | Productivity, Health | "Fall Routine Builder" |
| Holiday Season | Nov 20 - Dec 31 | All categories | "Holiday [Feature] Challenge" |

**Implementation:**

The Shipper configures In-App Events via the **App Store Connect API**:

```json
// Example In-App Event payload
{
  "type": "inAppEvents",
  "attributes": {
    "referenceName": "new-year-habits-2027",
    "deepLink": "routinerest://event/new-year-challenge",
    "territorySchedules": [
      {
        "publishStart": "2026-12-28T00:00:00Z",
        "eventStart": "2027-01-01T00:00:00Z",
        "eventEnd": "2027-01-15T23:59:59Z"
      }
    ],
    "eventState": "ACCEPTED",
    "badge": "challenge",
    "localizations": [
      {
        "locale": "en-US",
        "name": "New Year Habit Challenge",
        "shortDescription": "Build 3 new habits in 14 days",
        "longDescription": "Start 2027 right. Pick 3 habits, track daily, and see your streak grow."
      }
    ]
  }
}
```

**Automation pipeline:**
1. A cron job checks the seasonal calendar 14 days before each event window
2. The Shipper generates event metadata (name, description, imagery) tailored to each live app
3. The Marketer creates a matching event card image (1920x1080, following Apple's guidelines)
4. The Shipper submits the event via the App Store Connect API
5. The Marketer schedules social media posts timed to the event launch for cross-promotion
6. After the event ends, engagement metrics are logged to `content_performance` for future optimization

**In-app support:**
The Builder includes a lightweight event handler in each app. When a user opens the app via an In-App Event deep link, the app surfaces the relevant feature or challenge directly — skipping the home screen and reducing friction.

In-App Events are underused by most developers, which makes them a low-effort, high-impact channel for driving incremental installs.
