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

## Anti-Patterns to Avoid

1. **Dark patterns**: No fake countdown timers, no "last chance" urgency, no hiding the close button on paywalls
2. **Aggressive upsells**: The paywall appears once during onboarding and is accessible from settings. No random pop-ups.
3. **Misleading trial terms**: Always clearly state "7-day free trial, then $4.99/month. Cancel anytime."
4. **Feature stripping**: Don't remove free features in updates to push premium. Users notice and leave 1-star reviews.
5. **Price anchoring tricks**: Don't show a fake "original price" crossed out. Just show the real price.

These practices not only get apps rejected by Apple — they destroy user trust and generate negative reviews that kill organic growth.
