# API Specifications

## Overview

AppFactory integrates with four third-party API services to operate the full pipeline from build to ship to market. This document is the canonical reference for every external API call the system makes, including authentication, endpoints, rate limits, error handling, and retry strategy.

**APIs covered:**

| API | Used By | Purpose |
|-----|---------|---------|
| App Store Connect | Shipper | App record creation, subscription setup, build management, submission |
| Postiz | Marketer | Social media scheduling, analytics, media upload |
| Google AI (Nano Banana Pro) | Shipper | App icon generation |
| Google AI (Gemini Flash) | Builder | In-app AI features |
| RevenueCat | Shipper, Control Panel | Subscription analytics, webhook events |

---

## 1. App Store Connect API

### Authentication

App Store Connect uses JWT (JSON Web Token) authentication signed with an ES256 private key.

**Required credentials (from `.env`):**

| Variable | Description | Example |
|----------|-------------|---------|
| `ASC_KEY_ID` | Key ID from App Store Connect | `ABC1234DEF` |
| `ASC_ISSUER_ID` | Issuer ID from App Store Connect | `57246542-96fe-1a63-e053-0824d011072a` |
| `ASC_KEY_PATH` | Absolute path to the `.p8` private key file | `/keys/AuthKey_ABC1234DEF.p8` |

**Token generation (Node.js):**

```javascript
import jwt from 'jsonwebtoken';
import fs from 'fs';

function generateASCToken() {
  const privateKey = fs.readFileSync(process.env.ASC_KEY_PATH, 'utf8');

  const now = Math.floor(Date.now() / 1000);

  const payload = {
    iss: process.env.ASC_ISSUER_ID,
    iat: now,
    exp: now + (20 * 60), // 20-minute expiry
    aud: 'appstoreconnect-v1',
  };

  const token = jwt.sign(payload, privateKey, {
    algorithm: 'ES256',
    header: {
      alg: 'ES256',
      kid: process.env.ASC_KEY_ID,
      typ: 'JWT',
    },
  });

  return token;
}
```

**Token lifecycle:**
- Tokens expire after 20 minutes
- Generate a new token at the 15-minute mark (before expiry)
- On 401 response: regenerate token immediately and retry the request once
- Never reuse tokens across sessions; generate fresh on each Shipper invocation

### Base URL

```
https://api.appstoreconnect.apple.com/v1
```

All requests include:

```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

### Endpoints

#### App Record Creation

**`POST /apps`** -- Create a new app record in App Store Connect.

```json
{
  "data": {
    "type": "apps",
    "attributes": {
      "name": "SleepWell Tracker",
      "bundleId": "com.appfactory.sleepwell",
      "sku": "sleepwell-tracker-001",
      "primaryLocale": "en-US"
    },
    "relationships": {
      "bundleId": {
        "data": {
          "type": "bundleIds",
          "id": "<bundle_id_resource_id>"
        }
      }
    }
  }
}
```

**Response:** 201 Created with the app resource ID.

---

#### Subscription Group

**`POST /subscriptionGroups`** -- Create a subscription group for the app.

```json
{
  "data": {
    "type": "subscriptionGroups",
    "attributes": {
      "referenceName": "Premium Access"
    },
    "relationships": {
      "app": {
        "data": {
          "type": "apps",
          "id": "<app_id>"
        }
      }
    }
  }
}
```

---

#### Subscription Product

**`POST /subscriptions`** -- Create an auto-renewable subscription within the group.

```json
{
  "data": {
    "type": "subscriptions",
    "attributes": {
      "name": "Monthly Premium",
      "productId": "com.appfactory.sleepwell.monthly",
      "familySharable": false,
      "reviewNote": "Unlocks AI sleep analysis, smart alarms, and detailed sleep reports.",
      "subscriptionPeriod": "ONE_MONTH",
      "groupLevel": 1
    },
    "relationships": {
      "group": {
        "data": {
          "type": "subscriptionGroups",
          "id": "<group_id>"
        }
      }
    }
  }
}
```

---

#### Pricing Schedule

**`POST /inAppPurchasePriceSchedules`** -- Set pricing for the subscription product.

```json
{
  "data": {
    "type": "inAppPurchasePriceSchedules",
    "relationships": {
      "inAppPurchase": {
        "data": {
          "type": "inAppPurchases",
          "id": "<subscription_id>"
        }
      },
      "manualPrices": {
        "data": [
          {
            "type": "inAppPurchasePrices",
            "id": "${price-1}"
          }
        ]
      },
      "baseTerritory": {
        "data": {
          "type": "territories",
          "id": "USA"
        }
      }
    }
  },
  "included": [
    {
      "type": "inAppPurchasePrices",
      "id": "${price-1}",
      "attributes": {
        "startDate": null
      },
      "relationships": {
        "inAppPurchasePricePoint": {
          "data": {
            "type": "inAppPurchasePricePoints",
            "id": "<price_point_id_for_4.99>"
          }
        }
      }
    }
  ]
}
```

---

#### Free Trial (Introductory Offer)

**`POST /subscriptionIntroductoryOffers`** -- Configure a 7-day free trial.

```json
{
  "data": {
    "type": "subscriptionIntroductoryOffers",
    "attributes": {
      "duration": "ONE_WEEK",
      "numberOfPeriods": 1,
      "offerMode": "FREE_TRIAL",
      "startDate": null,
      "endDate": null
    },
    "relationships": {
      "subscription": {
        "data": {
          "type": "subscriptions",
          "id": "<subscription_id>"
        }
      },
      "territory": {
        "data": {
          "type": "territories",
          "id": "USA"
        }
      }
    }
  }
}
```

---

#### Build Processing

**`GET /builds`** -- Check if an uploaded build has finished processing.

Query parameters:
- `filter[app]` -- App resource ID
- `filter[processingState]` -- `PROCESSING`, `FAILED`, `INVALID`, `VALID`
- `sort` -- `-uploadedDate` (most recent first)
- `limit` -- `1`

```
GET /builds?filter[app]=<app_id>&filter[processingState]=VALID&sort=-uploadedDate&limit=1
```

**Polling strategy:** Check every 60 seconds for up to 30 minutes. If the build is still `PROCESSING` after 30 minutes, flag for human review.

---

#### Version Management

**`POST /appStoreVersions`** -- Create a new App Store version.

```json
{
  "data": {
    "type": "appStoreVersions",
    "attributes": {
      "platform": "IOS",
      "versionString": "1.0.0",
      "copyright": "2026 AppFactory",
      "releaseType": "AFTER_APPROVAL",
      "earliestReleaseDate": null
    },
    "relationships": {
      "app": {
        "data": {
          "type": "apps",
          "id": "<app_id>"
        }
      },
      "build": {
        "data": {
          "type": "builds",
          "id": "<build_id>"
        }
      }
    }
  }
}
```

---

#### Submit for Review

**`POST /appStoreVersionSubmissions`** -- Submit the version for App Review.

```json
{
  "data": {
    "type": "appStoreVersionSubmissions",
    "relationships": {
      "appStoreVersion": {
        "data": {
          "type": "appStoreVersions",
          "id": "<version_id>"
        }
      }
    }
  }
}
```

**Pre-submission checklist (Shipper verifies all before calling):**
- App record exists
- Build is `VALID`
- Version is created with build attached
- Screenshots uploaded for all required device sizes
- App description, keywords, and support URL set
- Subscription products configured and approved
- Age rating questionnaire completed

---

#### Review Status

**`GET /appStoreVersions/{id}`** -- Check the review status of a submitted version.

Response `attributes.appStoreState` values:

| State | Meaning | Action |
|-------|---------|--------|
| `WAITING_FOR_REVIEW` | Queued | Poll every 30 minutes |
| `IN_REVIEW` | Being reviewed | Poll every 15 minutes |
| `READY_FOR_SALE` | Approved and live | Transition to MARKET phase |
| `REJECTED` | Rejected | Parse resolution center, flag for human review |
| `DEVELOPER_REJECTED` | Self-rejected | Shipper uses this to pull a bad submission |
| `PENDING_DEVELOPER_RELEASE` | Approved, waiting for manual release | Shipper triggers release |

---

#### In-App Events

**`POST /inAppEvents`** -- Create promotional In-App Events (used for launch campaigns).

```json
{
  "data": {
    "type": "inAppEvents",
    "attributes": {
      "referenceName": "Launch Week Special",
      "badge": "NEW_CONTENT",
      "deepLink": "sleepwell://premium",
      "purchaseRequirement": "NO_COST_ASSOCIATED",
      "purpose": "APPROPRIATE_FOR_ALL_USERS",
      "territorySchedules": [
        {
          "territories": ["USA"],
          "publishStart": "2026-04-01T00:00:00-07:00",
          "eventStart": "2026-04-01T00:00:00-07:00",
          "eventEnd": "2026-04-08T00:00:00-07:00"
        }
      ]
    },
    "relationships": {
      "app": {
        "data": {
          "type": "apps",
          "id": "<app_id>"
        }
      }
    }
  }
}
```

---

#### Screenshots and Previews

**`POST /appScreenshotSets`** -- Create a screenshot set for a display type.

```json
{
  "data": {
    "type": "appScreenshotSets",
    "attributes": {
      "screenshotDisplayType": "APP_IPHONE_67"
    },
    "relationships": {
      "appStoreVersionLocalization": {
        "data": {
          "type": "appStoreVersionLocalizations",
          "id": "<localization_id>"
        }
      }
    }
  }
}
```

Display types used by AppFactory:
- `APP_IPHONE_67` -- iPhone 16 Pro Max (6.7")
- `APP_IPHONE_61` -- iPhone 16 (6.1")
- `APP_IPAD_PRO_13` -- iPad Pro 13" (M4)

**`POST /appPreviewSets`** -- Upload promo video set (same structure, type `appPreviewSets`).

Screenshot and preview upload follows a three-step process:
1. Create the set (above)
2. Reserve an upload with `POST /appScreenshots` (includes `fileName`, `fileSize`)
3. Upload the binary to the returned `uploadOperations` URLs

---

#### App Store Reviews

**`GET /apps/{id}/appStoreReviews`** -- Fetch user reviews (used by the Control Panel for monitoring).

Query parameters:
- `sort` -- `-createdDate` (most recent)
- `limit` -- `20`

---

### Rate Limits

| Limit | Value |
|-------|-------|
| Requests per hour per API key | 500 |
| Concurrent connections | 10 |

**Rate limit headers returned:**
- `X-Rate-Limit` -- Total allowed
- `X-Rate-Limit-Remaining` -- Remaining in window

When `X-Rate-Limit-Remaining` drops below 50, the Shipper slows request frequency to one request every 10 seconds.

### Error Handling

| Status | Classification | Action |
|--------|---------------|--------|
| 200-201 | Success | Continue |
| 401 | JWT expired | Regenerate token, retry once |
| 409 | Conflict (resource exists) | Read existing resource, use that ID |
| 429 | Rate limited | Retry with exponential backoff |
| 400, 403, 404, 422 | Permanent error | Abort, log error, flag for human review |
| 500, 502, 503, 504 | Transient error | Retry with exponential backoff |

---

## 2. Postiz API

### Authentication

Postiz uses a simple API key passed in the request header.

**Required credentials (from `.env`):**

| Variable | Description | Example |
|----------|-------------|---------|
| `POSTIZ_API_URL` | Base URL of the Postiz instance | `http://localhost:4200/api/v1` |
| `POSTIZ_API_KEY` | API key for authentication | `pz_live_abc123def456...` |

All requests include:

```
X-API-Key: <api_key>
Content-Type: application/json
```

### Base URL

```
Configurable via POSTIZ_API_URL (default: http://localhost:4200/api/v1)
```

### Endpoints

#### Schedule a Post

**`POST /posts`** -- Schedule content for publishing on one or more platforms.

```json
{
  "platform": "tiktok",
  "content": "3 sleep mistakes that are ruining your mornings (and what to do instead) [thread]",
  "media_urls": [
    "https://postiz.local/media/sleep-tips-video-001.mp4"
  ],
  "scheduled_at": "2026-04-02T09:00:00Z",
  "tags": ["sleep", "health", "wellness"],
  "metadata": {
    "app_slug": "sleepwell-tracker",
    "content_type": "value",
    "campaign": "pre-launch"
  }
}
```

**Response:** 201 Created with `{ "id": "post_abc123", "status": "scheduled" }`

**Platform values:** `tiktok`, `instagram`, `youtube_shorts`, `twitter`, `threads`

---

#### Check Post Status

**`GET /posts/{id}`** -- Get the current status of a scheduled or published post.

Response `status` values:

| Status | Meaning |
|--------|---------|
| `scheduled` | Queued for future publishing |
| `publishing` | Currently being published |
| `published` | Successfully posted |
| `failed` | Publishing failed |
| `deleted` | Removed |

---

#### Post Analytics

**`GET /posts/{id}/analytics`** -- Get engagement metrics for a published post.

**Response:**

```json
{
  "post_id": "post_abc123",
  "platform": "tiktok",
  "metrics": {
    "views": 12450,
    "likes": 843,
    "comments": 67,
    "shares": 124,
    "saves": 89,
    "engagement_rate": 0.09,
    "reach": 11200
  },
  "fetched_at": "2026-04-03T14:30:00Z"
}
```

The Marketer agent polls analytics at 24h, 48h, and 7 days after publishing. Results are written to `projects/<slug>/marketing/analytics.json`.

---

#### Media Upload

**`POST /media`** -- Upload an image or video file, returns a URL for use in posts.

```
Content-Type: multipart/form-data

file: <binary>
type: "video" | "image"
```

**Response:**

```json
{
  "media_url": "https://postiz.local/media/abc123.mp4",
  "media_id": "media_abc123",
  "size_bytes": 4521984,
  "mime_type": "video/mp4"
}
```

**File limits:**
- Images: Max 10MB, formats: PNG, JPG, WebP
- Videos: Max 100MB, formats: MP4, MOV

---

#### List Integrations

**`GET /integrations`** -- List all connected social media platform accounts.

**Response:**

```json
{
  "integrations": [
    {
      "id": "int_001",
      "platform": "tiktok",
      "account_name": "@brand_wellness",
      "status": "connected",
      "connected_at": "2026-01-15T10:00:00Z"
    }
  ]
}
```

The Marketer checks this before posting to verify the target platform is connected. If not connected, the post is skipped and a warning is logged.

---

#### Delete Post

**`DELETE /posts/{id}`** -- Delete a scheduled (not yet published) post.

**Response:** 204 No Content

Only works on posts with `status: scheduled`. Published posts cannot be deleted through the API.

---

### Webhooks

Postiz sends webhook events to a configured URL. The Control Panel listens on:

```
POST /api/webhooks/postiz
```

**Webhook events:**

| Event | Trigger | Payload Key Fields |
|-------|---------|-------------------|
| `post_published` | Post successfully published | `post_id`, `platform`, `published_url` |
| `post_failed` | Publishing failed | `post_id`, `platform`, `error_message`, `error_code` |
| `engagement_update` | Metrics updated (periodic) | `post_id`, `metrics` |

**Webhook payload structure:**

```json
{
  "event": "post_published",
  "timestamp": "2026-04-02T09:00:15Z",
  "data": {
    "post_id": "post_abc123",
    "platform": "tiktok",
    "published_url": "https://tiktok.com/@brand_wellness/video/123456"
  }
}
```

**On `post_failed`:** Retry publishing up to 3 times with a 5-second delay between attempts. If all retries fail, mark the post as `failed` in the project's marketing state and alert the Control Panel.

### Rate Limits

| Deployment | Limit |
|------------|-------|
| Self-hosted | No limit |
| Cloud | 100 requests/minute |

Self-hosted is the default and recommended deployment. See `docs/15_DEPLOYMENT.md` for Postiz setup.

### Error Handling

| Status | Classification | Action |
|--------|---------------|--------|
| 200-204 | Success | Continue |
| 400, 422 | Invalid request | Abort, log error details |
| 401 | Bad API key | Abort, alert operator |
| 404 | Resource not found | Log warning, skip |
| 500, 502, 503 | Transient error | Retry 3x with 5s delay |

---

## 3. Google AI APIs

Both Google AI services use the same API key but serve different purposes in the pipeline.

**Required credentials (from `.env`):**

| Variable | Description | Used By |
|----------|-------------|---------|
| `GOOGLE_AI_API_KEY` | Google AI Studio API key | Builder (Gemini Flash), Shipper (Nano Banana Pro) |

### 3a. Nano Banana Pro (Icon Generation)

Used by the Shipper agent to generate app icons. See `docs/08_PACKAGING_SHIPPING.md` for prompt templates and icon differentiation rules.

#### Authentication

```
x-goog-api-key: <api_key>
Content-Type: application/json
```

#### Base URL

```
https://generativelanguage.googleapis.com/v1beta
```

#### Endpoint

**`POST /models/imagen-3.0-generate-002:predict`**

Generate app icon images from a text prompt.

**Request:**

```json
{
  "instances": [
    {
      "prompt": "Design a minimal, modern iOS app icon for SleepWell Tracker, a health app that tracks sleep quality and provides AI-powered insights. Style: Clean, flat design with a single recognizable symbol. Color palette: Deep indigo with soft lavender accent. Must NOT include: text, words, letters, photographs, or complex scenes. Must include: One clear crescent moon symbol. Background: Solid deep indigo. Resolution: 1024x1024 pixels."
    }
  ],
  "parameters": {
    "sampleCount": 3,
    "outputOptions": {
      "mimeType": "image/png"
    }
  }
}
```

**Response:**

```json
{
  "predictions": [
    {
      "bytesBase64Encoded": "<base64_encoded_png>",
      "mimeType": "image/png"
    },
    {
      "bytesBase64Encoded": "<base64_encoded_png>",
      "mimeType": "image/png"
    },
    {
      "bytesBase64Encoded": "<base64_encoded_png>",
      "mimeType": "image/png"
    }
  ]
}
```

**Post-processing (Shipper):**
1. Decode all 3 base64 images
2. Validate each is exactly 1024x1024 pixels
3. Validate no transparency (alpha channel)
4. Score each variant on simplicity, color distinctiveness, and relevance
5. Select the best variant
6. If all 3 are poor quality, flag for human review

#### Rate Limits

| Tier | Limit |
|------|-------|
| Free | 10 requests/minute |
| Paid | 60 requests/minute |

#### Cost

Approximately $0.04 per image generated. At 3 images per app, icon generation costs ~$0.12 per app.

#### Error Handling

| Status | Classification | Action |
|--------|---------------|--------|
| 200 | Success | Process images |
| 400 | Bad prompt | Revise prompt, retry once |
| 429 | Rate limited | Retry with exponential backoff |
| 500, 503 | Transient | Retry with exponential backoff |
| Persistent failure | All retries exhausted | Flag for human review |

---

### 3b. Gemini Flash (In-App AI)

Used by the Builder agent to integrate AI features into apps. Gemini Flash calls are embedded in the Swift app code and execute at runtime on the user's device.

#### Authentication

```
x-goog-api-key: <api_key>
Content-Type: application/json
```

The API key is bundled into the app's configuration (not hardcoded -- loaded from a plist or environment at build time).

#### Base URL

```
https://generativelanguage.googleapis.com/v1beta
```

#### Endpoints

**`POST /models/gemini-2.0-flash:generateContent`** -- Standard (non-streaming) generation.

**Request:**

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        {
          "text": "Based on this sleep data, provide 3 specific, actionable recommendations to improve sleep quality:\n\nAverage bedtime: 11:45 PM\nAverage wake time: 6:30 AM\nAverage sleep duration: 6h 45m\nSleep efficiency: 82%\nWake-ups per night: 3.2\nDeep sleep: 14%\nREM sleep: 18%"
        }
      ]
    }
  ],
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 1024,
    "topP": 0.95,
    "topK": 40
  },
  "systemInstruction": {
    "parts": [
      {
        "text": "You are a sleep health assistant. Provide evidence-based, concise recommendations. Never diagnose medical conditions. Always suggest consulting a doctor for persistent sleep issues."
      }
    ]
  }
}
```

**Response:**

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "Here are 3 recommendations based on your sleep data..."
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "safetyRatings": [...]
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 142,
    "candidatesTokenCount": 356,
    "totalTokenCount": 498
  }
}
```

---

**`POST /models/gemini-2.0-flash:streamGenerateContent`** -- Streaming generation for real-time UI updates.

Same request body as `generateContent`. Response is a stream of server-sent events:

```
data: {"candidates":[{"content":{"parts":[{"text":"Here "}]},"finishReason":"","index":0}]}

data: {"candidates":[{"content":{"parts":[{"text":"are "}]},"finishReason":"","index":0}]}

data: {"candidates":[{"content":{"parts":[{"text":"3 recommendations..."}]},"finishReason":"STOP","index":0}]}
```

**When to use streaming vs. standard:**
- Streaming: User-facing AI responses displayed in real-time (chat, analysis results)
- Standard: Background processing, batch operations, or when the full response is needed before display

#### Rate Limits

| Tier | Limit |
|------|-------|
| Free | 15 requests/minute |
| Paid | 1,000 requests/minute |

Free tier is acceptable for development and testing. Production apps must use a paid tier key.

#### Cost

| Token Type | Cost per 1M Tokens |
|------------|-------------------|
| Input | $0.075 |
| Output | $0.30 |

At typical usage (~500 tokens per request, ~200 requests/day per active user), the AI cost per user is approximately $0.003/day or $0.09/month -- well within the margin of a $4.99/month subscription.

#### Error Handling

| Status | Classification | Action |
|--------|---------------|--------|
| 200 | Success | Process response |
| 400 | Bad request (malformed content) | Log error, surface user-facing error |
| 429 | Rate limited | Retry with exponential backoff, surface "please wait" to user |
| 500, 503 | Transient | Retry with exponential backoff |
| Persistent failure | All retries exhausted | Surface user-facing error: "AI features temporarily unavailable" |

**Safety filtering:** If `finishReason` is `SAFETY`, the response was blocked by Google's content filters. Log the event, do not surface the blocked content, and prompt the user to rephrase their input.

---

## 4. RevenueCat API

RevenueCat provides subscription analytics, user entitlement management, and webhook notifications for subscription lifecycle events.

### Authentication

```
Authorization: Bearer <api_key>
Content-Type: application/json
```

**Required credentials (from `.env`):**

| Variable | Description | Used By |
|----------|-------------|---------|
| `REVENUECAT_API_KEY` | RevenueCat public API key | Shipper, Control Panel |
| `REVENUECAT_WEBHOOK_SECRET` | Shared secret for webhook signature verification | Control Panel |

### Base URL

```
https://api.revenuecat.com/v1
```

### Endpoints

#### Get Subscriber Status

**`GET /subscribers/{app_user_id}`** -- Get the full subscription status for a user.

**Response (abbreviated):**

```json
{
  "request_date_ms": 1711929600000,
  "subscriber": {
    "entitlements": {
      "premium": {
        "expires_date": "2026-05-01T00:00:00Z",
        "product_identifier": "com.appfactory.sleepwell.monthly",
        "purchase_date": "2026-04-01T00:00:00Z"
      }
    },
    "subscriptions": {
      "com.appfactory.sleepwell.monthly": {
        "expires_date": "2026-05-01T00:00:00Z",
        "is_sandbox": false,
        "original_purchase_date": "2026-04-01T00:00:00Z",
        "period_type": "normal",
        "purchase_date": "2026-04-01T00:00:00Z",
        "store": "app_store",
        "unsubscribe_detected_at": null
      }
    },
    "first_seen": "2026-04-01T00:00:00Z",
    "management_url": "https://apps.apple.com/account/subscriptions"
  }
}
```

---

#### Fetch Offerings

**`POST /subscribers/{app_user_id}/offerings`** -- Get available subscription offerings for a user.

This is typically called from the app at launch to determine which products and pricing to display. RevenueCat handles A/B testing of offerings via its dashboard.

---

### Webhooks

RevenueCat sends webhook events to the Control Panel at:

```
POST /api/webhooks/revenuecat
```

#### Webhook Verification

Every webhook includes a signature header. Verify it using the shared secret before processing:

```javascript
import crypto from 'crypto';

function verifyRevenueCatWebhook(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload, 'utf8')
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}
```

**Always reject webhooks with invalid signatures.** Log the event and do not process the payload.

#### Webhook Events

| Event | Trigger | Action |
|-------|---------|--------|
| `INITIAL_PURCHASE` | User subscribes for the first time | Log conversion, update project revenue metrics |
| `RENEWAL` | Subscription renews successfully | Log renewal, update MRR |
| `CANCELLATION` | User cancels (still active until period end) | Log churn signal, trigger win-back eligibility check |
| `EXPIRATION` | Subscription period ends after cancellation | Log churn, update active subscriber count |
| `BILLING_ISSUE` | Payment failed (grace period starts) | Log billing issue, track grace period |
| `PRODUCT_CHANGE` | User changes subscription tier | Log upgrade/downgrade, update revenue metrics |

**Webhook payload structure:**

```json
{
  "api_version": "1.0",
  "event": {
    "type": "INITIAL_PURCHASE",
    "id": "evt_abc123",
    "app_user_id": "user_12345",
    "product_id": "com.appfactory.sleepwell.monthly",
    "price_in_purchased_currency": 4.99,
    "currency": "USD",
    "purchased_at_ms": 1711929600000,
    "expiration_at_ms": 1714521600000,
    "store": "APP_STORE",
    "environment": "PRODUCTION",
    "is_family_share": false
  }
}
```

The Control Panel writes every webhook event to `factory.db` in the `revenue_events` table and updates the project's rolling MRR in `projects/<slug>/revenue.json`.

### Error Handling

| Status | Classification | Action |
|--------|---------------|--------|
| 200 | Success | Process response |
| 401 | Bad API key | Abort, alert operator |
| 404 | Subscriber not found | Expected for new users, handle gracefully |
| 429 | Rate limited | Retry with exponential backoff |
| 500, 503 | Transient | Retry with exponential backoff |

---

## 5. Common Patterns

### Authentication Configuration

All API credentials are stored in `.env` at the project root. Never commit this file. A `.env.example` template is provided with placeholder values.

```bash
# ── App Store Connect ──────────────────────────────────
ASC_KEY_ID=ABC1234DEF
ASC_ISSUER_ID=57246542-96fe-1a63-e053-0824d011072a
ASC_KEY_PATH=/keys/AuthKey_ABC1234DEF.p8

# ── Postiz ─────────────────────────────────────────────
POSTIZ_API_URL=http://localhost:4200/api/v1
POSTIZ_API_KEY=pz_live_abc123def456

# ── Google AI ──────────────────────────────────────────
GOOGLE_AI_API_KEY=AIzaSyA-your-key-here

# ── RevenueCat ─────────────────────────────────────────
REVENUECAT_API_KEY=appl_your_key_here
REVENUECAT_WEBHOOK_SECRET=whsec_your_secret_here

# ── AI Agents (not covered in this doc) ────────────────
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
```

**Credential-to-agent mapping:**

| Credential | Agent(s) | Purpose |
|------------|----------|---------|
| `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_KEY_PATH` | Shipper | JWT generation for App Store Connect |
| `POSTIZ_API_URL`, `POSTIZ_API_KEY` | Marketer | Social media scheduling and analytics |
| `GOOGLE_AI_API_KEY` | Builder (Gemini Flash), Shipper (Nano Banana Pro) | In-app AI features, icon generation |
| `REVENUECAT_API_KEY` | Shipper, Control Panel | Subscription management, revenue analytics |
| `REVENUECAT_WEBHOOK_SECRET` | Control Panel | Webhook signature verification |

---

### Standard Retry Strategy

All API integrations share the same retry wrapper. This ensures consistent, predictable behavior across the system.

**Parameters:**
- Max retries: 3
- Backoff: Exponential -- 1s, 4s, 16s (base 4^n seconds, n = retry number)
- Jitter: +/-500ms random to prevent thundering herd
- Circuit breaker: After 5 consecutive failures within a 10-minute window, stop retrying and flag for human review

**Implementation (Node.js):**

```javascript
class CircuitBreaker {
  constructor(threshold = 5, windowMs = 10 * 60 * 1000) {
    this.threshold = threshold;
    this.windowMs = windowMs;
    this.failures = [];
    this.open = false;
  }

  recordFailure() {
    const now = Date.now();
    this.failures.push(now);
    // Remove failures outside the window
    this.failures = this.failures.filter(t => now - t < this.windowMs);
    if (this.failures.length >= this.threshold) {
      this.open = true;
    }
  }

  recordSuccess() {
    this.failures = [];
    this.open = false;
  }

  isOpen() {
    if (!this.open) return false;
    // Auto-reset after the window passes with no new failures
    const now = Date.now();
    this.failures = this.failures.filter(t => now - t < this.windowMs);
    if (this.failures.length < this.threshold) {
      this.open = false;
      return false;
    }
    return true;
  }
}

const TRANSIENT_ERRORS = new Set([429, 500, 502, 503, 504]);
const TRANSIENT_CODES = new Set(['ETIMEDOUT', 'ECONNRESET', 'ECONNREFUSED', 'EPIPE']);

async function apiRequest(fn, options = {}) {
  const {
    maxRetries = 3,
    circuitBreaker = null,
    context = 'api_request',
  } = options;

  if (circuitBreaker?.isOpen()) {
    throw new Error(
      `[${context}] Circuit breaker is open. ` +
      `${circuitBreaker.threshold} consecutive failures in the last ` +
      `${circuitBreaker.windowMs / 60000} minutes. Flagged for human review.`
    );
  }

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const result = await fn();
      circuitBreaker?.recordSuccess();
      return result;
    } catch (error) {
      const status = error.response?.status;
      const code = error.code;

      const isTransient =
        TRANSIENT_ERRORS.has(status) ||
        TRANSIENT_CODES.has(code);

      // Special case: ASC 401 = expired JWT, regenerate and retry once
      if (status === 401 && context === 'app_store_connect') {
        throw new TokenExpiredError('JWT expired, regenerate and retry');
      }

      if (!isTransient) {
        // Permanent error -- do not retry
        circuitBreaker?.recordFailure();
        throw error;
      }

      if (attempt === maxRetries) {
        // All retries exhausted
        circuitBreaker?.recordFailure();
        throw new Error(
          `[${context}] All ${maxRetries} retries exhausted. ` +
          `Last error: ${status || code} - ${error.message}`
        );
      }

      // Exponential backoff with jitter
      const baseDelay = Math.pow(4, attempt) * 1000; // 1s, 4s, 16s
      const jitter = (Math.random() - 0.5) * 1000;   // +/- 500ms
      const delay = Math.max(0, baseDelay + jitter);

      console.warn(
        `[${context}] Attempt ${attempt + 1}/${maxRetries + 1} failed ` +
        `(${status || code}). Retrying in ${Math.round(delay)}ms...`
      );

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

**Usage example:**

```javascript
const ascCircuitBreaker = new CircuitBreaker(5, 10 * 60 * 1000);

// Wrap any API call
const build = await apiRequest(
  () => ascClient.get(`/builds`, {
    params: {
      'filter[app]': appId,
      'filter[processingState]': 'VALID',
      'sort': '-uploadedDate',
      'limit': 1,
    },
  }),
  {
    maxRetries: 3,
    circuitBreaker: ascCircuitBreaker,
    context: 'app_store_connect',
  }
);
```

---

### Error Classification

All API errors are classified into two categories. This classification is universal across all integrations.

#### Transient Errors (Retry)

These are temporary failures. The request may succeed if retried.

| Error | Source | Meaning |
|-------|--------|---------|
| `429` | All APIs | Rate limited; slow down |
| `500` | All APIs | Internal server error |
| `502` | All APIs | Bad gateway |
| `503` | All APIs | Service unavailable |
| `504` | All APIs | Gateway timeout |
| `ETIMEDOUT` | Network | Connection timed out |
| `ECONNRESET` | Network | Connection reset by server |
| `ECONNREFUSED` | Network | Server refused connection |
| `EPIPE` | Network | Broken pipe |

#### Permanent Errors (Abort)

These indicate a problem with the request itself. Retrying will not help.

| Error | Source | Meaning | Action |
|-------|--------|---------|--------|
| `400` | All APIs | Bad request (malformed payload) | Log full request/response, flag for review |
| `401` | Postiz, Google, RevenueCat | Invalid API key | Abort, alert operator |
| `401` | App Store Connect | JWT expired | **Special case:** regenerate JWT, retry once |
| `403` | All APIs | Forbidden (insufficient permissions) | Abort, check API key scopes |
| `404` | All APIs | Resource not found | Log warning, handle gracefully |
| `409` | App Store Connect | Resource conflict (already exists) | Fetch existing resource, use that ID |
| `422` | All APIs | Unprocessable entity (validation error) | Log validation errors, fix payload |

#### Error Logging

Every API error is logged with the following structure:

```json
{
  "timestamp": "2026-04-02T09:15:30.123Z",
  "level": "error",
  "context": "app_store_connect",
  "operation": "POST /appStoreVersionSubmissions",
  "project": "sleepwell-tracker",
  "agent": "shipper",
  "status": 422,
  "error_code": "ENTITY_ERROR.ATTRIBUTE.INVALID",
  "error_detail": "The version string '1.0' is not valid. Must be in format X.Y.Z.",
  "request_id": "req_abc123",
  "attempt": 1,
  "will_retry": false
}
```

Errors are written to both the project-specific log (`projects/<slug>/logs/api_errors.json`) and the global log (`logs/api_errors.json`). The Control Panel surfaces errors with `will_retry: false` as alerts requiring human attention.

---

## 6. API Dependency Map

This section shows which pipeline phases depend on which APIs, and the order of operations.

```
BUILD Phase:
  └── Google AI (Gemini Flash) -- embedded in app code, not called during build

PACKAGE Phase:
  └── Google AI (Nano Banana Pro) -- icon generation (3 variants)

SHIP Phase:
  ├── App Store Connect -- full submission flow:
  │   ├── POST /apps (create record)
  │   ├── POST /subscriptionGroups (create group)
  │   ├── POST /subscriptions (create product)
  │   ├── POST /inAppPurchasePriceSchedules (set price)
  │   ├── POST /subscriptionIntroductoryOffers (free trial)
  │   ├── POST /appScreenshotSets + upload (screenshots)
  │   ├── POST /appPreviewSets + upload (promo video)
  │   ├── GET /builds (poll for processing)
  │   ├── POST /appStoreVersions (create version)
  │   ├── POST /appStoreVersionSubmissions (submit)
  │   └── GET /appStoreVersions/{id} (poll review status)
  └── RevenueCat -- configure offerings after ASC products are created

MARKET Phase:
  └── Postiz -- social media scheduling:
      ├── POST /media (upload assets)
      ├── POST /posts (schedule content)
      ├── GET /posts/{id} (check status)
      └── GET /posts/{id}/analytics (engagement tracking)

ONGOING (Control Panel):
  ├── RevenueCat webhooks -- subscription lifecycle events
  ├── Postiz webhooks -- post status and engagement
  └── App Store Connect -- review monitoring, In-App Events
```

---

## 7. Testing API Integrations

### Sandbox and Test Environments

| API | Test Environment |
|-----|-----------------|
| App Store Connect | Use sandbox testers in App Store Connect Users & Access |
| Postiz | Self-hosted instance with test social accounts |
| Google AI (Nano Banana Pro) | Free tier API key (10 req/min) |
| Google AI (Gemini Flash) | Free tier API key (15 req/min) |
| RevenueCat | Sandbox mode (automatically uses Apple sandbox) |

### Health Check Script

The Control Panel runs a health check every 5 minutes that verifies all API connections are live:

```javascript
async function checkAPIHealth() {
  const results = {};

  // App Store Connect: verify JWT generation and basic auth
  try {
    const token = generateASCToken();
    await ascClient.get('/apps', {
      params: { limit: 1 },
      headers: { Authorization: `Bearer ${token}` },
    });
    results.appStoreConnect = { status: 'healthy', latency_ms: elapsed };
  } catch (error) {
    results.appStoreConnect = { status: 'unhealthy', error: error.message };
  }

  // Postiz: verify API key and connectivity
  try {
    await postizClient.get('/integrations');
    results.postiz = { status: 'healthy', latency_ms: elapsed };
  } catch (error) {
    results.postiz = { status: 'unhealthy', error: error.message };
  }

  // Google AI: lightweight model info request
  try {
    await googleClient.get('/models/gemini-2.0-flash');
    results.googleAI = { status: 'healthy', latency_ms: elapsed };
  } catch (error) {
    results.googleAI = { status: 'unhealthy', error: error.message };
  }

  // RevenueCat: verify API key
  try {
    await revenuecatClient.get('/subscribers/health_check_user');
    results.revenuecat = { status: 'healthy', latency_ms: elapsed };
  } catch (error) {
    // 404 is expected for a non-existent user -- API is reachable
    if (error.response?.status === 404) {
      results.revenuecat = { status: 'healthy', latency_ms: elapsed };
    } else {
      results.revenuecat = { status: 'unhealthy', error: error.message };
    }
  }

  return results;
}
```

Health check results are stored in `factory.db` and displayed on the Control Panel dashboard. If any API is `unhealthy` for 3 consecutive checks, the system pauses all pipelines that depend on that API and alerts the operator.
