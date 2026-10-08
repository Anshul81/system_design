# Chapter 19: Rate Limiting

> **Phase 5 — Distributed Systems Concepts** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium

[← Previous: Monitoring & Logging](../phase-4-infrastructure/chapter-18-monitoring-logging.md) · [Next: Unique ID Generation →](./chapter-20-unique-id-generation.md)

---

## Learning Objectives

- ✅ Compare rate limiting algorithms: token bucket, leaky bucket, fixed window, sliding window
- ✅ Design a distributed rate limiter using Redis
- ✅ Place rate limiting at the right layer (API gateway, application, per-service)
- ✅ Handle rate limit responses correctly (429, Retry-After header)

---

## Introduction — Why This Matters

**Interview relevance:** "Design a rate limiter" is a standalone interview question AND a component you should mention in every design. It protects your system from abuse, DDoS attacks, and misbehaving clients. Interviewers test whether you understand the algorithms and can implement one in a distributed setting with multiple servers.

---

## Core Concepts

### 1. Rate Limiting Algorithms

#### Token Bucket (Most Common)

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

Time 0: [●●●●●●●●●●] (10 tokens, full)
  Request → consume 1 token → ✅ Allow
  Request → consume 1 token → ✅ Allow

Time 1: [●●●●●●●●●●] (8 + 2 refill = 10, capped at capacity)

Burst of 10 requests:
  [●●●●●●●●●●] → [          ] (all 10 consumed → all allowed)

Time 2: [●●          ] (0 + 2 refill = 2 tokens)
  Request → ✅ Allow (1 token left)
  Request → ✅ Allow (0 tokens)
  Request → ❌ REJECT (429 Too Many Requests)
```

| Pros | Cons |
|------|------|
| Allows bursts up to bucket size | Need to track per-user token count |
| Smooth rate limiting over time | Slightly complex implementation |
| Memory efficient | — |

#### Sliding Window Log

```
Window: 60 seconds, Limit: 100 requests

Maintain a sorted list of request timestamps per user.

Request at T=65s:
  Remove all timestamps < (65 - 60) = 5s
  Count remaining timestamps
  If count < 100 → Allow, add timestamp
  If count >= 100 → Reject

Timestamps: [5.1, 5.3, 10.2, 15.5, ..., 64.8]
  Count = 98 → Allow → Add 65.0
```

| Pros | Cons |
|------|------|
| Most accurate | High memory (stores all timestamps) |
| No boundary issues | Expensive to maintain sorted list |

#### Sliding Window Counter (Practical)

```
Combine fixed window counts with weighting:

Previous window (0-60s): 84 requests
Current window (60-120s): 36 requests
Current time: T=75s (25% into current window)

Weighted count = prev × (1 - 0.25) + curr × 1.0
              = 84 × 0.75 + 36 × 1.0
              = 63 + 36 = 99

If limit = 100 → Allow (99 < 100)
```

| Pros | Cons |
|------|------|
| Memory efficient (2 counters per window) | Approximate (not exact) |
| No boundary spike | Good enough for most use cases |

### 2. Distributed Rate Limiting with Redis

```
Multiple app servers, single Redis instance:

┌────────┐  ┌────────┐  ┌────────┐
│Server 1 │  │Server 2 │  │Server 3 │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
           ┌──────▼──────┐
           │    Redis     │
           │              │
           │ user:123 → 7 │  (7 tokens remaining)
           │ TTL: 60s     │
           └─────────────┘

Redis commands (Token Bucket):
  MULTI
    DECR rate_limit:user:123     # Decrement token count
    GET rate_limit:user:123      # Check remaining
  EXEC

  If remaining >= 0 → Allow
  If remaining < 0 → INCR (restore) → Reject (429)
```

### 3. Rate Limit Response

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1634567890

{
  "error": "Rate limit exceeded",
  "message": "Try again in 30 seconds"
}
```

### 4. Where to Apply Rate Limiting

| Layer | What It Protects | Example |
|-------|-----------------|---------|
| **API Gateway** | Entire system from external abuse | 1000 req/min per API key |
| **Per-Service** | Individual service from overload | 100 req/min per user per service |
| **Per-User** | Fair usage across users | 60 tweets/hour per user |
| **Per-IP** | DDoS protection | 100 req/min per IP |

---

## Common Interview Questions

### Q1: "Design a distributed rate limiter."

**Model answer:**
"I'd use the token bucket algorithm with Redis as the centralized counter. Each API key or user ID gets a Redis key with a token count and TTL. When a request arrives, the API gateway atomically decrements the token count in Redis using a Lua script (atomic operation). If tokens remain, the request proceeds. If not, return 429 with a Retry-After header. The token count is refilled by setting a TTL on the key — when it expires, the next request creates a fresh bucket. For sliding window, I'd use two Redis counters (current and previous window) with weighted averaging."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 19 CHEAT SHEET                        │
│                  Rate Limiting                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ALGORITHMS:                                                     │
│  Token Bucket: allows bursts, smooth long-term rate (DEFAULT)   │
│  Sliding Window Counter: accurate, memory-efficient              │
│  Fixed Window: simplest, but boundary spike problem              │
│                                                                  │
│  DISTRIBUTED: Use Redis for centralized counters                │
│  ATOMIC OPS: Redis Lua scripts for race-free operations         │
│  RESPONSE: 429 + Retry-After header + rate limit headers        │
│                                                                  │
│  APPLY AT: API Gateway (global), per-service, per-user, per-IP │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the token bucket algorithm? | A bucket with fixed capacity fills at a constant rate. Each request consumes a token. If no tokens left, request is rejected. Allows bursts up to bucket size. |
| 2 | How do you implement distributed rate limiting? | Use Redis as a centralized counter. Atomic Lua scripts to check/decrement token count. All API servers share the same Redis state. |
| 3 | What HTTP status code for rate limiting? | 429 Too Many Requests, with Retry-After header and X-RateLimit-* headers. |
| 4 | What is the fixed window boundary problem? | At the boundary of two windows, a user can make 2x the limit (max at end of window 1 + max at start of window 2). Sliding window fixes this. |
| 5 | Where should rate limiting be applied? | At the API Gateway for global protection, and optionally per-service for fine-grained control. |

---

[← Previous: Monitoring & Logging](../phase-4-infrastructure/chapter-18-monitoring-logging.md) · [Next: Unique ID Generation →](./chapter-20-unique-id-generation.md)
