# Monetization

## Strategy

Every app ships with a monetization model validated during the research phase. The default model is **freemium subscription with a 7-day free trial**, based on industry data showing that free trials convert approximately 6x better than hard paywalls.

The goal is not to extract maximum revenue from each user. The goal is to provide enough value that users willingly pay for premium features. Sustainable revenue comes from low churn, not aggressive monetization.

## Monetization Models

### 1. Freemium Subscription (Default)

```
Free tier: Core features that solve the primary pain point
Premium tier: AI features, advanced analytics, customization, sync
Price: $4.99/month or $29.99/year (50% annual discount)
Trial: 7-day free trial of premium
```

**When to use:** Most apps. Works especially well for:
- Apps with AI-powered features (the AI usage cost justifies ongoing subscription)
- Apps with data insights (more data over time = more value)
- Apps with sync features (CloudKit costs justify subscription)

**Free tier must be genuinely useful.** If the free tier is crippled, users leave 1-star reviews saying "everything is behind a paywall." The free tier should solve the core problem. The premium tier should solve it better.

### 2. One-Time Purchase

```
Price: $4.99 - $9.99
No subscription, no trial
Full access after purchase
```

**When to use:** Simple utility apps with no ongoing costs:
- QR code scanners
- Unit converters
- File managers
- Simple calculators

**Advantage:** Higher conversion rate (users prefer one-time purchases). No churn.
**Disadvantage:** No recurring revenue. Each new user is a one-time event.

### 3. Consumable IAP

```
Token/credit packs for AI features
$0.99 for 50 queries, $4.99 for 300 queries, $9.99 for 1000 queries
```

**When to use:** Apps where AI features are the core but usage varies wildly:
- AI writing assistants
- Image generators
- Translation tools

**Advantage:** Users pay proportionally to usage. Heavy users pay more.
**Disadvantage:** Harder to predict revenue. Users can feel nickel-and-dimed.

## StoreKit 2 Implementation

### Product Configuration

Products are configured in App Store Connect and mirrored in the app:

```swift
// StoreKitService.swift
@Observable
class StoreKitService {
    // Product IDs (set per app by Shipper agent)
    static let monthlyProductID = "com.appfactory.appname.premium.monthly"
    static let yearlyProductID = "com.appfactory.appname.premium.yearly"

    var products: [Product] = []
    var purchasedProductIDs: Set<String> = []
    var hasActiveSubscription: Bool { !purchasedProductIDs.isEmpty }

    private var updateListener: Task<Void, Error>?

    init() {
        updateListener = listenForTransactions()
        Task { await loadProducts() }
    }

    func loadProducts() async {
        do {
            products = try await Product.products(for: [
                Self.monthlyProductID,
                Self.yearlyProductID
            ])
        } catch {
            print("Failed to load products: \(error)")
        }
    }

    func purchase(_ product: Product) async throws -> Transaction? {
        let result = try await product.purchase()
        switch result {
        case .success(let verification):
            let transaction = try checkVerified(verification)
            await transaction.finish()
            await updatePurchasedProducts()
            return transaction
        case .userCancelled:
            return nil
        case .pending:
            return nil
        @unknown default:
            return nil
        }
    }

    func restorePurchases() async {
        try? await AppStore.sync()
        await updatePurchasedProducts()
    }

    private func updatePurchasedProducts() async {
        var purchased: Set<String> = []
        for await result in Transaction.currentEntitlements {
            if case .verified(let transaction) = result {
                purchased.insert(transaction.productID)
            }
        }
        purchasedProductIDs = purchased
    }

    private func listenForTransactions() -> Task<Void, Error> {
        Task.detached {
            for await result in Transaction.updates {
                if case .verified(let transaction) = result {
                    await transaction.finish()
                    await self.updatePurchasedProducts()
                }
            }
        }
    }

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified:
            throw StoreError.failedVerification
        case .verified(let safe):
            return safe
        }
    }
}

enum StoreError: Error {
    case failedVerification
}
```

### Free Trial Configuration

Configured in App Store Connect (not in code):
1. Create subscription group
2. Add monthly and yearly products
3. For each product, add an introductory offer:
   - Type: Free Trial
   - Duration: 7 days
   - Eligibility: New subscribers only

In the app, check eligibility:
```swift
func isEligibleForTrial(product: Product) async -> Bool {
    await product.subscription?.isEligibleForIntroOffer ?? false
}
```

### Entitlement Gating

```swift
// Use throughout the app to gate premium features
struct PremiumGateModifier: ViewModifier {
    @Environment(StoreKitService.self) var store

    func body(content: Content) -> some View {
        if store.hasActiveSubscription {
            content
        } else {
            // Show preview + "Upgrade to Premium" overlay
            content
                .blur(radius: 3)
                .overlay {
                    VStack {
                        Text("Premium Feature")
                            .font(.headline)
                        Button("Unlock with Free Trial") {
                            // Present paywall
                        }
                    }
                }
        }
    }
}
```

## Contextual Paywalls

Contextual paywalls — triggered the moment a user taps a premium feature — convert 2-3x better than paywalls shown only during onboarding. When a user is actively trying to accomplish something and hits a premium gate, their intent is at its peak. That is the highest-converting moment to present an upgrade prompt.

The default `paywall_strategy` in `onepager.json` should be `"hybrid"`:
- Show the paywall **once** during onboarding (introduces the premium tier)
- Show the paywall **again** when the user taps any gated feature (captures high-intent moments)

This hybrid approach maximizes both awareness (onboarding) and conversion (contextual).

### PremiumFeatureButton Modifier

```swift
// PremiumFeatureButton.swift
struct PremiumFeatureButton<Label: View>: ViewModifier {
    @Environment(StoreKitService.self) var store
    @State private var showPaywall = false

    let label: Label
    let premiumAction: () -> Void

    init(@ViewBuilder label: () -> Label, premiumAction: @escaping () -> Void) {
        self.label = label()
        self.premiumAction = premiumAction
    }

    func body(content: Content) -> some View {
        Button {
            if store.hasActiveSubscription {
                premiumAction()
            } else {
                showPaywall = true
            }
        } label: {
            label
        }
        .sheet(isPresented: $showPaywall) {
            PaywallView(source: .contextual)
        }
    }
}

// Usage in any view:
// Button with paywall gate
PremiumFeatureButton {
    Label("AI Insights", systemImage: "sparkles")
} premiumAction: {
    showAIInsights()
}
```

The `PaywallView` receives a `source` parameter (`.onboarding` or `.contextual`) so RevenueCat can track which placement converts better.

## RevenueCat Integration

RevenueCat provides subscription analytics and server-side receipt validation. It's optional but recommended for tracking key metrics.

### Why RevenueCat

- **Free** for up to $2,500 MRR (then 1% of MRR)
- Server-side receipt validation (more reliable than client-only)
- Real-time dashboards: MRR, trial conversions, churn, refunds
- Webhook notifications for subscription events
- Cross-platform support (if we ever expand to Android)

### Integration

```swift
// In AppDelegate or App init
Purchases.configure(withAPIKey: "appl_REVCAT_KEY")

// After StoreKit 2 purchase, also log to RevenueCat
Purchases.shared.logIn(appUserID) { customerInfo, created, error in
    // customerInfo.entitlements.all["premium"]?.isActive
}
```

### Metrics to Track

| Metric | Target | Notes |
|--------|--------|-------|
| Trial start rate | > 30% | Of users who see the paywall, how many start a trial |
| Trial-to-paid conversion | > 15% | Industry average for utility apps |
| Monthly churn | < 10% | Below 10% is healthy for subscription apps |
| MRR per app | > $500 | Target for each individual app |
| Refund rate | < 5% | High refund rate signals quality or expectation issues |
| LTV (lifetime value) | > $15 | At $4.99/month with 10% churn, LTV = $49.90 |

### RevenueCat Experiments

A/B testing pricing is built into the pipeline from Day 1 — not bolted on later.

After an app reaches **500 installs**, the system automatically begins A/B testing price points using RevenueCat Experiments. The typical test matrix is:

- **Variant A:** $3.99/month
- **Variant B:** $4.99/month (current default)
- **Variant C:** $6.99/month

RevenueCat splits users randomly and tracks trial start rate, trial-to-paid conversion, and LTV per cohort. After statistical significance is reached (usually 2-4 weeks at 500+ installs), the winning price is promoted to 100% of new users.

The `onepager.json` schema now includes a `pricing_experiment` field:

```json
{
  "monetization": {
    "model": "freemium_subscription",
    "base_price_monthly": 4.99,
    "pricing_experiment": {
      "enabled": true,
      "min_installs_to_start": 500,
      "variants": [3.99, 4.99, 6.99],
      "primary_metric": "ltv"
    }
  }
}
```

This ensures every app is automatically price-optimized once it has enough traffic. No manual intervention required.

## Pricing Strategy

### Research-Backed Defaults

| Category | Recommended Price | Trial Length | Rationale |
|----------|------------------|-------------|-----------|
| Health & Fitness | $4.99/month | 7 days | High willingness to pay, established category |
| Productivity | $3.99/month | 7 days | Price-sensitive audience, need to prove value fast |
| Personal Finance | $4.99/month | 7 days | Users expect to save more than they spend |
| Lifestyle | $2.99/month | 7 days | Lower perceived value, compensate with volume |
| Education | $4.99/month | 14 days | Longer trial needed to demonstrate learning outcomes |
| Utilities | $2.99 one-time | N/A | Users expect utilities to be cheap or free |

### Annual Pricing

Annual subscriptions should offer a 40-50% discount over monthly:
- $4.99/month → $29.99/year (50% off)
- $3.99/month → $24.99/year (48% off)
- $2.99/month → $19.99/year (44% off)

Annual plans reduce churn (users commit for a year) and increase LTV.

### Lifetime Purchase Option

Alongside monthly and yearly subscriptions, offer a **$79.99 lifetime purchase** for every subscription app. This is a one-time payment that grants permanent premium access.

**Why offer lifetime:**
- Lifetime purchasers have **zero churn** — they never cancel because there is nothing to cancel
- Net revenue per user is lower than a subscriber who stays 16+ months, but it captures **price-sensitive users who categorically refuse subscriptions**
- These users would otherwise never convert. A lifetime option turns them from free users into paying customers
- Lifetime purchases create goodwill and positive reviews ("love that they offer a lifetime option")

**Revenue math:**
- $79.99 × 0.70 (Apple's cut) = **$55.99 net**
- A $4.99/month subscriber who stays 12 months = $4.99 × 12 × 0.70 = $41.92 net
- So a lifetime purchase is actually more profitable than subscribers who churn within 16 months

RevenueCat tracks both subscription and lifetime purchase revenue in the same dashboard. Configure the lifetime product in App Store Connect as a non-consumable IAP and add it to the paywall as a third option below monthly and yearly.

### Regional Pricing

$4.99 USD should **not** be directly converted to local currency. A direct conversion makes the app unaffordable in emerging markets — $4.99 USD converts to approximately ₹415 INR, which is wildly overpriced for the Indian market.

Instead, use App Store Connect's **alternate price tier system** to set region-appropriate prices:

| Region | Monthly Price | Equivalent USD | Rationale |
|--------|--------------|----------------|-----------|
| United States | $4.99 | $4.99 | Base price |
| India | ₹129 | ~$1.55 | Price for volume in a massive market |
| Brazil | R$9.90 | ~$1.90 | Adjusted for local purchasing power |
| Indonesia | Rp 29,000 | ~$1.80 | Southeast Asia pricing |
| Turkey | ₺49.99 | ~$1.50 | High-inflation market |
| EU / UK / Japan | Market rate | ~$4.99 | Similar purchasing power to US |

Lower regional prices dramatically increase installs in emerging markets. A user paying ₹129/month who stays for a year generates more revenue than a user in India who never subscribes because ₹415/month is too expensive.

The **Shipper agent** configures regional pricing automatically using the App Store Connect API. It reads the base price from `onepager.json` and applies the regional multiplier table above when creating the subscription in App Store Connect. No manual price configuration needed.

### Pricing Iteration

After launch, the Marketing Engine tracks:
- Conversion rate at current price
- Comparison against competitors' pricing
- User feedback about pricing in App Store reviews
- RevenueCat A/B testing (if available)

If conversion is low, the Architect can recommend a price adjustment for the next app update.

## Apple's Commission

- **Apple takes 30%** of all in-app purchases and subscriptions in year 1
- **Apple takes 15%** of subscription revenue after a subscriber has been active for 12+ months (Small Business Program)
- The Small Business Program applies to developers earning under $1M/year — we qualify

**Net revenue per $4.99 subscription:**
- Year 1: $4.99 × 0.70 = $3.49
- Year 2+: $4.99 × 0.85 = $4.24

## Revenue Tracking

All revenue data flows to the Control Panel via RevenueCat webhooks:

```
RevenueCat → Webhook → Control Panel API → factory.db → Dashboard
```

The Revenue tab shows:
- Total MRR across all apps
- Per-app MRR breakdown
- Trial conversion funnel
- Churn trends
- Projected revenue (based on current growth rate)
- LTV analysis

## In-App Cross-Promotion

Build a **"More from AppFactory"** section in the Settings screen of every app. This module dynamically displays other apps in the portfolio with their icons, names, short descriptions, and direct App Store links.

This is the **highest-converting cross-sell channel** available and it costs nothing — no ad spend, no third-party networks, no revenue share. Users who already like one of your apps are the most likely audience to download another.

**How it works:**
- The Builder template includes a `CrossPromotionView` module in the Settings tab
- The module fetches a lightweight JSON manifest from a CDN (or bundled at build time) listing all published AppFactory apps
- It filters out the current app (no self-promotion) and shows the rest
- Each entry links to the App Store via `itms-apps://` URL scheme
- The module is unobtrusive — it sits at the bottom of Settings, not in the main experience

**Example manifest:**
```json
{
  "apps": [
    {
      "name": "FocusTimer Pro",
      "bundle_id": "com.appfactory.focustimer",
      "app_store_id": "1234567890",
      "tagline": "Stay focused. Get things done.",
      "icon_url": "https://cdn.appfactory.dev/icons/focustimer.png"
    }
  ]
}
```

As the portfolio grows, every new app automatically becomes a distribution channel for every other app. This creates a flywheel: more apps = more cross-promotion surface = more installs = more revenue.

## Win-Back Campaigns

When a subscriber churns, they are not gone forever. RevenueCat **promotional offers** allow you to offer a churned subscriber 1 month free to come back.

**How it works:**
1. RevenueCat fires a webhook when a subscription expires (churn event)
2. The **Marketer agent** receives the event and waits 7 days (let the user feel the absence of premium features)
3. After 7 days, the Marketer agent triggers a promotional offer via RevenueCat
4. The user receives a push notification or in-app message: "We miss you! Enjoy 1 month of Premium on us."
5. If the user accepts, their subscription reactivates with a free month, then resumes billing

**Why this works:**
- The user already demonstrated willingness to pay (they subscribed once)
- 1 free month costs nothing in marginal terms (no server costs for most apps)
- Win-back rates of 10-20% are typical, which directly reduces net churn
- RevenueCat handles the promotional offer mechanics — no custom StoreKit code needed

**Configuration:**
- Create promotional offers in App Store Connect for each subscription product
- Add the promotional offer signature key to RevenueCat
- The Marketer agent's win-back workflow triggers automatically — no manual campaigns

## SKAdNetwork Attribution

Every app template includes **SKAdNetwork attribution configuration** out of the box. This enables Apple's privacy-preserving ad attribution framework, which is essential for measuring the effectiveness of any future Apple Search Ads campaigns.

**What SKAdNetwork does:**
- Allows ad networks (including Apple Search Ads) to attribute app installs without revealing user-level data
- Works within Apple's App Tracking Transparency (ATT) framework — no user permission prompt required
- Reports conversion values back to ad networks so you can measure which ads drive paying users, not just installs

**Template configuration:**

In `Info.plist`, the Builder template includes the SKAdNetwork identifiers for Apple Search Ads and common ad networks:

```xml
<key>SKAdNetworkItems</key>
<array>
    <dict>
        <key>SKAdNetworkIdentifier</key>
        <!-- Apple Search Ads -->
        <string>YDX93A7ASS.skadnetwork</string>
    </dict>
</array>
```

**Why include this from Day 1:**
- Adding SKAdNetwork later requires an app update and resubmission
- Apple Search Ads is the highest-intent iOS ad channel (users are already searching for your category)
- Even if you do not run ads at launch, the configuration is ready when you are
- The Shipper agent ensures the correct network identifiers are included in every build

## Anti-Patterns to Avoid

1. **Dark patterns**: No fake countdown timers, no "last chance" urgency, no hiding the close button on paywalls
2. **Aggressive upsells**: The paywall appears once during onboarding and is accessible from settings. No random pop-ups.
3. **Misleading trial terms**: Always clearly state "7-day free trial, then $4.99/month. Cancel anytime."
4. **Feature stripping**: Don't remove free features in updates to push premium. Users notice and leave 1-star reviews.
5. **Price anchoring tricks**: Don't show a fake "original price" crossed out. Just show the real price.

These practices not only get apps rejected by Apple — they destroy user trust and generate negative reviews that kill organic growth.
