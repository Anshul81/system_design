# Chapter 27: Design a Chat System (WhatsApp / Slack)

> **Phase 6 — Classic Design Problems** · Estimated Time: 4 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Twitter / News Feed](./chapter-26-twitter-news-feed.md) · [Next: YouTube / Video Streaming →](./chapter-28-youtube-video-streaming.md)

---

## Why This Problem Matters

Chat is the second most common system design question. It tests WebSocket architecture, message ordering, delivery guarantees, presence (online/offline), group chat scaling, and push notifications. The statefulness of WebSocket connections is the core architectural challenge.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **1-on-1 chat** | Send/receive messages between two users |
| **Group chat** | Groups up to 500 members |
| **Online presence** | Show who's online/offline/last seen |
| **Read receipts** | Delivered ✓ and Read ✓✓ indicators |
| **Push notifications** | Notify offline users |
| **Media sharing** | Send images and files |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **DAU** | 500M |
| **Messages/day** | 50B |
| **Message latency** | < 100ms for online users |
| **Availability** | 99.99% |
| **Message ordering** | Guaranteed within a conversation |
| **Durability** | Messages must never be lost |

---

## Step 2: Estimation (3 min)

```
MESSAGE QPS:
  50B messages/day ÷ 86,400 ≈ 580K messages/sec
  Peak: ~1.2M messages/sec

CONCURRENT CONNECTIONS:
  500M DAU, ~50% online at peak = 250M concurrent WebSocket connections
  Each WS server handles ~50K connections → need ~5,000 WS servers

STORAGE:
  Avg message: 100 bytes text + 100 bytes metadata = 200 bytes
  50B × 200B = 10 TB/day (text only)
  Media: 5% have images (200KB avg) = 2.5B × 200KB = 500 TB/day
  → Media stored in S3, not in message DB

BANDWIDTH:
  580K msg/sec × 200 bytes = 116 MB/sec (manageable)
```

---

## Step 3: High-Level Design (10 min)

### Architecture Diagram

```
┌────────────┐    WebSocket     ┌──────────────┐
│ Client A    │◀══════════════▶│ WS Gateway    │
└────────────┘                 │ Server Pool   │
                                │ (5000 servers)│
┌────────────┐    WebSocket     │               │
│ Client B    │◀══════════════▶│               │
└────────────┘                 └───────┬───────┘
                                       │
                                ┌──────▼──────┐
                                │ Chat Service │
                                │ (routing +   │
                                │  persistence)│
                                └──────┬──────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
             ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
             │ Message DB   │   │ Message      │   │ Push Notif.  │
             │ (Cassandra)  │   │ Broker       │   │ Service      │
             │              │   │ (Kafka)      │   │ (APNs/FCM)   │
             └─────────────┘   └─────────────┘   └─────────────┘
```

### Message Flow

```
User A sends message to User B:

1. A's client sends message over WebSocket to WS Server 1
2. WS Server 1 → Chat Service
3. Chat Service:
   a. Persist message to Cassandra
   b. Check: Is User B online?
      → Yes: Lookup which WS server B is on
             Route message to that WS server → deliver via WebSocket
      → No:  Send push notification via APNs/FCM
4. B receives message in real-time (or via push notification)

┌──────┐  WS   ┌────────┐  route   ┌────────┐  WS   ┌──────┐
│User A │═════▶│WS Srv 1│────────▶│WS Srv 3│═════▶│User B │
└──────┘      └────────┘        └────────┘      └──────┘
                  │                                  │
                  ▼                                  │
           ┌──────────┐                             │
           │Cassandra  │  (message persisted)        │
           └──────────┘                             │
                                                     │
              If B is offline:                       │
              Push notification ─────────────────────┘
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: WebSocket Connection Management

```
CHALLENGE: Which WS server is User B connected to?

Session Registry (Redis):
  Key: user:{user_id}:session
  Value: { ws_server: "ws-server-3", connected_at: "..." }

When user connects:
  → WebSocket handshake with any WS server (via LB)
  → Register session in Redis: user:B → ws-server-3

When routing a message to User B:
  → Lookup Redis: user:B is on ws-server-3
  → Route message to ws-server-3 via internal message bus
  → ws-server-3 pushes to User B's WebSocket

When user disconnects:
  → Remove session from Redis
  → Update presence to "offline" / "last seen"

┌──────────────────────────────────────────────────────┐
│               SESSION REGISTRY (Redis)                │
│                                                      │
│  user:alice → { ws: "ws-042", since: "2024-01-..." } │
│  user:bob   → { ws: "ws-103", since: "2024-01-..." } │
│  user:carol → { ws: "ws-042", since: "2024-01-..." } │
│                                                      │
│  Cross-server routing via Redis Pub/Sub or Kafka:    │
│  Message for bob → publish to channel "ws-103"       │
│  → ws-103 receives → pushes to bob's WebSocket      │
└──────────────────────────────────────────────────────┘
```

### Deep Dive 2: Message Storage

```
Cassandra — perfect for chat messages:
  • Write-heavy (50B msgs/day)
  • Time-ordered (clustering key: message_timestamp)
  • Partition by conversation_id → all messages for a chat on one partition

Table: messages
  Partition Key: conversation_id
  Clustering Key: message_id (Snowflake, time-ordered) DESC

┌─────────────────┬────────────────┬───────────┬─────────────┐
│ conversation_id  │ message_id     │ sender_id │ content      │
│ (PK)             │ (CK DESC)      │           │              │
├─────────────────┼────────────────┼───────────┼─────────────┤
│ conv_123         │ msg_789        │ alice     │ "Hello!"     │
│ conv_123         │ msg_788        │ bob       │ "Hi there"   │
│ conv_123         │ msg_787        │ alice     │ "How are you"│
├─────────────────┼────────────────┼───────────┼─────────────┤
│ conv_456         │ msg_790        │ carol     │ "Meeting?"   │
└─────────────────┴────────────────┴───────────┴─────────────┘

Query: SELECT * FROM messages
       WHERE conversation_id = 'conv_123'
       ORDER BY message_id DESC
       LIMIT 50;

→ Single partition read, sorted, FAST.
```

### Deep Dive 3: Group Chat

```
Group message (group of 200 members):

User A sends "Hello group!" to Group G (200 members):

1. Persist message to Cassandra (conversation_id = group_G)
2. Lookup Group G members (from Group Service / cache)
3. For each online member:
   → Lookup their WS server in session registry
   → Route message to that WS server
4. For offline members:
   → Send push notification (batched)

Optimization for large groups:
  • Fan-out through Kafka: produce one message per member
  • WS servers subscribe to member-specific Kafka topics
  • Batch push notifications (don't send 200 individual pushes)

Group of 500 members, 50% online = 250 WebSocket pushes
  Per message: 250 Redis lookups + 250 inter-server routes
  With Kafka: 1 produce → consumers on each WS server filter relevant messages
```

### Deep Dive 4: Online Presence

```
HEARTBEAT-BASED PRESENCE:

Client sends heartbeat every 30 seconds:
  → WS Server receives heartbeat
  → Update Redis: user:alice:presence → { status: "online", last_heartbeat: T }

If no heartbeat for 60 seconds:
  → Mark user as "offline"
  → Broadcast presence change to user's contacts

Optimization — Don't broadcast every change:
  For users with 1000 contacts, broadcasting "online" to all 1000
  is expensive. Instead:
  
  • Only send presence to users who have the chat open with this user
  • Presence is "pulled" when opening a conversation, not "pushed"
  • Batch presence updates every 10-30 seconds
```

### Deep Dive 5: Read Receipts & Delivery Status

```
Message Status Flow:

SENT (✓):    Server received and persisted the message
DELIVERED (✓✓): Recipient's device received the message
READ (✓✓ blue): Recipient opened the conversation

User A sends message to User B:
  1. A → Server: message stored → Server → A: "SENT ✓"
  2. Server → B (via WS): message delivered → B → Server: "DELIVERED"
     → Server → A: "DELIVERED ✓✓"
  3. B opens conversation → B → Server: "READ msg_789"
     → Server → A: "READ ✓✓ blue"

Storage: delivery_status table or embedded in message metadata
  { msg_id: 789, status: "read", delivered_at: T1, read_at: T2 }
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     CHAT SYSTEM ARCHITECTURE                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────┐  WS  ┌────────────────────────────────────┐         │
│  │Clients  │═════▶│  WebSocket Gateway Servers          │         │
│  │(mobile, │      │  (5000 servers, 50K conns each)    │         │
│  │ web)    │      └──────────────┬─────────────────────┘         │
│  └────────┘                     │                                │
│                          ┌──────▼──────┐                         │
│                          │ Chat Service │                         │
│                          └──────┬──────┘                         │
│                                 │                                │
│        ┌────────────────────────┼────────────────────────┐       │
│        │                        │                        │       │
│  ┌─────▼─────┐           ┌─────▼─────┐           ┌─────▼─────┐ │
│  │ Cassandra  │           │  Redis     │           │ Kafka      │ │
│  │ (messages) │           │ (sessions, │           │ (message   │ │
│  │            │           │  presence) │           │  routing)  │ │
│  └───────────┘           └───────────┘           └───────────┘ │
│                                                                  │
│  ┌───────────┐  ┌───────────┐  ┌─────────────┐                 │
│  │ S3 (media)│  │ Push Svc  │  │ Group Svc   │                 │
│  │ + CDN     │  │ (APNs/FCM)│  │ (membership)│                 │
│  └───────────┘  └───────────┘  └─────────────┘                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs Discussed

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Real-time transport | WebSocket | Long polling, SSE | Bidirectional needed for chat. WS is lowest latency. |
| Message storage | Cassandra | MySQL, MongoDB | Write-heavy, time-ordered, scales linearly. Partition by conversation_id. |
| Session registry | Redis | Database | Sub-ms lookups for "which server is user on?" Ephemeral data. |
| Cross-server routing | Kafka/Redis Pub/Sub | Direct server-to-server | Decoupled. WS servers subscribe to their own channel. |
| Presence | Heartbeat (30s) + pull | Push to all contacts | Push to all contacts doesn't scale. Pull on conversation open is efficient. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | Why WebSocket for chat instead of HTTP polling? | WebSocket provides full-duplex, persistent connections with sub-100ms delivery. Polling adds latency (polling interval / 2 average) and wastes bandwidth. |
| 2 | How do you route messages between WS servers? | Session registry in Redis maps user → WS server. Cross-server routing via Kafka or Redis Pub/Sub. |
| 3 | Why Cassandra for chat messages? | Write-heavy (50B msg/day), time-ordered (clustering key), partition by conversation_id (all chat messages together), scales linearly. |
| 4 | How does presence (online/offline) work? | Heartbeat every 30s via WebSocket. If no heartbeat for 60s → offline. Presence is pulled when opening a chat, not pushed to all contacts. |
| 5 | How do you handle group messages for 500 members? | Lookup all members, check who's online (session registry), route to their WS servers. Offline members get push notifications (batched). |
| 6 | What are the 3 message statuses? | Sent (✓): server persisted. Delivered (✓✓): recipient's device received. Read (✓✓ blue): recipient opened conversation. |

---

[← Previous: Twitter / News Feed](./chapter-26-twitter-news-feed.md) · [Next: YouTube / Video Streaming →](./chapter-28-youtube-video-streaming.md)
