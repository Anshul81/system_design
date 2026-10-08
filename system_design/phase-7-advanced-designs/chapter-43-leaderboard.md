# Chapter 43: Design a Leaderboard / Ranking System

> **Phase 7 — Advanced Design Problems** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium · Type: 🏗️ DESIGN

[← Previous: Key-Value Store](./chapter-42-key-value-store.md) · [Next: Interview Framework →](../phase-8-final-boss/chapter-44-interview-framework-detailed.md)

---

## Why This Problem Matters

Leaderboards appear in gaming, competitive platforms, and social apps. The core challenge: maintaining a sorted ranking of millions of users that updates in real-time. Naively sorting millions of rows on every query is prohibitively expensive. The key insight is using Redis Sorted Sets for O(log N) rank operations.

---

## Step 1: Requirements

### Functional
| Feature | Description |
|---------|-------------|
| **Update score** | User scores update in real-time |
| **Get rank** | "What rank am I?" — instant response |
| **Get top N** | Fetch top 10, top 100, etc. |
| **Get surrounding** | "Show 5 users above and below me" |
| **Multiple leaderboards** | Daily, weekly, all-time, per-game |
| **Historical** | View past leaderboards (e.g., last week's winner) |

### Non-Functional
| Requirement | Target |
|-------------|--------|
| **Users** | 50M scored users |
| **Score updates/sec** | 50K |
| **Rank query latency** | < 10ms |
| **Top-N query latency** | < 10ms |
| **Availability** | 99.99% |

---

## Step 2: Estimation

```
DATA SIZE:
  50M users × (user_id: 8B + score: 8B) = 800 MB
  → Fits entirely in a single Redis instance (trivial!)

THROUGHPUT:
  50K score updates/sec → Redis handles 100K+ ops/sec easily
  Rank queries: ~200K/sec → Redis handles this

MULTIPLE LEADERBOARDS:
  Per-game: 100 games × 800 MB = 80 GB
  Daily/weekly resets: keep snapshots → S3 for historical
```

---

## Step 3: High-Level Design

### Architecture

```
┌────────┐    ┌────────────┐    ┌──────────────────────────┐
│ Client  │───▶│ API Gateway │───▶│ Leaderboard Service      │
│ (game)  │    └────────────┘    │                           │
└────────┘                      │ • Update score            │
                                 │ • Get rank               │
                                 │ • Get top N              │
                                 │ • Get surrounding        │
                                 └──────────┬───────────────┘
                                            │
                                 ┌──────────▼───────────────┐
                                 │  Redis (Sorted Sets)      │
                                 │                           │
                                 │  Key: leaderboard:{game}  │
                                 │  Members: user_ids        │
                                 │  Scores: user scores      │
                                 │                           │
                                 │  ZADD, ZRANK, ZREVRANGE  │
                                 └──────────┬───────────────┘
                                            │
                                 ┌──────────▼───────────────┐
                                 │  MySQL (user profiles,    │
                                 │  historical leaderboards, │
                                 │  audit log)               │
                                 └──────────────────────────┘
```

### API Design

```
POST /api/v1/leaderboard/{board_id}/score
  Request:  { "user_id": "u_123", "score": 2500 }
  Response: { "rank": 42, "total_players": 1200000 }

GET /api/v1/leaderboard/{board_id}/top?limit=10
  Response: { "entries": [
    { "rank": 1, "user_id": "u_789", "username": "ProGamer", "score": 9999 },
    { "rank": 2, "user_id": "u_456", "username": "StarPlayer", "score": 9800 },
    ...
  ] }

GET /api/v1/leaderboard/{board_id}/rank/{user_id}
  Response: { "user_id": "u_123", "rank": 42, "score": 2500 }

GET /api/v1/leaderboard/{board_id}/around/{user_id}?range=5
  Response: { "entries": [
    { "rank": 37, "user_id": "u_111", "score": 2700 },
    ...
    { "rank": 42, "user_id": "u_123", "score": 2500, "is_self": true },
    ...
    { "rank": 47, "user_id": "u_222", "score": 2300 },
  ] }
```

---

## Step 4: Detailed Design

### Deep Dive 1: Redis Sorted Set Operations

```
Redis Sorted Set: ordered collection where each member has a score.
Internally implemented as a SKIP LIST + HASH TABLE.
All operations: O(log N)

CORE OPERATIONS:

Update score:
  ZADD leaderboard:game_1 2500 "user:123"
  → O(log N) — inserts or updates the member's score

Get rank:
  ZREVRANK leaderboard:game_1 "user:123"
  → Returns: 41 (0-indexed, highest score first)
  → O(log N)

Get top 10:
  ZREVRANGE leaderboard:game_1 0 9 WITHSCORES
  → Returns top 10 users with scores, highest first
  → O(log N + K) where K = number of results

Get surrounding users (rank 37-47):
  ZREVRANGE leaderboard:game_1 36 46 WITHSCORES
  → Returns 11 entries around the user
  → O(log N + K)

Increment score:
  ZINCRBY leaderboard:game_1 100 "user:123"
  → Atomically adds 100 to user's score
  → O(log N)

Count total players:
  ZCARD leaderboard:game_1
  → Returns total members
  → O(1)

SKIP LIST VISUALIZATION:
  Level 4:  H ──────────────────────────── 50 ────── T
  Level 3:  H ────────── 20 ────────────── 50 ────── T
  Level 2:  H ──── 10 ── 20 ──── 35 ───── 50 ────── T
  Level 1:  H ─ 5─ 10 ── 20 ─ 25─ 35 ─ 42─ 50 ─ 60─ T

  Lookup/insert/delete: O(log N) average
  Range queries: O(log N + K) — jump to start, walk K nodes
```

### Deep Dive 2: Multiple Leaderboards & Time-Based Resets

```
MULTIPLE LEADERBOARD STRATEGY:

Different Redis keys for different scopes:
  leaderboard:game_1:alltime     → all-time leaderboard
  leaderboard:game_1:weekly:2024-W01  → week 1 of 2024
  leaderboard:game_1:daily:2024-01-15  → January 15

DAILY/WEEKLY RESET:
  Option A: Delete old key, start fresh
    At midnight: DEL leaderboard:game_1:daily:2024-01-14
    New key auto-created on first ZADD

  Option B: Snapshot + archive
    1. Before reset: dump leaderboard to MySQL for history
       INSERT INTO leaderboard_history 
         SELECT * FROM redis_sorted_set_export
    2. Delete Redis key
    3. Historical queries served from MySQL

MULTIPLE GAMES:
  100 games × 50M users each?
  → Not all users play all games
  → Realistic: 100 games × avg 500K active players = 50M sorted set entries
  → Total: ~4 GB in Redis → single instance or small cluster

COMPOSITE SCORES:
  Problem: two users have same score → who ranks higher?
  Solution: encode timestamp in score for tiebreaking

  score = (actual_score * 10^10) + (MAX_TIMESTAMP - unix_timestamp)
  
  Higher score wins. For same score, earlier achiever wins.
  Redis sorts by this composite value → deterministic ordering.
```

### Deep Dive 3: Scaling Beyond a Single Redis Instance

```
WHAT IF 50M USERS IN ONE SORTED SET ISN'T ENOUGH?

Problem: 500M users globally, or 1B score updates/sec

OPTION 1: Shard by user_id range
  Shard 1: users A-M → Redis instance 1
  Shard 2: users N-Z → Redis instance 2
  
  Problem: Can't get global rank across shards easily.
  "What's my global rank?" requires querying all shards.

OPTION 2: Shard by score range
  Shard 1: scores 0-999 → Redis instance 1
  Shard 2: scores 1000-4999 → Redis instance 2
  Shard 3: scores 5000+ → Redis instance 3

  Global rank = count of users in higher shards + rank within my shard
  Better for rank queries, but score updates may cause cross-shard moves.

OPTION 3: Two-tier architecture (Recommended)
  Tier 1: Per-region leaderboard (Redis, per-region)
  Tier 2: Global leaderboard (batch aggregation every 5 min)

  Users see per-region rank in real-time.
  Global rank updated every 5 minutes (acceptable for most use cases).

  ┌──────────────────────────────────────────────┐
  │  Region US    │  Region EU    │  Region APAC  │
  │  Redis (50M)  │  Redis (30M)  │  Redis (20M)  │
  └───────┬───────┴───────┬───────┴───────┬──────┘
          │               │               │
          └───────────────┼───────────────┘
                          ▼
                  ┌───────────────┐
                  │ Global Rank    │
                  │ Aggregator     │
                  │ (every 5 min)  │
                  └───────────────┘

FOR MOST INTERVIEWS:
  50M users in a single Redis Sorted Set is fine.
  Only discuss sharding if interviewer pushes on scale.
```

### Deep Dive 4: Real-Time Notifications

```
RANK CHANGE NOTIFICATIONS:

When a user's rank changes significantly:
  "You moved from #42 to #15!"

Implementation:
  1. Before ZADD: ZREVRANK to get old rank
  2. ZADD with new score
  3. ZREVRANK to get new rank
  4. If |old_rank - new_rank| > threshold → push notification

  All 3 Redis commands in a single pipeline: ~1ms total

  For top 100 changes:
    Subscribe to top-100 changes via Kafka
    When someone enters/leaves top 100 → notify all top-100 users

LIVE LEADERBOARD UPDATES (Spectator Mode):
  WebSocket connection to leaderboard service
  Broadcast top-10 changes every 1 second
  Only the top-10 diffs sent (not full leaderboard)
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   LEADERBOARD SYSTEM                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐    ┌────────────┐    ┌──────────────────┐               │
│  │ Game    │───▶│ API Gateway │───▶│ Leaderboard Svc  │               │
│  │ Client  │    │ (auth,      │    │ (score updates,  │               │
│  │         │◀───│  rate limit)│◀───│  rank queries)   │               │
│  └────────┘    └────────────┘    └────────┬─────────┘               │
│       │                                    │                         │
│       │ WebSocket                          │                         │
│       │ (live rank updates)    ┌───────────┼───────────┐            │
│       │                        ▼           ▼           ▼            │
│       │                  ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│       │                  │ Redis    │ │ MySQL   │ │ User    │      │
│       │                  │ Sorted   │ │(history,│ │ Profile │      │
│       └──────────────────│ Sets     │ │ archive)│ │ Cache   │      │
│                          │(real-time│ └─────────┘ │(Redis)  │      │
│                          │ ranking) │             └─────────┘      │
│                          └─────────┘                                │
│                                                                      │
│  ┌───────────┐  ┌───────────────┐  ┌─────────────┐                 │
│  │ Kafka      │  │ Notification  │  │ Snapshot    │                 │
│  │ (score     │  │ Service       │  │ Worker      │                 │
│  │  events)   │  │ (rank alerts) │  │ (daily      │                 │
│  └───────────┘  └───────────────┘  │  archival)  │                 │
│                                     └─────────────┘                 │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Data store | Redis Sorted Set | MySQL ORDER BY score | Redis: O(log N) rank queries. MySQL: full table sort, far too slow for real-time. |
| Data structure | Skip list (Sorted Set internals) | Balanced BST | Skip list: simpler, cache-friendly, similar O(log N) performance. Used by Redis internally. |
| Tiebreaking | Composite score (score × 10^10 + timestamp) | Secondary sort | Redis only sorts by one score. Encoding timestamp into score handles ties deterministically. |
| Time-based boards | Separate Redis keys per period | Single key with filters | Separate keys: clean reset, simple archival. No need to filter within a set. |
| Scaling | Single Redis (50M) or regional shards | Global sorted set shard | 50M entries in one sorted set is ~800MB — trivially fits. Only shard for extreme scale. |
| History | Snapshot to MySQL on reset | Keep all in Redis | Redis is for real-time. Historical data belongs in durable, queryable storage (MySQL). |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | Why Redis Sorted Set for leaderboards? | O(log N) for insert, rank lookup, and range queries. 50M users fits in ~800MB RAM. Supports ZADD, ZREVRANK, ZREVRANGE natively. |
| 2 | How do you get a user's rank? | ZREVRANK key user_id → returns 0-indexed rank in O(log N). Highest score = rank 0. |
| 3 | How do you handle score ties? | Composite score: actual_score × 10^10 + (MAX_TIMESTAMP - time). Same score → earlier achiever ranks higher. Single sort key in Redis. |
| 4 | How do you implement daily leaderboard resets? | Separate Redis key per day (leaderboard:daily:2024-01-15). Snapshot to MySQL before delete. New key auto-created on first write. |
| 5 | What data structure does Redis Sorted Set use internally? | Skip list + hash table. Skip list provides O(log N) ordered operations. Hash table provides O(1) score lookups by member. |
| 6 | How do you show "users around me"? | Get user's rank with ZREVRANK, then ZREVRANGE from (rank-5) to (rank+5). Two Redis commands, both O(log N). |
| 7 | How do you scale beyond one Redis instance? | Regional sharding (per-region Redis) with periodic global aggregation. Or score-range sharding where global rank = higher-shard-count + local rank. |
| 8 | How do you notify users of rank changes? | Check rank before and after score update (pipeline both commands). If significant change → push notification via Kafka → notification service. |

---

[← Previous: Key-Value Store](./chapter-42-key-value-store.md) · [Next: Interview Framework →](../phase-8-final-boss/chapter-44-interview-framework-detailed.md)
