# Chapter 8: Database Scaling — Replication & Sharding

> **Phase 2 — Databases & Storage** · Estimated Time: 3.5 hours · Difficulty: 🟡 Medium

[← Previous: NoSQL Databases](./chapter-07-nosql-databases.md) · [Next: SQL vs NoSQL →](./chapter-09-sql-vs-nosql.md)

---

## Learning Objectives

- ✅ Explain leader-follower, multi-leader, and leaderless replication
- ✅ Design a sharding strategy using hash-based or range-based partitioning
- ✅ Understand consistent hashing and why naive hashing fails
- ✅ Handle rebalancing when adding or removing database nodes
- ✅ Discuss replication lag and its impact on system design

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 6 (SQL Databases) | ✅ Yes |
| Chapter 7 (NoSQL Databases) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 2.5 hrs |
| Strategy exercises | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** Replication is like making photocopies of an important document — if one copy is destroyed, others survive. Sharding is like splitting a phone book into volumes (A-M, N-Z) — each volume is smaller and faster to search, but you need to know which volume to check.

**Interview relevance:** "How would you scale the database?" is asked in nearly every system design interview. You MUST be able to explain replication (for read scaling and availability) and sharding (for write scaling and storage). If you can also discuss consistent hashing, you'll stand out.

---

## Visual Diagram — Replication vs Sharding

```
REPLICATION (copies of the SAME data):
┌──────────────┐
│   Primary     │ ←── All writes go here
│   (Leader)    │
└──┬─────────┬──┘
   │ replicate│ replicate
   ▼         ▼
┌────────┐ ┌────────┐
│Replica 1│ │Replica 2│ ←── Reads distributed across replicas
│(Follower)│ │(Follower)│
└────────┘ └────────┘

Use case: Read scaling, high availability, disaster recovery
All nodes have ALL the data.


SHARDING (DIFFERENT data on each node):
┌────────────────────────────────────────────┐
│              All Data (Users A-Z)           │
└────┬──────────────┬──────────────┬─────────┘
     │              │              │
     ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1   │  │ Shard 2   │  │ Shard 3   │
│ Users A-I │  │ Users J-R │  │ Users S-Z │
└──────────┘  └──────────┘  └──────────┘

Use case: Write scaling, storage scaling
Each node has a SUBSET of the data.


TYPICAL PRODUCTION SETUP (both):
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1   │  │ Shard 2   │  │ Shard 3   │
│ Primary   │  │ Primary   │  │ Primary   │
│ Users A-I │  │ Users J-R │  │ Users S-Z │
├──────────┤  ├──────────┤  ├──────────┤
│ Replica   │  │ Replica   │  │ Replica   │
│ Replica   │  │ Replica   │  │ Replica   │
└──────────┘  └──────────┘  └──────────┘

Each shard has its own replicas. Best of both worlds.
```

---

## Core Concepts

### 1. Replication Strategies

#### Leader-Follower (Master-Slave) Replication

The most common pattern. One node (leader/primary) handles all writes. Followers replicate the leader's data and serve reads.

```
┌──────────────┐
│   Leader      │ ←── Writes (INSERT, UPDATE, DELETE)
│   (Primary)   │
└──┬─────────┬──┘
   │  async  │  async
   │  or     │  or
   │  sync   │  sync
   ▼         ▼
┌────────┐ ┌────────┐
│Follower │ │Follower │ ←── Reads (SELECT)
│   1     │ │   2     │
└────────┘ └────────┘

Write path:  Client → Leader → commit → replicate to followers
Read path:   Client → any Follower (or Leader)
```

**Synchronous vs Asynchronous replication:**

| Mode | How it works | Trade-off |
|------|-------------|-----------|
| **Synchronous** | Leader waits for follower(s) to confirm before responding to client | Strong consistency, but slower writes and less available (follower down = writes blocked) |
| **Asynchronous** | Leader responds immediately, replicates in background | Fast writes, highly available, but followers may serve stale data (replication lag) |
| **Semi-synchronous** | 1 follower is sync, rest are async | Balance: at least 1 up-to-date copy guaranteed |

**Replication lag:** With async replication, followers may be milliseconds to seconds behind the leader. This causes:
- **Read-after-write inconsistency:** User posts a tweet, refreshes, doesn't see it (read hit a lagging follower)
- **Fix:** Read-your-writes consistency — route the user's own reads to the leader for recently written data

#### Multi-Leader Replication

Multiple nodes accept writes. Used for multi-datacenter setups.

```
Datacenter 1              Datacenter 2
┌──────────┐              ┌──────────┐
│ Leader 1  │◄═══════════▶│ Leader 2  │  (bidirectional replication)
│ (writes)  │              │ (writes)  │
├──────────┤              ├──────────┤
│ Followers │              │ Followers │
└──────────┘              └──────────┘

Challenge: WRITE CONFLICTS
  User A edits document on Leader 1
  User B edits same document on Leader 2
  → Both leaders have different versions!

Conflict resolution strategies:
  • Last write wins (LWW) — timestamp-based, simple but lossy
  • Merge values — application-specific logic
  • Keep both versions — let the user resolve
```

**When to use multi-leader:** Multi-datacenter deployments where you need low-latency writes in each region (e.g., global applications like Google Docs).

#### Leaderless Replication

All nodes accept reads AND writes. Used by DynamoDB, Cassandra, Riak.

```
Client writes to MULTIPLE nodes simultaneously:

Client ──────┬──── Write ────▶ Node A ✅
             ├──── Write ────▶ Node B ✅
             └──── Write ────▶ Node C ❌ (failed)

If W nodes acknowledge (W=2), write is considered successful.

Client reads from MULTIPLE nodes:
Client ──────┬──── Read ─────▶ Node A → version 5
             ├──── Read ─────▶ Node B → version 5
             └──── Read ─────▶ Node C → version 4 (stale)

If R nodes respond (R=2), take the latest version.

QUORUM: R + W > N ensures you always read the latest write.
  N=3 nodes, W=2, R=2: 2+2=4 > 3 ✅ (quorum satisfied)
```

---

### 2. Sharding (Partitioning) Strategies

When a single database can't hold all your data or handle all your writes, you split data across multiple nodes.

#### Hash-Based Sharding

```
shard_number = hash(shard_key) % number_of_shards

Example with 4 shards:
  hash("user_123") % 4 = 2 → Shard 2
  hash("user_456") % 4 = 0 → Shard 0
  hash("user_789") % 4 = 3 → Shard 3

┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Shard 0  │  │Shard 1  │  │Shard 2  │  │Shard 3  │
│user_456 │  │user_234 │  │user_123 │  │user_789 │
│user_012 │  │user_567 │  │user_345 │  │user_901 │
└────────┘  └────────┘  └────────┘  └────────┘
```

| Pros | Cons |
|------|------|
| Even data distribution | No range queries (data is scattered) |
| Simple to implement | Adding/removing shards requires rehashing ALL data |
| Prevents hot spots | — |

#### Range-Based Sharding

```
Shard by ranges of the shard key:

┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard 1   │  │ Shard 2   │  │ Shard 3   │
│ A-I       │  │ J-R       │  │ S-Z       │
│ (users    │  │ (users    │  │ (users    │
│  whose    │  │  whose    │  │  whose    │
│  name     │  │  name     │  │  name     │
│  starts   │  │  starts   │  │  starts   │
│  A-I)     │  │  J-R)     │  │  S-Z)     │
└──────────┘  └──────────┘  └──────────┘
```

| Pros | Cons |
|------|------|
| Range queries work (get all users A-C) | Uneven distribution (hot spots) |
| Data locality for related items | Some shards may be much larger than others |

#### Directory-Based Sharding

A lookup service maps each key to its shard.

```
┌──────────────────┐
│ Shard Directory   │
│ user_123 → Shard 2│
│ user_456 → Shard 0│
│ user_789 → Shard 3│
└────────┬─────────┘
         │ lookup
         ▼
   Route to correct shard
```

| Pros | Cons |
|------|------|
| Flexible, can rebalance easily | Directory is a SPOF and bottleneck |
| No constraints on distribution | Extra lookup latency |

### 3. Choosing a Shard Key

The shard key determines how data is distributed. A bad shard key creates hot spots.

```
GOOD shard key: user_id
  • Even distribution (millions of users)
  • All of a user's data on one shard (locality)
  • Most queries are per-user

BAD shard key: created_date
  • All today's writes go to ONE shard (hot spot!)
  • Yesterday's shard sits idle

BAD shard key: country
  • USA shard is 100x larger than Iceland shard
  • Extremely uneven distribution
```

**Rule of thumb:** Choose a shard key with high cardinality (many unique values) and even distribution that aligns with your most common query pattern.

### 4. Consistent Hashing (Preview)

The problem with `hash(key) % N`: when you add or remove a shard, almost ALL data must be remapped.

```
With 3 shards: hash(key) % 3
  key_1 → 2, key_2 → 0, key_3 → 1, key_4 → 2

Add a 4th shard: hash(key) % 4
  key_1 → 1(!), key_2 → 0, key_3 → 3(!), key_4 → 0(!)

3 out of 4 keys moved! At scale with billions of keys,
this means massive data migration.

Consistent hashing solves this: only K/N keys move when
a node is added (K = total keys, N = number of nodes).
```

> Full deep dive in [Chapter 22: Consistent Hashing](../phase-5-distributed-systems/chapter-22-consistent-hashing.md).

### 5. Cross-Shard Queries and Joins

The biggest challenge with sharding: queries that span multiple shards.

```
Query: "Get all tweets from users that user_123 follows"

If user_123 follows users on shards 1, 2, and 3:
  → Must query ALL 3 shards and merge results
  → Called a "scatter-gather" query
  → Much slower than a single-shard query

Solutions:
  1. Denormalize: store follower tweets in user's shard
  2. Application-level joins: query multiple shards in parallel
  3. Avoid cross-shard queries by design (choose shard key wisely)
```

---

## Comparison Table — Replication vs Sharding

| Aspect | Replication | Sharding |
|--------|-------------|----------|
| **Purpose** | Read scaling, availability | Write scaling, storage scaling |
| **Data distribution** | All nodes have ALL data | Each node has a SUBSET |
| **Write scaling** | ❌ No (single leader bottleneck) | ✅ Yes |
| **Read scaling** | ✅ Yes (read from replicas) | ✅ Yes |
| **Availability** | ✅ Replicas survive leader failure | ❌ Shard failure = data unavailable |
| **Complexity** | Low-medium | High |
| **When to use** | First scaling step | When single node can't hold all data |

---

## Real-World Example — Instagram's Sharding

```
Instagram shards PostgreSQL by user_id using Vitess:

┌─────────────────────────────────────────────────────┐
│                Instagram Data Layer                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Shard Key: user_id                                 │
│  Strategy: Hash-based (consistent hashing)          │
│  Shards: ~1000+ PostgreSQL instances                │
│                                                     │
│  Benefits:                                          │
│  • All of a user's photos on one shard              │
│  • User profile, followers, likes — same shard      │
│  • No cross-shard JOINs for common queries          │
│                                                     │
│  Challenge: "Who likes this photo?" spans shards    │
│  Solution: Denormalized like counts on photo row    │
│            + async fan-out for notifications        │
│                                                     │
│  Each shard:                                        │
│  ┌─────────────────┐                                │
│  │ Primary (writes) │                                │
│  │ Replica (reads)  │                                │
│  │ Replica (reads)  │                                │
│  └─────────────────┘                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Common Interview Questions

### Q1: "When would you replicate vs shard?"

**Model answer:**
"Replicate first, shard later. Replication is simpler — I'd start with a primary database and 2-3 read replicas to distribute read load. This handles most read-heavy workloads. I'd only shard when the single primary can't handle the write load, or when the data doesn't fit on a single machine. Sharding adds significant complexity: I need to choose a shard key, handle cross-shard queries, and deal with rebalancing. In practice, I'd shard by user_id so all of a user's data lives on one shard, minimizing cross-shard operations."

### Q2: "What happens when a leader fails in leader-follower replication?"

**Model answer:**
"This is called failover. A follower is promoted to become the new leader. The steps are: detect the leader is down (via heartbeat timeout), elect a new leader (typically the most up-to-date follower), and reconfigure clients and other followers to point to the new leader. With async replication, the promoted follower might be behind — any writes not yet replicated are lost. This is why some systems use semi-synchronous replication: at least one follower is always up-to-date and can be safely promoted."

### Q3: "How do you handle hot spots in sharding?"

**Model answer:**
"Hot spots occur when one shard gets disproportionately more traffic. For example, if I shard by user_id and a celebrity with 100M followers posts, their shard gets hammered. Solutions: first, use consistent hashing to distribute data evenly. Second, for known hot keys (celebrities), add a random suffix to the shard key to spread their data across multiple shards — this is called key-based salting. Third, add caching in front of the database to absorb repeated reads for hot data. Fourth, if a shard is consistently hot, split it into smaller shards."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "We'll shard from day one" | Start with replication. Shard only when you must. Premature sharding adds unnecessary complexity. |
| Ignoring replication lag | Async replication has lag. Plan for read-after-write consistency where it matters. |
| Using a low-cardinality shard key | Shard key must have many unique values. Sharding by country or status gives terrible distribution. |
| "Cross-shard joins are fine" | Cross-shard operations are expensive. Design your shard key to avoid them for common queries. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 8 CHEAT SHEET                         │
│                  Replication & Sharding                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  REPLICATION:                                                    │
│  • Leader-follower: 1 writer, N readers (most common)           │
│  • Multi-leader: N writers (multi-datacenter)                   │
│  • Leaderless: all nodes read+write (DynamoDB, Cassandra)       │
│  • Quorum: R + W > N for consistency                            │
│                                                                  │
│  SHARDING STRATEGIES:                                            │
│  • Hash-based: even distribution, no range queries              │
│  • Range-based: range queries work, risk hot spots              │
│  • Directory-based: flexible, but adds lookup overhead          │
│                                                                  │
│  SHARD KEY RULES:                                                │
│  • High cardinality (many unique values)                        │
│  • Even distribution                                            │
│  • Aligns with common query patterns                            │
│  • Common choice: user_id                                       │
│                                                                  │
│  SCALING ORDER:                                                  │
│  Vertical → Read replicas → Caching → Sharding                 │
│  (shard only when you absolutely must)                          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the difference between replication and sharding? | Replication: all nodes have all data (read scaling). Sharding: each node has a subset of data (write/storage scaling). |
| 2 | What is the quorum formula? | R + W > N, where R = read replicas queried, W = write replicas confirmed, N = total replicas |
| 3 | Why does `hash(key) % N` fail when N changes? | Almost all keys get remapped to different shards, requiring massive data migration |
| 4 | What's the best default shard key for most applications? | user_id — high cardinality, even distribution, keeps user's data on one shard |
| 5 | What is replication lag? | The delay between a write on the leader and that write appearing on followers (ms to seconds with async replication) |
| 6 | What is a "scatter-gather" query? | A query that must be sent to all shards, with results merged — expensive and slow |
| 7 | When should you shard a database? | When a single node can't handle write throughput or storage capacity, and you've already tried vertical scaling, read replicas, and caching |
| 8 | What is semi-synchronous replication? | One follower replicates synchronously (guaranteed up-to-date), rest replicate asynchronously (fast but may lag) |
| 9 | How does consistent hashing help with sharding? | Only K/N keys move when adding a node (vs. nearly all keys with modular hashing) |
| 10 | What is the "celebrity problem" in sharding? | A single hot key (e.g., celebrity user_id) creates a hot shard. Solved by key salting or caching. |

---

[← Previous: NoSQL Databases](./chapter-07-nosql-databases.md) · [Next: SQL vs NoSQL →](./chapter-09-sql-vs-nosql.md)
