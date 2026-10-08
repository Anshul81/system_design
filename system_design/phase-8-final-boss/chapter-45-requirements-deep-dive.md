# Chapter 45: Functional vs Non-Functional Requirements — Deep Dive

> **Phase 8 — Interview Strategy** · Estimated Time: 2 hours · Difficulty: 🟢 Beginner · Type: 📋 STRATEGY

[← Previous: Interview Framework](./chapter-44-interview-framework-detailed.md) · [Next: Trade-off Analysis →](./chapter-46-tradeoff-analysis.md)

---

## Why This Chapter Matters

Requirements define what you're building. Getting them wrong means building the wrong system — even if the architecture is technically sound. This chapter teaches you to systematically extract requirements and translate them into architectural decisions.

---

## Functional Requirements — What the System Does

### How to Extract Them

```
TECHNIQUE: Start with user actions (verbs)

"Design YouTube"
  Users can:
    → Upload videos
    → Watch/stream videos
    → Search for videos
    → Like, comment, subscribe
    → Get recommendations

  ❌ DON'T list 20 features. Scope to 3-5 CORE features.
  ✅ DO ask: "Should I focus on upload + streaming? Or also 
     include comments and recommendations?"

PRIORITIZE with the interviewer:
  "The core flow is upload → process → stream. 
   I'll design that first, then add search and social if time allows."
```

### Templates by Problem Type

```
SOCIAL MEDIA (Twitter, Instagram):
  • Post content (text/image/video)
  • View feed (timeline from followed users)
  • Follow/unfollow
  • Like/comment
  • Search

MESSAGING (WhatsApp, Slack):
  • Send/receive messages (1-on-1, group)
  • Online presence
  • Read receipts
  • Push notifications
  • Media sharing

STORAGE/SYNC (Dropbox, Google Drive):
  • Upload/download files
  • Sync across devices
  • Share with other users
  • Version history

MARKETPLACE (Amazon, Uber):
  • Browse/search catalog
  • Place order / request ride
  • Payment processing
  • Real-time tracking
  • Reviews/ratings

INFRASTRUCTURE (Rate Limiter, Cache, KV Store):
  • Core operations (GET/PUT/DELETE)
  • Configuration (rules, policies)
  • Monitoring/dashboards
```

---

## Non-Functional Requirements — How the System Behaves

### The 8 Key Non-Functional Requirements

```
┌────────────────────┬──────────────────────────────────────────────┐
│ Requirement         │ What to discuss                              │
├────────────────────┼──────────────────────────────────────────────┤
│ 1. SCALE            │ DAU, QPS, data volume                        │
│                    │ "How many users? How many requests/sec?"     │
├────────────────────┼──────────────────────────────────────────────┤
│ 2. LATENCY          │ Response time targets                        │
│                    │ "Reads < 200ms? Writes < 500ms?"            │
├────────────────────┼──────────────────────────────────────────────┤
│ 3. AVAILABILITY     │ Uptime target                                │
│                    │ "99.9% (8.7 hr/yr downtime) or 99.99%?"     │
├────────────────────┼──────────────────────────────────────────────┤
│ 4. CONSISTENCY      │ Strong vs eventual                           │
│                    │ "Is it OK if a like count is stale by 5s?"  │
├────────────────────┼──────────────────────────────────────────────┤
│ 5. DURABILITY       │ Data loss tolerance                          │
│                    │ "Can we ever lose a message? A payment?"     │
├────────────────────┼──────────────────────────────────────────────┤
│ 6. THROUGHPUT       │ Operations per second                        │
│                    │ "Read QPS vs write QPS?"                     │
├────────────────────┼──────────────────────────────────────────────┤
│ 7. STORAGE          │ Total data, retention, growth                │
│                    │ "How much data? How long to keep?"           │
├────────────────────┼──────────────────────────────────────────────┤
│ 8. GLOBAL           │ Multi-region, user geography                 │
│                    │ "Global users or single region?"             │
└────────────────────┴──────────────────────────────────────────────┘
```

### Which Non-Functionals Matter Most?

```
NOT ALL 8 ARE EQUALLY IMPORTANT FOR EVERY PROBLEM.

Focus on 3-4 that are CRITICAL for the specific problem:

URL Shortener:     Scale (100M URLs), Latency (< 100ms redirect), Availability
Twitter Feed:      Scale (250M DAU), Latency (< 200ms), Read:Write ratio
Chat System:       Latency (< 100ms), Ordering, Durability (no lost messages)
Payment System:    Consistency (STRONG), Durability, Correctness
Video Streaming:   Bandwidth, Global CDN, Storage
Ticket Booking:    Consistency (no overselling), Peak traffic handling
Rate Limiter:      Latency (< 1ms), Throughput, Accuracy

PATTERN: The non-functional requirements drive your architecture.
  • High read:write ratio → caching + pre-computation
  • Low latency requirement → CDN, Redis, denormalization
  • Strong consistency → ACID database, synchronous replication
  • High availability → redundancy, eventual consistency
  • High write throughput → write-optimized DB (LSM), message queue buffering
```

---

## Requirements → Architecture Mapping

```
REQUIREMENTS DIRECTLY DRIVE TECHNOLOGY CHOICES:

Requirement                      → Architecture Decision
────────────────────────         ──────────────────────────
Low latency reads (< 10ms)       → Redis cache, CDN, denormalization
High write throughput             → Kafka buffer, LSM-based DB (Cassandra)
Strong consistency                → MySQL/PostgreSQL, synchronous replication
High availability (99.99%)        → Multi-region, no single points of failure
Large files (video, images)       → S3 + CDN, pre-signed URLs
Real-time updates                 → WebSocket, Server-Sent Events
Full-text search                  → Elasticsearch
Geospatial queries                → Redis GEO, GeoHash, QuadTree
High read:write ratio (100:1)     → Heavy caching, read replicas
Exactly-once processing           → Idempotency keys, dedup
Global users                      → Multi-region deploy, CDN, geo-routing
```

### Example: Translating Requirements to Design

```
PROBLEM: "Design a URL Shortener"

REQUIREMENTS I GATHER:
  FR: Shorten URL, redirect, analytics
  NFR: 100M URLs/day, 100:1 read:write, < 100ms redirect, 99.99% uptime

MY REASONING (SAY THIS OUT LOUD):
  "100:1 read:write → this is read-heavy → I need aggressive caching.
   < 100ms → Redis cache for hot URLs (sub-ms reads).
   99.99% → need redundancy everywhere, no SPOF.
   100M URLs/day → 1,200 write QPS → single MySQL can handle this.
   Read QPS 120K → mostly served from cache, DB sees ~12K (misses)."

ARCHITECTURE DECISIONS (flow naturally from requirements):
  ✅ Redis cache (read-heavy, low latency)
  ✅ MySQL primary + replicas (moderate writes, need ACID for dedup)
  ✅ Stateless app servers behind LB (availability, horizontal scaling)
  ✅ No complex distributed DB needed (single MySQL handles write QPS)

Every decision justified by a specific requirement. THIS is what 
interviewers want to see.
```

---

## Anti-Patterns in Requirement Gathering

```
❌ NOT SCOPING:
   "Let me design ALL of YouTube including live streaming, 
    monetization, content moderation, creator studio..."
   → You'll spend 40 minutes and cover nothing well.

✅ FIX: "I'll focus on upload + streaming. I'll mention 
   search and recommendations at a high level."

❌ MAKING UP REQUIREMENTS:
   "I'll assume we need blockchain and machine learning."
   → Don't add complexity that isn't needed.

✅ FIX: Ask the interviewer. "Do we need ML-based 
   recommendations, or is chronological feed sufficient?"

❌ IGNORING NON-FUNCTIONALS:
   "I'll use MySQL" (without knowing the scale)
   → MySQL might work at 1K QPS but not at 1M QPS.

✅ FIX: Always establish scale before choosing technology.

❌ OVER-ENGINEERING:
   "We need 5 microservices, Kafka, Redis, Cassandra, 
    Elasticsearch, and a service mesh."
   → For a URL shortener at 1K write QPS, this is absurd.

✅ FIX: Let the requirements drive complexity. 
   Simple requirements → simple architecture.
```

---

## Availability Numbers Reference

```
  AVAILABILITY    DOWNTIME PER YEAR    DOWNTIME PER MONTH
  ────────────    ────────────────    ──────────────────
  99%             3.65 days           7.3 hours
  99.9%           8.76 hours          43.8 minutes
  99.99%          52.6 minutes        4.4 minutes
  99.999%         5.26 minutes        26 seconds

  Most interviews: assume 99.99% (four nines)
  Payment systems: often 99.999% (five nines)
  Internal tools: 99.9% may be acceptable
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the first thing to do in a system design interview? | Ask clarifying questions to establish functional and non-functional requirements. NEVER jump straight into design. |
| 2 | How many functional requirements should you focus on? | 3-5 core features. Scope with the interviewer: "Should I focus on X and Y, or also include Z?" |
| 3 | Name the 8 key non-functional requirements. | Scale, Latency, Availability, Consistency, Durability, Throughput, Storage, Global reach. Focus on 3-4 most critical for the specific problem. |
| 4 | How do requirements drive architecture? | Directly. High read:write → caching. Low latency → Redis/CDN. Strong consistency → ACID DB. High writes → Kafka + LSM DB. |
| 5 | What does 99.99% availability mean? | 52.6 minutes of downtime per year, or 4.4 minutes per month. Requires redundancy at every layer. |
| 6 | What's the biggest anti-pattern in requirements? | Not scoping. Trying to design everything leads to shallow coverage of each component. Focus on core flows, mention others briefly. |

---

[← Previous: Interview Framework](./chapter-44-interview-framework-detailed.md) · [Next: Trade-off Analysis →](./chapter-46-tradeoff-analysis.md)
