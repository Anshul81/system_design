# Chapter 2: Communication Patterns

> **Phase 0 — Networking & Communication** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium

[← Previous: APIs — REST, GraphQL, gRPC](./chapter-01-apis-rest-graphql-grpc.md) · [Next: Scaling 101 →](../phase-1-scalability/chapter-03-scaling-101.md)

---

## Learning Objectives

After this chapter, you will be able to:

- ✅ Compare synchronous vs asynchronous communication and choose the right one
- ✅ Explain the trade-offs between polling, long-polling, WebSockets, and SSE
- ✅ Design real-time features using the appropriate communication pattern
- ✅ Understand push vs pull architectures and their implications
- ✅ Know when to use webhooks for inter-service communication
- ✅ Draw communication pattern diagrams in a system design interview

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 0 (Internet, DNS, TCP, HTTP) | ✅ Yes |
| Chapter 1 (APIs) | ✅ Yes |
| Understanding of HTTP request-response | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Comparison exercises | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** Imagine you're waiting for a package.
- **Polling:** You walk to the door every 5 minutes to check if the delivery arrived.
- **Long-polling:** You stand at the door waiting. If the package arrives, great — you grab it. If 30 seconds pass with nothing, you close the door, then immediately open it again.
- **WebSockets:** You install a two-way intercom with the delivery driver. They tell you when they're arriving, and you can tell them "leave it at the back door."
- **SSE:** The delivery company sends you real-time tracking updates to your phone (one-way — you can't respond through the same channel).

**Interview relevance:** This is the most critical decision when designing real-time features. Design a chat system? You need WebSockets. Design a live dashboard? SSE might be better. Design a notification system? Could be polling, SSE, or push notifications. If you pick the wrong pattern, your design falls apart. Interviewers specifically test whether you understand these trade-offs.

---

## Visual Diagram — Communication Patterns Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMMUNICATION PATTERNS OVERVIEW                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SYNCHRONOUS (Request-Response):                                        │
│                                                                         │
│  Client ──── Request ────▶ Server                                      │
│  Client ◀─── Response ─── Server                                      │
│  (Client waits/blocks until response arrives)                          │
│                                                                         │
│  ────────────────────────────────────────────────────────               │
│                                                                         │
│  ASYNCHRONOUS (Fire-and-Forget / Event-Driven):                        │
│                                                                         │
│  Client ──── Message ────▶ [Queue] ──── Message ────▶ Server           │
│  (Client doesn't wait. Server processes when ready.)                   │
│                                                                         │
│  ────────────────────────────────────────────────────────               │
│                                                                         │
│  REAL-TIME (Server → Client Updates):                                  │
│                                                                         │
│  Polling:        Client ──?──▶ Server (every N seconds)                │
│  Long-Polling:   Client ──?──▶ Server ── waits... ──▶ Response         │
│  SSE:            Client ◀════ Server (one-way stream)                  │
│  WebSocket:      Client ◀═══▶ Server (full-duplex)                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### 1. Synchronous vs Asynchronous Communication

This is the fundamental fork in system design communication. Every time two components need to talk, you choose one.

#### Synchronous Communication

```
Service A                              Service B
   │                                       │
   │──── HTTP Request ────────────────────▶│
   │          (A is BLOCKED, waiting)       │
   │                                       │── Process
   │                                       │── request
   │                                       │── (100ms)
   │◀─── HTTP Response ──────────────────  │
   │                                       │
   │  (A can now continue)                 │
```

**Characteristics:**
- Client sends request and **blocks until response**
- Simple, easy to reason about — you know the result immediately
- Creates **tight coupling** — if Service B is down, Service A fails
- **Latency adds up** — if A calls B, and B calls C, total latency = latency(A→B) + latency(B→C)
- Used for: API calls, database queries, user-facing requests where you need the answer NOW

#### Asynchronous Communication

```
Service A            Message Queue            Service B
   │                     │                        │
   │── Publish msg ─────▶│                        │
   │   (A continues       │                        │
   │    immediately)      │                        │
   │                      │                        │
   │                      │──── Deliver msg ──────▶│
   │                      │                        │── Process
   │                      │                        │── message
   │                      │                        │── (500ms)
   │                      │                        │
   │  (A doesn't know     │                        │
   │   or care about      │                        │
   │   the result)        │                        │
```

**Characteristics:**
- Client sends message and **continues immediately** — doesn't wait for response
- **Loose coupling** — if Service B is down, messages queue up and are processed later
- **Better scalability** — decouple producers and consumers, scale independently
- **Eventually consistent** — the action will happen, but not necessarily right now
- Used for: Email sending, video processing, analytics events, order processing, anything that doesn't need an immediate response

#### When to Use Each

| Scenario | Sync or Async? | Why? |
|----------|----------------|------|
| User clicks "Submit Order" and needs confirmation | **Sync** (for validation), then **Async** (for processing) | Validate & return order ID immediately, process payment/shipping async |
| Sending a welcome email after signup | **Async** | User doesn't need to wait for email to be sent |
| Fetching user profile for display | **Sync** | You need the data to render the page |
| Processing a video upload | **Async** | Transcoding takes minutes, user can't wait |
| Real-time chat message delivery | **Async** (via WebSocket) | Messages are pushed, not pulled |
| Generating a analytics report | **Async** | Reports can take minutes/hours to generate |

**Interview tip:** The right answer is often "both." When a user places an order: synchronous API to validate the order and return a confirmation, then asynchronous message queue to process payment, update inventory, send confirmation email, etc.

---

### 2. Short Polling

The simplest way to get updates from a server: **ask repeatedly at fixed intervals**.

```
Client                                 Server
  │                                       │
  │── GET /api/notifications ────────────▶│── Query DB
  │◀── 200 { "notifications": [] } ──────│   (no new data)
  │                                       │
  │   (wait 5 seconds)                    │
  │                                       │
  │── GET /api/notifications ────────────▶│── Query DB
  │◀── 200 { "notifications": [] } ──────│   (still nothing)
  │                                       │
  │   (wait 5 seconds)                    │
  │                                       │
  │── GET /api/notifications ────────────▶│── Query DB
  │◀── 200 { "notifications":             │   (found one!)
  │    [{"msg": "New follower!"}] } ──────│
  │                                       │
```

| Pros | Cons |
|------|------|
| Dead simple to implement | Wastes bandwidth (most responses are empty) |
| Works with any HTTP client | Higher latency (average delay = polling interval / 2) |
| Stateless server | Puts unnecessary load on the server |
| Easy to debug and monitor | Doesn't scale well with many clients |

**When to use polling:**
- Low-frequency updates (checking order status every 30 seconds)
- When WebSocket infrastructure is overkill
- Legacy systems that can't support persistent connections
- When "near real-time" (seconds of delay) is acceptable

---

### 3. Long Polling

An improvement over short polling. The client sends a request, and the server **holds the connection open** until new data is available or a timeout expires.

```
Client                                 Server
  │                                       │
  │── GET /api/notifications ────────────▶│── Hold connection
  │        (connection hangs...)          │   open...
  │                                       │   waiting for
  │                                       │   new data...
  │                                       │
  │                                       │── (New data arrives
  │                                       │    after 15 seconds!)
  │◀── 200 { "notifications":             │
  │    [{"msg": "New follower!"}] } ──────│
  │                                       │
  │── GET /api/notifications ────────────▶│── Hold connection
  │        (immediately reconnect)        │   open again...
  │                                       │
  │        (30 second timeout)            │
  │◀── 204 No Content ───────────────────│   (timeout, no data)
  │                                       │
  │── GET /api/notifications ────────────▶│── Hold connection
  │        (immediately reconnect)        │   open again...
```

| Pros | Cons |
|------|------|
| Near real-time delivery | Server must hold many open connections |
| Less wasted bandwidth than polling | Hard to scale (each client = 1 open connection) |
| Works with standard HTTP | Timeout/reconnection logic adds complexity |
| Simple to implement on client-side | Each message requires a new HTTP request |

**When to use long polling:**
- When you need near real-time but can't use WebSockets
- Simple notification systems
- When the number of concurrent clients is manageable (< 10K)

**Interview tip:** Long polling was how Facebook chat worked before they switched to MQTT/WebSockets. It's a good "stepping stone" answer: "We could start with long polling for simplicity, but at scale we'd want WebSockets."

---

### 4. WebSockets

WebSocket is a **full-duplex** protocol that provides a persistent, bidirectional communication channel over a single TCP connection. Both client and server can send messages at any time.

```
HTTP Upgrade Handshake:

Client                                 Server
  │                                       │
  │── GET / HTTP/1.1 ────────────────────▶│
  │   Connection: Upgrade                 │
  │   Upgrade: websocket                  │
  │   Sec-WebSocket-Key: dGhlIHNh...      │
  │                                       │
  │◀── HTTP/1.1 101 Switching Protocols ──│
  │    Upgrade: websocket                 │
  │    Sec-WebSocket-Accept: s3pPLMBi...  │
  │                                       │
  │═══════════════════════════════════════│
  │       FULL-DUPLEX WEBSOCKET           │
  │       CONNECTION ESTABLISHED          │
  │═══════════════════════════════════════│
  │                                       │
  │── "Hello!" ──────────────────────────▶│  (client sends)
  │◀── "Hi back!" ───────────────────────│  (server sends)
  │◀── "New message from Alice" ─────────│  (server pushes)
  │── "Mark as read" ────────────────────▶│  (client sends)
  │                                       │
  │   (connection stays open indefinitely) │
```

#### WebSocket Architecture in System Design

```
                    ┌──────────────────────────────────────┐
                    │          WebSocket Servers             │
                    │  ┌─────────┐  ┌─────────┐  ┌───────┐│
 ┌────────┐  WS    │  │  WS     │  │  WS     │  │  WS   ││
 │Client 1 │───────▶│  │Server 1 │  │Server 2 │  │Srv 3  ││
 └────────┘        │  │ (5K     │  │ (5K     │  │(5K    ││
                    │  │  conns) │  │  conns) │  │conns) ││
 ┌────────┐  WS    │  └────┬────┘  └────┬────┘  └───┬───┘│
 │Client 2 │───────▶│       │            │           │     │
 └────────┘        └───────┼────────────┼───────────┼─────┘
                           │            │           │
                    ┌──────▼────────────▼───────────▼─────┐
                    │          Message Broker               │
                    │        (Redis Pub/Sub or Kafka)       │
                    │                                       │
                    │  When Client 1 (on Server 1) sends    │
                    │  a message to Client 2 (on Server 3), │
                    │  the broker routes it between servers  │
                    └───────────────────────────────────────┘

Challenge: WebSocket connections are STATEFUL.
Client 1 is connected to Server 1. If Server 1 goes down,
Client 1 must reconnect to a different server.
This is why we need a broker for cross-server communication.
```

| Pros | Cons |
|------|------|
| True real-time, bidirectional | Stateful — harder to scale horizontally |
| Low overhead after connection (no HTTP headers per message) | Need sticky sessions or a message broker |
| Great for high-frequency messaging | Load balancers must support WS connections |
| Native browser support | Connection management is complex (heartbeats, reconnection) |

**When to use WebSockets:**
- Chat applications (WhatsApp, Slack)
- Live collaboration (Google Docs cursor sharing)
- Real-time gaming
- Live sports scores, stock tickers
- Any feature requiring bidirectional, high-frequency communication

---

### 5. Server-Sent Events (SSE)

SSE is a **one-way** stream from server to client over a standard HTTP connection. The server can push events, but the client can only receive — to send data, the client uses regular HTTP requests.

```
Client                                 Server
  │                                       │
  │── GET /api/events ───────────────────▶│
  │   Accept: text/event-stream           │
  │                                       │
  │◀── HTTP/1.1 200 OK ─────────────────│
  │    Content-Type: text/event-stream    │
  │    Connection: keep-alive             │
  │                                       │
  │◀── data: {"type":"new_tweet"} ───────│  (push event 1)
  │                                       │
  │◀── data: {"type":"new_like"} ────────│  (push event 2)
  │                                       │
  │    (5 seconds of silence)             │
  │                                       │
  │◀── :keepalive ───────────────────────│  (heartbeat)
  │                                       │
  │◀── data: {"type":"new_follower"} ────│  (push event 3)
  │                                       │
  │    (connection stays open, server     │
  │     sends events as they happen)      │

Client JavaScript:
  const events = new EventSource('/api/events');
  events.onmessage = (e) => {
    const data = JSON.parse(e.data);
    // Handle server-sent event
  };
```

| Pros | Cons |
|------|------|
| Simpler than WebSockets | One-way only (server → client) |
| Automatic reconnection built into browser API | Limited to ~6 connections per domain in HTTP/1.1 |
| Text-based, easy to debug | Not suitable for bidirectional communication |
| Works through HTTP proxies and load balancers | Binary data support requires encoding |
| Built-in event ID for resuming after disconnect | Less widely supported for non-browser clients |

**When to use SSE:**
- Live notification feeds
- Real-time dashboards (metrics, monitoring)
- Stock price tickers
- Social media feeds (new tweets appearing)
- Any one-way server-to-client streaming

---

### 6. Push vs Pull Architectures

At a higher level, systems are either **push-based** (server sends data proactively) or **pull-based** (client requests data when needed).

```
PULL Architecture:
┌────────┐                        ┌────────┐
│ Client  │──── "Do you have ────▶│ Server  │
│         │      new data?" ──────│         │
│         │                       │         │
│         │◀─── "Here it is" ────│         │
└────────┘      (or "nothing")   └────────┘

PUSH Architecture:
┌────────┐                        ┌────────┐
│ Client  │                       │ Server  │
│         │                       │         │
│         │◀─── "New data for ───│         │
│         │      you!" ──────────│         │
└────────┘                       └────────┘
```

#### Push vs Pull Decision

| Factor | Pull | Push |
|--------|------|------|
| **Data freshness** | Stale (depends on poll interval) | Real-time |
| **Server load** | Can be high (many empty polls) | Lower (only sends when needed) |
| **Client complexity** | Simple (just HTTP calls) | Higher (must handle persistent connections) |
| **Server complexity** | Simple (stateless) | Higher (track connected clients) |
| **Scalability** | Easier (stateless) | Harder (stateful connections) |
| **Use case** | Email checking, batch processing | Chat, notifications, live data |

**Interview tip:** Many systems use a **hybrid** approach. Twitter's feed uses **fan-out-on-write (push)** for most users but **fan-out-on-read (pull)** for celebrity accounts. This comes up directly in the Twitter system design question.

---

### 7. Webhooks

Webhooks are **server-to-server push notifications** via HTTP POST. Instead of polling an API for changes, you register a callback URL and the external service sends you data when events occur.

```
Traditional Polling (client pulls):

Your Server                            External Service
    │                                        │
    │── GET /api/status ────────────────────▶│
    │◀── { "status": "pending" } ───────────│
    │                                        │
    │── GET /api/status ────────────────────▶│
    │◀── { "status": "pending" } ───────────│
    │                                        │
    │── GET /api/status ────────────────────▶│
    │◀── { "status": "completed" } ─────────│
    │                                        │
    (3 requests, 2 were wasted)

Webhook (server pushes):

1. Registration:
Your Server                            External Service
    │── POST /api/webhooks                   │
    │   { "url": "https://myapp.com/hook",  │
    │     "events": ["payment.completed"] }  │
    │──────────────────────────────────────▶│
    │◀── 201 Created ──────────────────────│

2. Event occurs:
External Service                       Your Server
    │                                        │
    │── POST https://myapp.com/hook ────────▶│
    │   { "event": "payment.completed",      │
    │     "data": { "amount": 99.99 } }      │
    │                                        │── Process
    │◀── 200 OK ────────────────────────────│
    │                                        │
    (1 request, exactly when needed)
```

| Pros | Cons |
|------|------|
| No wasted requests | Your server must be reachable (need public URL) |
| Real-time notification | Webhook delivery isn't guaranteed (need retries) |
| Simple to implement | Security (must verify webhook authenticity) |
| Industry standard for integrations | Must handle duplicate deliveries (idempotency) |

**Common webhook patterns:**
- **Stripe** → sends `payment.succeeded` webhooks to your server
- **GitHub** → sends push/PR events to your CI/CD pipeline
- **Twilio** → sends incoming SMS/call events to your server

---

## Comparison Table — All Communication Patterns

| Pattern | Direction | Latency | Complexity | Scalability | Best For |
|---------|-----------|---------|------------|-------------|----------|
| **Short Polling** | Client → Server | High (interval/2 avg) | Very Low | Good | Low-frequency updates |
| **Long Polling** | Client → Server | Low-Medium | Low | Moderate | Simple real-time, few clients |
| **WebSocket** | Bidirectional | Very Low | High | Challenging | Chat, gaming, collaboration |
| **SSE** | Server → Client | Very Low | Medium | Good | Dashboards, feeds, notifications |
| **Webhook** | Server → Server | Very Low | Medium | Good | Third-party integrations |

### Decision Tree for Interviews

```
Do you need real-time updates?
├── No → Standard HTTP request-response (REST/gRPC)
│
└── Yes →
    ├── Is it bidirectional (client AND server send data)?
    │   ├── Yes → WebSocket
    │   │         (Chat, gaming, live collaboration)
    │   │
    │   └── No (server → client only) →
    │       ├── High frequency (>1 msg/sec)? → SSE or WebSocket
    │       ├── Low frequency? → Long Polling or SSE
    │       └── Server-to-server? → Webhook + Message Queue
    │
    └── Is "near real-time" (few seconds delay) OK?
        ├── Yes → Short Polling (simplest)
        └── No → Long Polling, SSE, or WebSocket
```

---

## Real-World Example — Slack's Communication Architecture

```
Slack uses a combination of communication patterns:

┌──────────┐     WebSocket        ┌────────────────┐
│  Desktop  │◀═══════════════════▶│  Gateway        │
│  App      │                     │  Server         │
└──────────┘                     │  (per-user      │
                                  │   WS conn)      │
┌──────────┐     WebSocket        │                 │
│  Mobile   │◀═══════════════════▶│                 │
│  App      │                     └────────┬────────┘
└──────────┘                              │
                                           │ Internal: gRPC
                                           ▼
                                  ┌────────────────┐
                                  │  Message        │
                                  │  Server         │
                                  │                 │
                                  │  • Persists msg │
                                  │  • Fan out to   │
                                  │    channel      │
                                  │    members      │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌────────────────┐
                                  │  Message Queue   │
                                  │  (Kafka)         │
                                  │                 │
                                  │  • Push notif.  │
                                  │  • Search index │
                                  │  • Analytics    │
                                  └────────────────┘

Key decisions:
1. WebSocket for real-time message delivery (bidirectional needed)
2. gRPC for internal service communication (performance)
3. Kafka for async processing (push notifications, search indexing)
4. Fallback to long-polling for clients that can't use WebSocket
5. Mobile push notifications (APNs/FCM) for when app is in background
```

---

## Common Interview Questions

### Q1: "How would you design the communication layer for a chat application?"

**Model answer:**
"I'd use WebSockets for the primary communication channel. When a user opens the app, they establish a WebSocket connection to a gateway server. Messages are sent over this connection in both directions. For scalability, I'd use Redis Pub/Sub or Kafka to route messages between WebSocket servers — if User A is on Server 1 and User B is on Server 3, the message goes through the broker. For mobile users with the app in the background, I'd use push notifications via APNs (iOS) and FCM (Android). As a fallback for restrictive networks, I'd support long-polling."

### Q2: "Polling vs WebSocket — when would you use each?"

**Model answer:**
"Polling is simpler and stateless — the server doesn't track connected clients. I'd use it for low-frequency updates where seconds of delay are acceptable, like checking order status or a dashboard that refreshes every 30 seconds. WebSockets are for high-frequency, bidirectional communication — chat, gaming, collaborative editing — where you need sub-100ms delivery and both sides need to send data. The trade-off is WebSockets are stateful, making horizontal scaling more complex because you need session affinity and a message broker between servers."

### Q3: "What is SSE and when would you choose it over WebSocket?"

**Model answer:**
"Server-Sent Events is a one-way streaming protocol where the server pushes events to the client over a standard HTTP connection. I'd choose SSE over WebSocket when I only need server-to-client communication — live dashboards, notification feeds, real-time stock prices. SSE is simpler because it uses standard HTTP, works with existing load balancers and proxies without special configuration, and has automatic reconnection built into the browser API. WebSocket is overkill if the client never needs to send data through the same channel."

### Q4: "Explain webhooks and their challenges."

**Model answer:**
"Webhooks are HTTP callbacks — you register a URL with an external service, and they POST events to your URL when something happens. For example, Stripe sends a webhook when a payment succeeds. The challenges are: first, delivery isn't guaranteed — the receiving server might be down, so the sender needs retry logic with exponential backoff. Second, security — you must verify webhook signatures to prevent spoofing. Third, idempotency — retries might deliver the same event multiple times, so you need to handle duplicates. Fourth, ordering — webhooks may arrive out of order, so don't assume event sequence."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "I'll use WebSockets for everything" | Use the simplest pattern that works. REST for APIs, SSE for one-way streams, WebSockets only when bidirectional is truly needed. |
| Ignoring the stateful nature of WebSockets | WebSocket connections are tied to specific servers. Plan for: sticky sessions, a message broker for cross-server routing, and reconnection logic. |
| "Polling is bad, never use it" | Polling is perfectly fine for low-frequency, non-critical updates. It's stateless and simple. Don't over-engineer. |
| Not mentioning a fallback mechanism | Always have a fallback: "We use WebSockets with a long-polling fallback for clients behind restrictive firewalls." |
| Using polling for a chat application | Chat needs real-time, bidirectional communication. Polling would introduce unacceptable latency and waste bandwidth. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 2 CHEAT SHEET                         │
│                  Communication Patterns                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SYNC vs ASYNC:                                                  │
│  • Sync: Client blocks until response. Use for reads/queries.   │
│  • Async: Fire-and-forget via queue. Use for background tasks.  │
│  • Most systems use BOTH: sync for API, async for processing.   │
│                                                                  │
│  REAL-TIME PATTERNS:                                             │
│  • Short Polling: Simplest. Check every N seconds. Wasteful.    │
│  • Long Polling: Server holds connection. Better than polling.  │
│  • SSE: Server pushes events. One-way. Simple. HTTP-friendly.   │
│  • WebSocket: Full-duplex. Bidirectional. Complex. Stateful.    │
│                                                                  │
│  WHEN TO USE WHAT:                                               │
│  • Chat/gaming → WebSocket                                      │
│  • Live dashboard → SSE                                         │
│  • Order status → Polling (30s interval)                        │
│  • Notifications → SSE or WebSocket                             │
│  • Third-party events → Webhooks                                │
│                                                                  │
│  WEBHOOKS:                                                       │
│  • Server-to-server push via HTTP POST                          │
│  • Must handle: retries, verification, idempotency, ordering    │
│                                                                  │
│  SCALING WebSocket:                                              │
│  • Need sticky sessions (or connection-aware routing)            │
│  • Message broker (Redis Pub/Sub, Kafka) between WS servers     │
│  • Each server handles ~5K-65K concurrent connections            │
│  • Always implement heartbeat + reconnection                     │
│                                                                  │
│  PUSH vs PULL:                                                   │
│  • Pull = client asks. Simpler, staler. (Email checking)        │
│  • Push = server sends. Complex, real-time. (Chat messages)     │
│  • Hybrid = best of both. (Twitter's fan-out strategy)          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the key difference between sync and async communication? | Sync: client blocks until response. Async: client sends message and continues immediately, doesn't wait for result. |
| 2 | What are the 4 real-time communication patterns? | Short polling, long polling, Server-Sent Events (SSE), and WebSockets |
| 3 | When is short polling acceptable? | Low-frequency updates where seconds of delay are OK, small number of clients, when simplicity is valued |
| 4 | How does long polling differ from short polling? | In long polling, the server holds the connection open until new data is available or a timeout expires, instead of returning immediately |
| 5 | What makes WebSocket different from SSE? | WebSocket is full-duplex (both sides send/receive). SSE is one-way (server → client only). WebSocket needs special load balancer support; SSE works with standard HTTP. |
| 6 | Why are WebSockets harder to scale? | They're stateful — each client has a persistent connection to a specific server. Need sticky sessions and a message broker for cross-server communication. |
| 7 | What are the 4 challenges with webhooks? | Delivery reliability (retries needed), security (signature verification), idempotency (handle duplicates), ordering (events may arrive out of order) |
| 8 | What is the "hybrid push-pull" approach? | Use push for most cases but fall back to pull for special scenarios. Example: Twitter pushes tweets to most users' feeds but pulls (on-read) for celebrity followers. |
| 9 | What's the maximum concurrent WebSocket connections per server? | Typically 5K-65K depending on the server and message frequency. Limited by file descriptors, memory, and CPU for message processing. |
| 10 | When would you use SSE over WebSocket for a notification system? | When notifications are server → client only (user doesn't send notifications back). SSE is simpler, works through HTTP proxies, has built-in reconnection, and doesn't require special load balancer config. |

---

[← Previous: APIs — REST, GraphQL, gRPC](./chapter-01-apis-rest-graphql-grpc.md) · [Next: Scaling 101 →](../phase-1-scalability/chapter-03-scaling-101.md)
