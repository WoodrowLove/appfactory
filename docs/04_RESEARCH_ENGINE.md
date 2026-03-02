# Research Engine

## Purpose

The Research Engine is the Scout agent's toolbox. It discovers app ideas by analyzing real user pain points across social media, app stores, and trend platforms. Every idea that enters the pipeline starts here — validated by data, not intuition.

## Data Sources

### 1. Reddit API

**Why Reddit:** Reddit is the single best source for unfiltered user complaints. Users on Reddit describe their problems in detail, compare existing solutions, and explicitly state what they wish existed.

**Target subreddits (by category):**

| Category | Subreddits |
|----------|-----------|
| Health & Fitness | r/sleep, r/fitness, r/loseit, r/nutrition, r/meditation, r/running, r/bodyweightfitness |
| Productivity | r/productivity, r/getdisciplined, r/ADHD, r/studytips, r/bulletjournal, r/timemanagement |
| Personal Finance | r/personalfinance, r/ynab, r/frugal, r/financialindependence, r/budgeting |
| Lifestyle | r/homeimprovement, r/cooking, r/gardening, r/organizationporn, r/declutter |
| Education | r/learnprogramming, r/languagelearning, r/GetStudying, r/Anki |
| Utilities | r/iphone, r/ios, r/shortcuts, r/apple |

**Search queries:**
```
"I wish there was an app"
"why doesn't any app"
"looking for an app that"
"frustrated with" + [category keyword]
"switched from" + [competitor name]
"better alternative to"
"this app sucks because"
```

**Signals to extract:**
- Post upvote count (demand signal)
- Comment count and sentiment (engagement signal)
- Number of "me too" replies (volume signal)
- Specific feature requests mentioned
- Named competitors and their shortcomings

**API setup:**
- Reddit API (OAuth2, read-only scope)
- Rate limit: 100 requests per minute per OAuth2 client
- Use `PRAW` (Python Reddit API Wrapper) or direct HTTP if running in Claude Code
- Store raw data in `research/<idea-slug>/reddit_raw.json`

### 2. X/Twitter API v2

**Why X:** Real-time sentiment and trending complaints. X is noisier than Reddit but catches emerging frustrations faster.

**Search operators:**
```
"wish there was an app" -is:retweet lang:en
"why is there no app" -is:retweet lang:en
"app for" + [category] + "sucks" -is:retweet lang:en
"need an app" + [category] -is:retweet lang:en
```

**Signals to extract:**
- Tweet impression count
- Reply count and sentiment
- Quote tweet count (signal of resonance)
- User profile: are they in the target demographic?

**API setup:**
- X API v2 (Basic tier: $100/month, 10,000 tweets/month read)
- Bearer token authentication
- Rate limit: 300 requests per 15-minute window
- Store in `research/<idea-slug>/x_raw.json`

### 3. App Store Review Mining

**Why reviews:** 1-star and 2-star reviews are product managers writing free specifications. They tell you exactly what's wrong with existing apps and what they'd pay for.

**Method:**
1. Identify top 10 apps in a target category
2. Pull their reviews (sorted by most recent) using the iTunes Search API or App Store scraping
3. Filter for 1-2 star reviews
4. Extract recurring themes using NLP/LLM analysis
5. Count frequency of each complaint type

**Target data per competitor:**
```json
{
  "app_name": "Sleep Cycle",
  "bundle_id": "com.northcube.SleepCycle",
  "rating": 4.6,
  "review_count": 189000,
  "price": "Free (Premium $39.99/year)",
  "last_updated": "2026-02-15",
  "top_complaints": [
    {"theme": "No routine builder", "frequency": 142, "sample": "Great at tracking but doesn't help me build better habits"},
    {"theme": "Alarm is unreliable", "frequency": 89, "sample": "..."},
    {"theme": "Subscription too expensive", "frequency": 67, "sample": "..."}
  ]
}
```

**API options:**
- iTunes Search API (free, limited to basic metadata)
- AppFollow or AppTweak API (paid, full review data)
- Scraping (use responsibly, respect robots.txt)

### 4. Apple Search Ads API

**Why Search Ads data:** Tells you what keywords people are actually searching for in the App Store and how competitive those keywords are.

**Data to extract:**
- Search volume estimate (low/medium/high) for target keywords
- Competition score (0-100)
- Suggested bid amount (indicator of commercial intent)
- Related keyword suggestions

**API setup:**
- Apple Search Ads API (requires Apple Search Ads account)
- Campaign management API can pull keyword insights
- Alternative: SearchAdsHQ or AppTweak for keyword intelligence

### 5. Product Hunt

**Why Product Hunt:** Identifies emerging categories and validated product concepts. If a simple tool gets 500+ upvotes on Product Hunt, there's demand.

**Method:**
- Monitor daily/weekly top products in relevant categories
- Note which categories are trending
- Identify gaps: popular Product Hunt tools that don't have good iOS equivalents
- Cross-reference with App Store to check if the gap exists on mobile

### 6. Google Trends

**Why Google Trends:** Confirms that demand is growing, not shrinking. An app idea for a declining trend is a bad investment.

**Method:**
- Check 12-month trend for primary keywords
- Compare related queries for emerging sub-topics
- Filter for mobile-related queries (add "app" or "iPhone" to searches)

## Scoring Methodology

### Demand Score (1-10)

| Score | Criteria |
|-------|----------|
| 1-2 | Fewer than 5 mentions across all sources. No clear pattern. |
| 3-4 | 5-20 mentions. Some pattern but could be noise. |
| 5-6 | 20-50 mentions. Clear pattern across multiple sources. Moderate engagement. |
| 7-8 | 50-200 mentions. Strong pattern. High engagement (100+ upvotes on Reddit posts). Multiple "me too" signals. |
| 9-10 | 200+ mentions. Viral complaints. Active communities asking for solutions. Google Trends showing growth. |

### Competition Score (1-10)

| Score | Criteria |
|-------|----------|
| 1-2 | 10+ direct competitors, including well-funded apps with 4.5+ ratings and frequent updates. Market is saturated. |
| 3-4 | 5-10 competitors. Strong incumbents but some have stale updates or missing features. |
| 5-6 | 3-5 competitors. Mixed quality. Clear feature gaps we can exploit. |
| 7-8 | 1-2 competitors. They're mediocre (< 4.0 rating) or haven't updated in 6+ months. |
| 9-10 | No direct competitors on iOS. The concept exists on web or Android but not as a quality iOS app. |

### Feasibility Score (1-10)

| Score | Criteria |
|-------|----------|
| 1-2 | Requires custom hardware, real-time multiplayer infrastructure, or technologies not available on iOS. |
| 3-4 | Requires complex server infrastructure, real-time data feeds, or technologies at the bleeding edge. |
| 5-6 | Requires moderate server support (e.g., user accounts with sync) or complex framework integration. |
| 7-8 | Fully buildable with SwiftUI + Apple frameworks. May need a simple API integration (Gemini Flash, CloudKit). |
| 9-10 | Pure SwiftUI app. Local data only. Standard Apple frameworks. Can be built from templates with minimal customization. |

### Validation Score

`total = demand + competition + feasibility`

| Total | Decision |
|-------|----------|
| >= 24 | Strong idea. Fast-track to spec phase. |
| 21-23 | Good idea. Proceed normally. |
| 18-20 | Borderline. Flag for human review before proceeding. |
| < 18 | Kill. Document the reason and move on. |

## Differentiation Check

Before any idea proceeds past validation, it must pass the differentiation check:

1. **Against competitors**: The idea must have a clear angle that no current App Store app addresses well. "Better UI" is not sufficient. "Different core value proposition" is required.

2. **Against our catalog**: The idea must be in a different category OR solve a fundamentally different problem from every other AppFactory app. Two fitness apps is fine if one tracks nutrition and one builds workout routines. Two sleep trackers is not fine.

3. **Apple Guideline 4.3 simulation**: Would an Apple reviewer, looking at this app alongside our other apps, flag it as spam? If yes, kill the idea.

This check is critical. See [Apple Compliance](13_APPLE_COMPLIANCE.md) for the full strategy.

## Research Cadence

- **Continuous research**: The Scout runs research cycles whenever the factory has fewer than 5 active projects and the idea queue is depleted.
- **Category rotation**: Each research cycle focuses on a different category to maintain diversity in the app catalog.
- **Seasonal awareness**: The Scout checks for seasonal demand patterns (New Year's resolutions → fitness, tax season → finance, back to school → education).
- **Trend monitoring**: Weekly scan of Product Hunt and Google Trends for emerging opportunities.

## Output Format

Every research cycle produces a file per idea:

```
research/<idea-slug>/
├── research.json        # Structured idea data (see schema)
├── reddit_raw.json      # Raw Reddit API responses
├── x_raw.json           # Raw X API responses
├── competitors.json     # Detailed competitor analysis
└── keywords.json        # Keyword research data
```

Only `research.json` is passed to the Architect. The raw data is archived for audit purposes and future re-analysis.

## Cost Estimation

| Service | Monthly Cost | Notes |
|---------|-------------|-------|
| Reddit API | Free | Read-only OAuth2 |
| X API v2 | $100/month | Basic tier, 10K tweets/month |
| Apple Search Ads | Variable | Pay-per-keyword-insight or use free tier |
| AppTweak/AppFollow | $100-300/month | Optional, for deep keyword/review data |
| Google Trends | Free | No API needed, web scraping or unofficial API |
| Product Hunt | Free | Public API or web scraping |

**Total research infrastructure cost: $100-$400/month**

This is the cheapest part of the pipeline and the highest leverage. Good research means building apps people actually want.
