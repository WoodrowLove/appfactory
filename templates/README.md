# Templates

This directory contains code templates used by the Builder and Shipper agents.

## SwiftUI Templates (`swiftui/`)

| Template | Purpose | Used By |
|----------|---------|---------|
| `AppTemplate/` | Base Xcode project structure | Builder (starting point for every app) |
| `Onboarding.swift` | Configurable 3-5 screen onboarding flow | Builder |
| `Paywall.swift` | StoreKit 2 subscription paywall | Builder |
| `Settings.swift` | Standard settings screen | Builder |
| `GeminiWrapper.swift` | Gemini Flash AI integration | Builder (for AI-powered apps) |

## Fastlane Templates (`fastlane/`)

| Template | Purpose | Used By |
|----------|---------|---------|
| `Fastfile` | Build, screenshot, and deploy lanes | Shipper |
| `Appfile` | App identity configuration | Shipper |
| `Snapfile` | Screenshot device and language configuration | Shipper |

## Marketing Templates (`marketing/`)

| Template | Purpose | Used By |
|----------|---------|---------|
| `tiktok_hook.md` | TikTok video script template | Marketer |
| `app_promo.md` | App promo video script template | Marketer |

## How Templates Are Used

1. The Builder copies `AppTemplate/` as the starting point for every new app
2. It integrates `Onboarding.swift`, `Paywall.swift`, and `Settings.swift` into the project
3. If the spec calls for AI features, it adds `GeminiWrapper.swift`
4. The Shipper uses Fastlane templates when setting up the build and deployment pipeline
5. The Marketer uses marketing templates when generating content

Templates are designed to be customized per app. They handle the boilerplate so agents can focus on the unique features.
