# Chapter 32: Design a Notification System

> **Phase 6 — Classic Design Problems** · Estimated Time: 3 hours · Difficulty: 🟡 Medium · Type: 🏗️ DESIGN

[← Previous: Dropbox / Google Drive](./chapter-31-dropbox-google-drive.md) · [Next: Web Crawler →](../phase-7-advanced-designs/chapter-33-web-crawler.md)

---

## Why This Problem Matters

Notifications appear in nearly every system design as a supporting component. Knowing how to design push notifications (iOS/Android), SMS, email, and in-app notifications as a standalone system shows you can handle cross-cutting infrastructure. This is also a common standalone interview question at companies with notification-heavy products.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Push notifications** | iOS (APNs) and Android (FCM) |
| **SMS notifications** | Via third-party provider (Twilio) |
| **Email notifications** | Via SES/SendGrid |
| **In-app notifications** | Real-time via WebSocket, stored for later |
| **User preferences** | Opt-in/out per channel and category |
| **Templates** | Templated notifications with variable substitution |
| **Scheduling** | Schedule notifications for future delivery |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Notifications/day** | 10B (push + email + SMS + in-app) |
| **Delivery latency** | Push < 5s, Email < 30s, SMS < 30s |
| **Reliability** | No lost notifications |
| **No duplicates** | Exactly-once delivery (idempotent) |
| **Scalability** | Handle traffic spikes (marketing blasts) |

---

## Step 2: Estimation (3 min)

```
TOTAL QPS: 10B/day ÷ 86,400 ≈ 115K notifications/sec
Peak (marketing blast): ~500K/sec

BREAKDOWN:
  Push: 60% → 69K/sec
  In-app: 25% → 29K/sec
  Email: 10% → 11.5K/sec
  SMS: 5% → 5.8K/sec

STORAGE:
  In-app notification history: 10B × 500B = 5 TB/day
  Retain 30 days: ~150 TB → need Cassandra or S3 archival
```

---

## Step 3: High-Level Design (10 min)

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    NOTIFICATION SYSTEM                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────┐                                          │
│  │ Services    │ (Order Svc, User Svc, Marketing, etc.)  │
│  │ (producers) │                                          │
│  └──────┬─────┘                                          │
│         │ Send notification request                       │
│         ▼                                                │
│  ┌────────────────┐                                      │
│  │ Notification     │  • Validate request                │
│  │ Service          │  • Check user preferences           │
│  │                  │  • Apply rate limiting               │
│  │                  │  • Render template                   │
│  │                  │  • Route to correct channel          │
│  └──────┬─────────┘                                      │
│         │ Enqueue to channel-specific queues              │
│         ▼                                                │
│  ┌──────────────────────────────────────┐                │
│  │         Message Queues (Kafka)        │                │
│  │  ┌────────┐ ┌────┐ ┌─────┐ ┌──────┐│                │
│  │  │Push Q  │ │SMS │ │Email│ │In-app││                │
│  │  │        │ │ Q  │ │  Q  │ │  Q   ││                │
│  │  └───┬────┘ └─┬──┘ └──┬──┘ └──┬───┘│                │
│  └──────┼────────┼───────┼───────┼─────┘                │
│         ▼        ▼       ▼       ▼                       │
│  ┌──────────┐ ┌─────┐ ┌──────┐ ┌────────┐              │
│  │Push Wrkrs│ │SMS  │ │Email │ │In-app  │              │
│  │          │ │Wrkrs│ │Wrkrs │ │Wrkrs   │              │
│  │→ APNs   │ │→Twil│ │→SES/ │ │→WebSkt │              │
│  │→ FCM    │ │ io  │ │ Sndgd│ │→Cassandr│              │
│  └──────────┘ └─────┘ └──────┘ └────────┘              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### API Design

```
POST /api/v1/notifications/send
  Request:  { "user_ids": ["u_123", "u_456"],
              "template_id": "order_confirmed",
              "template_vars": { "order_id": "ORD_789", "total": "$42.50" },
              "channels": ["push", "email"],
              "priority": "high",
              "idempotency_key": "order_confirmed:ORD_789" }
  Response: { "notification_id": "notif_abc", "status": "queued" }

POST /api/v1/notifications/send-batch     (marketing blast)
  Request:  { "segment": "all_active_users",
              "template_id": "promo_summer_sale",
              "channels": ["push"],
              "scheduled_at": "2024-07-01T09:00:00Z" }
  Response: { "batch_id": "batch_123", "estimated_recipients": 5000000 }

GET /api/v1/users/{user_id}/notifications?cursor={cursor}&limit=20
  Response: { "notifications": [
    { "id": "notif_abc", "title": "Order Confirmed",
      "body": "Your order ORD_789 ($42.50) is confirmed!",
      "read": false, "created_at": "2024-01-15T10:30:00Z" },
    ...
  ], "unread_count": 5, "next_cursor": "..." }

PUT /api/v1/users/{user_id}/notifications/{notif_id}/read

PUT /api/v1/users/{user_id}/preferences
  Request:  { "preferences": [
    { "channel": "push", "category": "marketing", "enabled": false },
    { "channel": "email", "category": "transactional", "enabled": true },
    { "channel": "sms", "category": "security", "enabled": true }
  ] }
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: Notification Pipeline

```
WHEN A SERVICE SENDS A NOTIFICATION:

1. VALIDATE:
   → Is the request well-formed?
   → Does the template exist?
   → Is the user valid?

2. CHECK PREFERENCES:
   → SELECT enabled FROM notification_preferences
     WHERE user_id = ? AND channel = ? AND category = ?
   → If user opted out → skip this channel

3. RATE LIMIT:
   → Don't spam users. Limits:
     Push: max 10/hour per user
     Email: max 5/day per user
     SMS: max 3/day per user
   → Redis counter: INCR rate:{channel}:{user_id}
   → If over limit → drop or queue for later

4. DEDUPLICATE:
   → Redis SETNX with idempotency_key (TTL: 24h)
   → If key exists → duplicate, skip
   → If new → proceed

5. RENDER TEMPLATE:
   → Template: "Your order {{order_id}} ({{total}}) is confirmed!"
   → Variables: { order_id: "ORD_789", total: "$42.50" }
   → Result: "Your order ORD_789 ($42.50) is confirmed!"

6. ROUTE TO CHANNEL QUEUES:
   → Produce message to Kafka topic: notif.push, notif.email, etc.
   → Each channel has independent consumer workers

7. DELIVER:
   → Push workers: call APNs (iOS) or FCM (Android)
   → Email workers: call SES or SendGrid API
   → SMS workers: call Twilio API
   → In-app workers: push via WebSocket + store in Cassandra
```

### Deep Dive 2: Push Notification Delivery

```
PUSH NOTIFICATION ARCHITECTURE:

To send push to a user's phone:
  1. App registers with APNs/FCM → gets a device token
  2. Store device token: user_id → [device_token_1, device_token_2]
     (user may have multiple devices)
  3. When sending push:
     → Look up device tokens for user
     → Call APNs with token + payload (iOS)
     → Call FCM with token + payload (Android)

DEVICE TOKEN MANAGEMENT:
  CREATE TABLE device_tokens (
      user_id      BIGINT NOT NULL,
      device_token VARCHAR(255) NOT NULL,
      platform     ENUM('ios', 'android', 'web'),
      created_at   TIMESTAMP,
      last_used_at TIMESTAMP,
      
      PRIMARY KEY (user_id, device_token)
  );

  Token invalidation:
    APNs/FCM returns "invalid token" → delete from table
    User uninstalls app → token becomes invalid
    User logs out → remove token

HANDLING FAILURES:
  APNs/FCM may return errors:
    - Invalid token → delete token
    - Rate limited → backoff and retry
    - Server error → retry with exponential backoff
    - Payload too large → truncate message

  Retry policy: 3 retries with exponential backoff (1s, 5s, 25s)
  After 3 failures → Dead Letter Queue for manual investigation
```

### Deep Dive 3: Handling Traffic Spikes (Marketing Blasts)

```
PROBLEM: Marketing wants to send push to 50M users at once.
  50M notifications at once → 50M Kafka messages → workers overwhelmed

SOLUTION: Controlled fan-out with rate limiting

  1. Marketing creates a BATCH notification:
     "Send promo to segment: all_active_users (50M users)"

  2. Fan-out service:
     → Query user segment (50M user_ids)
     → Produce to Kafka at controlled rate: 100K messages/sec
     → Total time: 50M ÷ 100K = ~8 minutes to queue all
     
  3. Push workers consume at their natural rate
     → Auto-scale workers based on queue depth
     → Kafka absorbs the spike (buffer)

  4. Third-party rate limits:
     → APNs: ~50K notifications/sec per app
     → FCM: ~10K notifications/sec default
     → Workers self-throttle to stay within provider limits

SCHEDULING:
  Notifications can be scheduled for future delivery:
    Scheduled at 9:00 AM in user's timezone
    → Scheduler service checks every minute
    → At 9:00 AM UTC → produce to Kafka for UTC users
    → At 9:00 AM EST → produce for EST users
    → Timezone-aware delivery
```

### Deep Dive 4: In-App Notifications

```
IN-APP NOTIFICATIONS (badge count + notification center):

STORAGE: Cassandra
  Partition key: user_id
  Clustering key: created_at DESC (newest first)
  
  Perfect for: "Get user's last 20 notifications"
  One partition read, pre-sorted by time

DELIVERY:
  Option 1: WebSocket (if user is online)
    → Push notification in real-time
    → Update badge count

  Option 2: Polling (if WebSocket not available)
    → Client polls /notifications every 30 seconds
    → Response includes unread_count

UNREAD COUNT:
  Redis counter: unread:{user_id}
  INCR on new notification
  SET to 0 when user opens notification center
  
  Displayed as badge number on app icon and bell icon
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   NOTIFICATION SYSTEM ARCHITECTURE                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────┐           │
│  │ PRODUCERS (internal services)                         │           │
│  │ Order Svc │ Auth Svc │ Social Svc │ Marketing Engine  │           │
│  └──────────────────────┬───────────────────────────────┘           │
│                          ▼                                           │
│  ┌──────────────────────────────────────┐                           │
│  │ NOTIFICATION SERVICE                  │                           │
│  │ • Validate • Preferences • Rate limit │                           │
│  │ • Dedup • Template render • Route     │                           │
│  └─────────────────┬────────────────────┘                           │
│                    ▼                                                 │
│  ┌──────────────────────────────────────────────────┐               │
│  │              KAFKA (channel topics)               │               │
│  │ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────┐ │               │
│  │ │ Push   │ │ SMS    │ │ Email  │ │ In-app     │ │               │
│  │ └───┬────┘ └───┬────┘ └───┬────┘ └─────┬──────┘ │               │
│  └─────┼──────────┼─────────┼─────────────┼────────┘               │
│        ▼          ▼         ▼             ▼                         │
│  ┌──────────┐ ┌──────┐ ┌──────┐ ┌──────────────┐                  │
│  │Push Wrkrs│ │SMS   │ │Email │ │In-app Wrkrs  │                  │
│  │→ APNs   │ │Wrkrs │ │Wrkrs │ │→ WebSocket   │                  │
│  │→ FCM    │ │→Twilio│ │→SES  │ │→ Cassandra   │                  │
│  └─────┬────┘ └──────┘ └──────┘ └──────────────┘                  │
│        │                                                            │
│  ┌─────▼──────────┐  ┌──────────┐  ┌──────────────┐               │
│  │ Device Token DB │  │ DLQ      │  │ Preference DB│               │
│  │ (MySQL)         │  │ (failed  │  │ (MySQL)      │               │
│  └────────────────┘  │  notifs) │  └──────────────┘               │
│                       └──────────┘                                  │
│                                                                      │
│  ┌────────────────┐  ┌──────────────┐                               │
│  │ Redis           │  │ Scheduler Svc │                               │
│  │ • Rate limits   │  │ (future       │                               │
│  │ • Dedup keys    │  │  delivery)    │                               │
│  │ • Unread counts │  └──────────────┘                               │
│  └────────────────┘                                                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Queue per channel | Separate Kafka topics | Single queue | Each channel has different throughput and failure characteristics. SMS rate limits differ from push. |
| Push providers | APNs (iOS) + FCM (Android) | Self-hosted | Platform-specific requirements. Can't bypass APNs/FCM for mobile push. |
| Deduplication | Redis SETNX with TTL | DB unique constraint | Atomic, fast, auto-expiring. Prevents re-sending on retries. |
| Email/SMS delivery | Third-party (SES, Twilio) | Self-hosted SMTP/telco | Don't build email/SMS infrastructure. Managed services handle deliverability, compliance. |
| Template rendering | Server-side | Client-side | Consistent formatting, easy to update without app release. Supports all channels uniformly. |
| In-app storage | Cassandra | MySQL | Write-heavy (29K/sec), time-ordered by user. Cassandra partition per user, clustering by time. |
| Rate limiting | Redis sliding window | Application-level | Redis is fast, atomic (INCR + EXPIRE), and shared across all worker instances. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 4 notification channels? | Push (APNs/FCM), SMS (Twilio), Email (SES/SendGrid), In-app (WebSocket + stored in Cassandra). |
| 2 | How do you prevent duplicate notifications? | Idempotency key per notification. Redis SETNX — if key exists, skip. If new, proceed and set with 24h TTL. |
| 3 | How do you handle notification failures? | Retry 3 times with exponential backoff (1s, 5s, 25s). After 3 failures, move to Dead Letter Queue for manual investigation. |
| 4 | Why separate queues per channel? | Different throughput, latency, and failure modes. Push is fast (APNs/FCM), email is slower (SMTP), SMS has carrier rate limits. |
| 5 | How do you respect user preferences? | Notification preferences table: (user_id, channel, category, enabled). Check before sending. Never send to opted-out channels. |
| 6 | How do you prevent spamming users? | Rate limiting per user per channel. E.g., max 10 push/hour. Implemented with Redis counters + TTL. |
| 7 | How do you handle a 50M user marketing blast? | Controlled fan-out: produce to Kafka at 100K/sec (8 min to queue all). Workers auto-scale. Kafka buffers the spike. |
| 8 | How do you manage device tokens? | Store user_id → [device_tokens] in MySQL. Delete on APNs/FCM "invalid token" response. User may have multiple devices. |

---

[← Previous: Dropbox / Google Drive](./chapter-31-dropbox-google-drive.md) · [Next: Web Crawler →](../phase-7-advanced-designs/chapter-33-web-crawler.md)
