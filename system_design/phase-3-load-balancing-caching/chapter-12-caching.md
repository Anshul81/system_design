# Chapter 12: Caching — Strategies & Patterns

> **Phase 3 — Load Balancing, Caching & Proxies** · Estimated Time: 3.5 hours · Difficulty: 🟡 Medium

[← Previous: Load Balancing](./chapter-11-load-balancing.md) · [Next: CDN & Edge Computing →](./chapter-13-cdn-edge.md)

---

## Learning Objectives

- ✅ Explain and compare 5 caching strategies: cache-aside, read-through, write-through, write-behind, write-around
- ✅ Choose the right eviction policy (LRU, LFU, TTL)
- ✅ Understand cache invalidation and why it's "the hardest problem in CS"
- ✅ Compare local vs distributed caching (Redis vs Memcached)
- ✅ Design a caching layer for a system design interview

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapters 6-7 (Databases) | ✅ Yes |
| Chapter 11 (Load Balancing) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 2.5 hrs |
| Strategy exercises | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** A cache is like your desk vs a filing cabinet. Your desk (cache) is small but everything on it is instantly accessible. The filing cabinet (database) holds everything but requires you to walk across the room. You keep the documents you need most frequently on your desk.

**Interview relevance:** Caching reduces database load, decreases latency, and is expected in every system design. The interviewer will test whether you understand: which strategy to use, where to place the cache, how to handle invalidation, and what eviction policy to choose. "Cache invalidation" is the most commonly asked follow-up question.

---

## Visual Diagram — Cache in Architecture

```
┌────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│ Client  │────▶│ Load Balancer │────▶│ App Server   │────▶│ Database  │
└────────┘     └──────────────┘     └──────┬──────┘     └──────────┘
                                           │
                                    ┌──────▼──────┐
                                    │   Cache      │
                                    │   (Redis)    │
                                    │              │
                                    │  Hit: <1ms   │
                                    │  Miss: query │
                                    │  DB (~10ms)  │
                                    └─────────────┘

Cache hit ratio target: 90-99%
  90% hit rate: DB gets 10% of traffic (10x reduction!)
  99% hit rate: DB gets 1% of traffic (100x reduction!)
```

---

## Core Concepts

### 1. Caching Strategies

#### Cache-Aside (Lazy Loading) — Most Common

The application manages the cache directly. On read, check cache first. On miss, read from DB and populate cache.

```
READ:
  App ──── GET key ────▶ Cache
  App ◀─── HIT: value ── Cache    → Return to client

  App ──── GET key ────▶ Cache
  App ◀─── MISS ──────── Cache
  App ──── SELECT * ───▶ DB
  App ◀─── Result ────── DB
  App ──── SET key ────▶ Cache    → Populate cache
  App ◀─── OK ────────── Cache   → Return to client

WRITE:
  App ──── UPDATE ─────▶ DB       → Write to DB
  App ──── DELETE key ─▶ Cache    → Invalidate cache
```

| Pros | Cons |
|------|------|
| Simple and widely used | Cache miss = 3 calls (cache + DB + cache write) |
| Only caches data that's actually requested | Data can become stale (if DB updated without cache invalidation) |
| Cache failure doesn't break the system | Initial "cold start" = all requests hit DB |

**Interview default:** Use cache-aside unless you have a specific reason for another strategy.

#### Read-Through

Cache sits between app and DB. Cache automatically fetches from DB on miss.

```
App ──── GET key ────▶ Cache ──── (miss) ────▶ DB
App ◀─── value ────── Cache ◀─── value ────── DB
```

| Pros | Cons |
|------|------|
| App code is simpler (just talks to cache) | Cache library must support read-through |
| Cache handles the DB interaction | Less control over what gets cached |

#### Write-Through

Every write goes to cache AND database synchronously.

```
App ──── WRITE ────▶ Cache ──── WRITE ────▶ DB
App ◀─── OK ─────── Cache ◀─── OK ─────── DB
(Both cache and DB are always in sync)
```

| Pros | Cons |
|------|------|
| Cache is always up-to-date | Higher write latency (2 writes per operation) |
| No stale data | Caches data that may never be read (wasteful) |

#### Write-Behind (Write-Back)

Write to cache immediately, asynchronously write to DB later.

```
App ──── WRITE ────▶ Cache ──── OK ────▶ App (fast!)
                       │
                       │ (async, batched)
                       ▼
                       DB
```

| Pros | Cons |
|------|------|
| Very fast writes (cache speed) | Risk of data loss if cache crashes before DB write |
| Can batch DB writes (reduces DB load) | Complex implementation |

#### Write-Around

Write directly to DB, bypassing cache. Cache only populated on reads.

```
App ──── WRITE ────▶ DB (cache not updated)
App ──── READ ─────▶ Cache (miss) → DB → Cache
```

| Pros | Cons |
|------|------|
| Doesn't pollute cache with write-heavy data | First read after write is always a cache miss |
| Good for data written once but rarely read | Higher read latency for recently written data |

### Strategy Comparison

| Strategy | Write Latency | Read Latency | Consistency | Best For |
|----------|--------------|-------------|-------------|----------|
| **Cache-Aside** | Low (DB only) | Miss: High | Eventual | General-purpose (default) |
| **Read-Through** | Low (DB only) | Miss: High | Eventual | Simplified app code |
| **Write-Through** | High (cache+DB) | Always Low | Strong | Read-heavy with consistency needs |
| **Write-Behind** | Very Low (cache) | Always Low | Weak | Write-heavy, latency-sensitive |
| **Write-Around** | Low (DB only) | Miss after write | Eventual | Write-once-read-rarely data |

---

### 2. Cache Eviction Policies

When the cache is full, which item do you remove?

| Policy | How It Works | Best For |
|--------|-------------|----------|
| **LRU** (Least Recently Used) | Evict the item not accessed for the longest time | General-purpose (most common) |
| **LFU** (Least Frequently Used) | Evict the item with the fewest total accesses | Data with stable popularity |
| **FIFO** (First In First Out) | Evict the oldest item | Simple, time-based |
| **TTL** (Time To Live) | Item expires after N seconds regardless of access | Time-sensitive data |
| **Random** | Evict a random item | Simple, surprisingly OK at scale |

**Interview default:** "I'd use LRU eviction with a TTL as a safety net. LRU ensures hot data stays cached, and TTL prevents indefinitely stale data."

---

### 3. Cache Invalidation — "The Hardest Problem"

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**The problem:** When data in the DB changes, the cache has stale data. How do you keep them in sync?

```
SCENARIO: User updates their profile name

Without invalidation:
  DB: name = "John Smith" (updated)
  Cache: name = "John Doe" (STALE!)
  Users see the old name until cache expires.

Strategy 1: DELETE on write (recommended)
  App updates DB → App deletes cache key
  Next read → cache miss → reads fresh data from DB → populates cache

Strategy 2: UPDATE on write
  App updates DB → App updates cache with new value
  Risk: Race condition if two updates happen simultaneously

Strategy 3: TTL expiration
  Set TTL = 60 seconds
  Cache automatically expires → next read gets fresh data
  Simple but stale for up to 60 seconds
```

**Race condition with cache invalidation:**
```
Thread A: UPDATE user SET name='Alice' → DELETE cache
Thread B: UPDATE user SET name='Bob' → DELETE cache

What if:
  1. Thread A updates DB to 'Alice'
  2. Thread B updates DB to 'Bob'
  3. Thread A reads from DB and caches 'Bob' (Thread B already updated)
  4. Thread B deletes cache
  Wait... now the cache has no data but DB has 'Bob' — OK.

  BUT with UPDATE strategy:
  1. Thread A updates DB to 'Alice'
  2. Thread B updates DB to 'Bob'
  3. Thread B updates cache to 'Bob'
  4. Thread A updates cache to 'Alice' (WRONG! DB has 'Bob')
  → Cache has 'Alice', DB has 'Bob' — INCONSISTENT!

This is why DELETE is safer than UPDATE for cache invalidation.
```

**Interview tip:** Always use DELETE (not UPDATE) for cache invalidation. Mention TTL as a safety net: "Even if invalidation fails, the data expires after 60 seconds."

---

### 4. Local vs Distributed Cache

```
LOCAL CACHE (in-memory, per server):
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ App Server 1   │  │ App Server 2   │  │ App Server 3   │
│ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │
│ │Local Cache│ │  │ │Local Cache│ │  │ │Local Cache│ │
│ │key:A → v1 │ │  │ │key:A → v1 │ │  │ │key:A → v2!│ │
│ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │
└───────────────┘  └───────────────┘  └───────────────┘
Problem: Inconsistency! Server 3 has a different version.

DISTRIBUTED CACHE (shared, external):
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ App Server 1   │  │ App Server 2   │  │ App Server 3   │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                    ┌──────▼──────┐
                    │ Redis Cluster│  Single source of truth
                    │ key:A → v1  │  for cached data
                    └─────────────┘
```

| Feature | Local Cache | Distributed Cache (Redis) |
|---------|-------------|--------------------------|
| **Latency** | ~100ns (in-process) | ~1ms (network hop) |
| **Consistency** | Inconsistent across servers | Consistent (single source) |
| **Size** | Limited by server RAM | Can be very large (cluster) |
| **Failure** | Lost on server restart | Persists across app restarts |
| **Best for** | Very hot data, config, rarely-changing | Sessions, user data, feed caches |

**Interview tip:** "I'd use a distributed cache (Redis) as the primary cache for consistency, and optionally a small local cache for extremely hot, rarely-changing data like configuration settings."

### 5. Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Strings, hashes, lists, sets, sorted sets | Strings only |
| Persistence | Yes (RDB snapshots, AOF log) | No (pure memory) |
| Replication | Yes (leader-follower) | No |
| Clustering | Yes (Redis Cluster) | Client-side sharding |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Threading | Single-threaded (mostly) | Multi-threaded |
| **Best for** | Feature-rich caching, sessions, queues | Simple, high-throughput caching |

**Interview default:** Choose Redis. It does everything Memcached does and more. Memcached only wins on pure get/set throughput for very simple caching.

---

## Real-World Example — Facebook's Caching Architecture

```
Facebook uses a multi-layer caching strategy:

┌─────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────┐
│ Client   │────▶│  CDN          │────▶│ Web Tier  │────▶│ Cache    │
│ (Browser)│     │  (Static)     │     │           │     │ (Memcached│
└─────────┘     └──────────────┘     └──────────┘     │  clusters)│
                                                        └────┬─────┘
                                                             │ miss
                                                        ┌────▼─────┐
                                                        │  Database │
                                                        │  (MySQL   │
                                                        │   + TAO)  │
                                                        └──────────┘

Facebook's TAO (The Associations and Objects):
  • Custom distributed cache for social graph data
  • Cache-aside pattern with MySQL backend
  • Handles billions of reads/sec across clusters
  • Leader-follower topology: writes to leader cache,
    reads from follower caches
  • Eventual consistency (OK for social data)

Key numbers:
  • Memcached: ~1000 servers, many TB of cached data
  • Cache hit rate: >99%
  • Latency: <1ms for cache hits
```

---

## Common Interview Questions

### Q1: "How would you design a caching strategy for a Twitter timeline?"

**Model answer:**
"I'd use cache-aside with Redis. When a user requests their timeline, I check Redis first. On a cache hit, return the cached timeline (sub-millisecond). On a miss, query the database to build the timeline, store it in Redis with a 5-minute TTL, and return it. When a new tweet is posted by someone the user follows, I'd invalidate (delete) the user's cached timeline so the next request rebuilds it with the new tweet. For celebrity accounts, I wouldn't cache individual timelines but compute them on-read to avoid massive fan-out invalidation. Cache key: `timeline:{user_id}`. Eviction: LRU with 5-minute TTL."

### Q2: "What is the thundering herd problem?"

**Model answer:**
"The thundering herd happens when a popular cache key expires and hundreds or thousands of concurrent requests all get a cache miss simultaneously. They all query the database at once, overwhelming it. Solutions: first, use a mutex/lock — only one request rebuilds the cache, others wait. Second, use 'stale-while-revalidate' — serve the stale cached value while one request refreshes it in the background. Third, set random TTL variation to prevent many keys from expiring at the same time."

### Q3: "Cache-aside vs write-through — when would you use each?"

**Model answer:**
"Cache-aside is my default — it's simple, only caches data that's actually accessed, and a cache failure doesn't break the system. Write-through is better when reads vastly outnumber writes AND I need strong consistency between cache and database — like a product catalog where I can't show stale prices. The trade-off is write-through has higher write latency (writing to both cache and DB) and may cache data that's never read."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "Cache everything" | Cache hot data only. Use 80/20 rule — cache the 20% of data that gets 80% of reads. |
| Using UPDATE for cache invalidation | Use DELETE. UPDATE has race condition risks. Let the next read populate fresh data. |
| No TTL on cache entries | Always set a TTL as a safety net, even if you have explicit invalidation. |
| Ignoring thundering herd | For popular keys, use mutex locking or stale-while-revalidate to prevent DB storms. |
| "Redis is a database" | Redis is primarily a cache. It can persist data, but for critical data, always have a durable primary database (PostgreSQL, MySQL). |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 12 CHEAT SHEET                        │
│                  Caching Strategies                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STRATEGIES:                                                     │
│  • Cache-Aside: App checks cache → miss → DB → populate cache  │
│  • Read-Through: Cache auto-fetches from DB on miss             │
│  • Write-Through: Write to cache AND DB synchronously           │
│  • Write-Behind: Write to cache, async to DB (fast but risky)  │
│  • Write-Around: Write to DB only, cache populated on read      │
│                                                                  │
│  DEFAULT: Cache-Aside + LRU + TTL                               │
│                                                                  │
│  INVALIDATION:                                                   │
│  • DELETE on write (not UPDATE — avoids race conditions)        │
│  • TTL as safety net (30s-5min depending on staleness tolerance)│
│                                                                  │
│  EVICTION: LRU (default), LFU (stable popularity), TTL (time)  │
│                                                                  │
│  PROBLEMS TO KNOW:                                               │
│  • Thundering herd: mutex lock or stale-while-revalidate        │
│  • Cache penetration: cache "null" results for non-existent keys│
│  • Cold start: pre-warm cache on deployment                     │
│                                                                  │
│  Redis: 100K+ ops/sec, rich data structures, persistence        │
│  Memcached: simpler, multi-threaded, pure caching               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is cache-aside? | App checks cache first. On miss, reads from DB, then populates cache. App manages the cache directly. |
| 2 | Why use DELETE instead of UPDATE for cache invalidation? | UPDATE has race conditions — concurrent writes can leave stale data in cache. DELETE is safe: next read rebuilds with fresh data. |
| 3 | What is the thundering herd problem? | A popular cache key expires, causing many concurrent requests to miss cache and hit DB simultaneously, overwhelming it. |
| 4 | LRU vs LFU — when to use each? | LRU: general-purpose, evicts least recently accessed (good for recency). LFU: evicts least frequently accessed (good for stable popularity). |
| 5 | What is cache penetration? | Repeated requests for non-existent data bypass the cache every time. Fix: cache "null" results with a short TTL. |
| 6 | Local cache vs distributed cache? | Local: fastest (~100ns) but inconsistent across servers. Distributed (Redis): consistent, shared, ~1ms latency. |
| 7 | What is write-behind caching? | Write to cache immediately, asynchronously write to DB later. Fast writes but risk data loss if cache crashes. |
| 8 | What hit rate should a well-designed cache achieve? | 90-99%. A 99% hit rate means the DB gets 100x less traffic. |
| 9 | Redis vs Memcached? | Redis: data structures, persistence, replication, pub/sub. Memcached: simpler, multi-threaded, pure caching. Default: Redis. |
| 10 | What TTL would you set for a Twitter timeline cache? | 1-5 minutes. Balances freshness (new tweets appear) with performance (reduce DB load). |

---

[← Previous: Load Balancing](./chapter-11-load-balancing.md) · [Next: CDN & Edge Computing →](./chapter-13-cdn-edge.md)
