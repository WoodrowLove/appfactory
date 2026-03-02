# Revenue Targets

## Goal

**$5,000-$10,000 MRR within 6 months of first app launch.**

This document breaks down how we get there.

## Unit Economics

### Per-Subscriber Economics

| Metric | Value | Notes |
|--------|-------|-------|
| Average subscription price | $4.99/month | Category-weighted average |
| Apple commission (year 1) | 30% | Small Business Program |
| Apple commission (year 2+) | 15% | After 12 months of subscriber retention |
| Net revenue per subscriber (year 1) | $3.49/month | $4.99 × 0.70 |
| Net revenue per subscriber (year 2+) | $4.24/month | $4.99 × 0.85 |
| Average monthly churn | 8-12% | Industry average for utility apps |
| Average subscriber lifetime | 8-12 months | 1 / churn_rate |
| Lifetime value (LTV) | $28-$42 | Net revenue × lifetime |

### Per-App Economics

| Metric | Target | Notes |
|--------|--------|-------|
| Monthly installs | 500-1,000 | From organic + marketing |
| Paywall view rate | 40-60% | % of users who see the paywall |
| Trial start rate | 25-35% | % of paywall viewers who start trial |
| Trial-to-paid conversion | 12-18% | % of trials that convert to paid |
| Steady-state subscribers per app | 20-50 | After 3-6 months |
| MRR per app | $70-$175 | Net revenue |

### Funnel Math (per app, per month)

```
1,000 installs
  × 50% see paywall = 500
  × 30% start trial = 150
  × 15% convert to paid = 22.5 new subscribers

Minus churn (10% of existing):
  Month 1: 22 subscribers → $77 MRR
  Month 2: 22 + 20 = 42 → $147 MRR
  Month 3: 42 + 18 = 60 → $209 MRR
  Month 6: ~80 subscribers → $279 MRR (approaching steady state)
```

## Revenue Projection Model

### Assumptions
- Ship 2 apps per month (conservative)
- Each app reaches steady state in 3-6 months
- 70% of apps succeed (pass quality, get approved, gain traction)
- 30% of apps underperform and contribute minimal revenue

### Month-by-Month Projection

| Month | Apps Shipped (cumulative) | Live Apps | Total Subscribers | Net MRR |
|-------|--------------------------|-----------|-------------------|---------|
| 1 | 2 | 1-2 | 15-30 | $52-$105 |
| 2 | 4 | 3-4 | 60-120 | $209-$419 |
| 3 | 6 | 4-5 | 130-250 | $454-$873 |
| 4 | 8 | 6-7 | 220-400 | $768-$1,396 |
| 5 | 10 | 7-9 | 350-600 | $1,222-$2,094 |
| 6 | 12 | 8-10 | 500-900 | $1,745-$3,141 |
| 9 | 18 | 12-15 | 900-1,600 | $3,141-$5,584 |
| 12 | 24 | 16-20 | 1,400-2,800 | $4,886-$9,772 |

**$5,000 MRR target**: Achievable between month 9-12 with conservative assumptions.
**$10,000 MRR target**: Achievable by month 12-15 if apps perform at the higher end.

### Accelerators

These factors can speed up the timeline:

1. **Viral content**: A single TikTok with 1M+ views can drive 5,000-10,000 installs in a week
2. **App Store featuring**: Apple's editorial team features apps, driving massive organic traffic
3. **Annual subscriptions**: Users who switch to annual have 0% monthly churn for 12 months
4. **Cross-promotion**: Users of one app converting to another app at higher rates
5. **ASO optimization**: Improving keywords and listing quality over time increases organic discovery

### Decelerators

These factors can slow down the timeline:

1. **Apple rejections**: Each rejection adds 1-2 weeks to the timeline
2. **Quality gate failures**: Multiple review cycles consume compute without revenue
3. **Market misses**: Apps that don't resonate despite research
4. **Increased competition**: New competitors entering validated niches
5. **Account issues**: Guideline 4.3 flags slowing down review times

## Cost vs. Revenue

### Monthly P&L Projection

| Month | Revenue (Net) | Costs | Profit | Cumulative |
|-------|--------------|-------|--------|------------|
| 1 | $80 | $400 | -$320 | -$320 |
| 2 | $300 | $450 | -$150 | -$470 |
| 3 | $650 | $450 | $200 | -$270 |
| 4 | $1,100 | $500 | $600 | $330 |
| 5 | $1,650 | $500 | $1,150 | $1,480 |
| 6 | $2,400 | $500 | $1,900 | $3,380 |
| 9 | $4,300 | $550 | $3,750 | $13,880 |
| 12 | $7,300 | $550 | $6,750 | $34,130 |

**Break-even**: Month 3-4 (monthly). Cumulative break-even by month 4.

**First-year projected profit**: $30,000-$40,000 (conservative to moderate estimate).

## Pricing Optimization

### A/B Testing Strategy

After the first 5 apps are live, begin pricing experiments:

1. **Test $3.99 vs. $4.99 vs. $6.99**: Compare conversion rates and MRR
2. **Test 3-day vs. 7-day vs. 14-day trials**: Find the optimal trial length per category
3. **Test annual discount levels**: 40% vs. 50% vs. 60% off monthly price
4. **Test paywall timing**: Show after 1 day vs. 3 days vs. on premium feature tap

RevenueCat supports A/B testing natively. Results feed back to the Architect for future app specs.

### Category-Specific Pricing Insights

Based on App Store benchmarks:

| Category | Optimal Monthly Price | Optimal Trial | Annual Conversion |
|----------|----------------------|---------------|-------------------|
| Health & Fitness | $4.99-$6.99 | 7 days | 30-40% |
| Productivity | $3.99-$4.99 | 7 days | 25-35% |
| Personal Finance | $4.99-$6.99 | 7 days | 35-45% |
| Lifestyle | $2.99-$3.99 | 7 days | 20-30% |
| Education | $4.99-$6.99 | 14 days | 30-40% |
| Utilities | $1.99-$3.99 (one-time) | N/A | N/A |

## Revenue Diversification

### Phase 1 (Months 1-6): Subscription Revenue Only
Focus on shipping apps and building subscriber base.

### Phase 2 (Months 6-12): Add Cross-Promotion
Use marketing channels to cross-sell apps within the portfolio. A user who subscribes to one app is 3-5x more likely to try another from the same developer.

### Phase 3 (Months 12+): Consider Additional Revenue Streams

| Stream | Potential | Effort |
|--------|-----------|--------|
| **Annual plans push** | +30% MRR from existing subscribers | Low (marketing only) |
| **Higher price tiers** | +20% MRR from power users | Medium (build premium tiers) |
| **Consumable IAP** | Variable, additive to subscription | Medium (add to AI features) |
| **Bundle discount** | Subscribe to 3+ apps at 30% off total | Low (App Store feature) |
| **Affiliate/partnership** | Commission from recommended services | Low (add referral links) |

## Key Metrics Dashboard

The Revenue tab in the Control Panel tracks:

### Real-Time
- **Total MRR**: Sum of all active subscriptions (net of Apple's commission)
- **Total ARR**: MRR × 12
- **Active trials**: Users in free trial across all apps
- **Subscribers**: Total paying subscribers across all apps

### Trends
- **MRR growth rate**: Month-over-month percentage change
- **Net subscriber growth**: New subscribers minus churned subscribers
- **Revenue per app**: Which apps are performing best
- **Churn by cohort**: Are newer users churning less (product improvement signal)

### Per-App
- **Installs**: Total and monthly
- **Conversion funnel**: Install → Paywall → Trial → Paid
- **MRR**: Current monthly revenue
- **LTV**: Average lifetime value of subscribers
- **Rating**: App Store rating and trend

### Health Signals

| Signal | Green | Yellow | Red |
|--------|-------|--------|-----|
| Monthly churn | < 8% | 8-15% | > 15% |
| Trial conversion | > 15% | 10-15% | < 10% |
| App Store rating | > 4.3 | 3.8-4.3 | < 3.8 |
| Refund rate | < 3% | 3-8% | > 8% |
| MRR growth | > 15%/month | 5-15%/month | < 5%/month |

Red signals trigger alerts in the Control Panel and may prompt the Router to prioritize fixes or content changes.

## Long-Term Vision

### Year 1: Foundation
- 15-24 live apps
- $5,000-$10,000 MRR
- 5 social accounts with 10K-50K followers each
- Established pipeline running autonomously

### Year 2: Scale
- 30-40 live apps
- $15,000-$30,000 MRR
- Begin international expansion (localized apps)
- Launch Android versions of top-performing apps
- Consider hiring a part-time designer for custom assets

### Year 3: Compound
- 50+ live apps
- $30,000-$50,000+ MRR
- Multiple developer accounts for geographic expansion
- Revenue reinvested into the broader Axia ecosystem
- AppFactory revenue funds AMORATRADER, Namora, and Sophos development

The factory is not the end goal. It's the engine that funds the vision.
