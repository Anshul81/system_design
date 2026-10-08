# Chapter 46: Trade-off Analysis — The Art of System Design

> **Phase 8 — Interview Strategy** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium · Type: 📋 STRATEGY

[← Previous: Requirements Deep Dive](./chapter-45-requirements-deep-dive.md) · [Next: Common Mistakes →](./chapter-47-common-mistakes.md)

---

## Why This Chapter Matters

Trade-off analysis is the **single most important skill** in system design interviews. Every technology choice is a trade-off. Interviewers don't want you to pick the "right" answer — they want to see you **reason about alternatives** and explain **why** your choice fits the specific constraints. This chapter catalogs the most common trade-offs and teaches you how to discuss them.

---

## The Trade-off Discussion Framework

```
For EVERY major decision, follow this pattern:

1. STATE THE DECISION: "For the message store, I'm choosing between
   MySQL and Cassandra."

2. COMPARE OPTIONS:
   "MySQL gives me ACID transactions and strong consistency.
    Cassandra gives me higher write throughput and easier horizontal scaling."

3. JUSTIFY YOUR CHOICE:
   "Since we have 50B messages/day and need to scale writes linearly,
    I'll go with Cassandra. We don't need cross-row transactions for 
    messages, so we don't lose anything by giving up ACID."

4. ACKNOWLEDGE THE TRADE-OFF:
   "The trade-off is that queries across conversations are harder in 
    Cassandra. If we needed complex joins, I'd reconsider MySQL with
    sharding."
```

---

## The 15 Most Common Trade-offs

### 1. SQL vs NoSQL

```
┌────────────┬──────────────────────┬──────────────────────┐
│            │ SQL (MySQL, PG)       │ NoSQL (Cassandra,    │
│            │                      │ DynamoDB, MongoDB)    │
├────────────┼──────────────────────┼──────────────────────┤
│ Consistency│ Strong (ACID)        │ Eventual (tunable)   │
│ Schema     │ Fixed, structured    │ Flexible, schemaless │
│ Joins      │ Native support       │ No joins (denormalize│
│ Scale      │ Vertical (sharding   │ Horizontal (built-in)│
│            │ requires effort)     │                      │
│ Best for   │ Relational data,     │ Write-heavy, large   │
│            │ transactions         │ scale, simple queries│
├────────────┼──────────────────────┼──────────────────────┤
│ Use when   │ User accounts, orders│ Messages, logs, time-│
│            │ payments, inventory  │ series, IoT data     │
└────────────┴──────────────────────┴──────────────────────┘

INTERVIEW TIP:
  Start with SQL unless you have a specific reason for NoSQL.
  "I'll use MySQL because ACID is important for [reason].
   If write throughput becomes a bottleneck, I'd consider 
   sharding or migrating to Cassandra."
```

### 2. Cache-Aside vs Write-Through vs Write-Behind

```
CACHE-ASIDE (Lazy loading):
  Read:  Check cache → miss → read DB → populate cache
  Write: Write DB → invalidate cache
  Pro: Only caches what's actually read (no wasted memory)
  Con: First read always misses (cold start)
  Use: URL shortener, most read-heavy systems

WRITE-THROUGH:
  Write: Write cache AND DB simultaneously
  Read:  Always hits cache
  Pro: Cache always consistent with DB
  Con: Write latency increases (write to both)
  Use: Session stores, data that's read immediately after write

WRITE-BEHIND (Write-back):
  Write: Write cache only → async flush to DB
  Read:  Always hits cache
  Pro: Fastest writes
  Con: Data loss risk if cache crashes before flush
  Use: View counters, non-critical metrics

INTERVIEW PATTERN:
  "I'll use cache-aside because most shortened URLs may never be
   accessed. Write-through would waste memory caching URLs that 
   nobody visits."
```

### 3. Push vs Pull (Fan-out on Write vs Fan-out on Read)

```
PUSH (Fan-out on Write):
  When data is created → push to all consumers' feeds/inboxes
  Pro: Reads are instant (pre-computed)
  Con: Write amplification (celebrity = millions of writes)
  Use: Twitter feed (for normal users)

PULL (Fan-out on Read):
  When user reads → fetch from all sources and merge
  Pro: No write amplification
  Con: Reads are slow (N queries + merge)
  Use: Celebrity tweets, low-frequency consumers

HYBRID:
  Push for normal users, pull for high-fan-out entities
  Best of both worlds
  Use: Twitter (actual implementation), Instagram

INTERVIEW PATTERN:
  "Pure push doesn't work because of the celebrity problem.
   Pure pull is too slow for reading. I'll use a hybrid:
   push for users with < 10K followers, pull for celebrities."
```

### 4. Synchronous vs Asynchronous Processing

```
SYNCHRONOUS:
  Client waits for the entire operation to complete
  Pro: Simple, client gets immediate confirmation
  Con: Slow if operation is expensive
  Use: Payment confirmation, user login

ASYNCHRONOUS (via message queue):
  Client gets immediate ACK, processing happens in background
  Pro: Fast response, handles spikes, retry-friendly
  Con: Complexity, eventual consistency
  Use: Video transcoding, email sending, fan-out

INTERVIEW PATTERN:
  "Video transcoding takes 10-60 minutes. I can't make the user wait.
   I'll accept the upload synchronously, return 'processing', and 
   transcode asynchronously via a Kafka job queue."
```

### 5. Strong vs Eventual Consistency

```
STRONG:
  After a write, all reads see the latest value
  Pro: Simple programming model, no stale data
  Con: Higher latency (must wait for replication), lower availability
  Use: Payments, inventory counts, user authentication

EVENTUAL:
  After a write, reads may return stale data temporarily
  Pro: Lower latency, higher availability
  Con: Application must handle stale reads
  Use: Like counts, view counts, social feeds, analytics

INTERVIEW PATTERN:
  "For payment amounts, I need strong consistency — we can't show 
   a wrong balance. For like counts, eventual consistency is fine — 
   showing 1,234 likes instead of 1,235 for 5 seconds is acceptable."
```

### 6. Monolith vs Microservices

```
MONOLITH:
  One deployable unit, one codebase
  Pro: Simple, faster development, easier debugging, no network calls
  Con: Hard to scale individual components, deployment risk
  Use: Early-stage startups, small teams

MICROSERVICES:
  Independent services, own databases, communicate via API/queue
  Pro: Independent scaling, team autonomy, technology flexibility
  Con: Network complexity, distributed transactions, debugging difficulty
  Use: Large-scale systems, big engineering teams

INTERVIEW PATTERN:
  For system design interviews, use MICROSERVICES for the diagram 
  (it's what interviewers expect at scale). But acknowledge:
  "In practice, I'd start with a modular monolith and extract 
   services as the team and scale grow."
```

### 7. REST vs gRPC vs GraphQL

```
REST:     Simple, HTTP/JSON, browser-friendly, cacheable
gRPC:     Binary (Protobuf), fast, streaming, service-to-service
GraphQL:  Client-specified queries, flexible, mobile-friendly

INTERVIEW PATTERN:
  External APIs: REST (familiar, browser-compatible)
  Service-to-service: gRPC (lower latency, type-safe)
  Mobile apps with varied data needs: GraphQL
  
  In interviews: default to REST unless there's a specific reason not to.
```

### 8. Polling vs WebSocket vs SSE

```
POLLING:       Client requests every N seconds
               Simple but wasteful. Use for: dashboards, non-real-time.

LONG POLLING:  Client requests, server holds until data is available
               Better latency, but connection-heavy. Use for: notifications.

WebSocket:     Full-duplex persistent connection
               Best for: chat, live tracking, collaborative editing.

SSE:           Server pushes events to client (one-way)
               Use for: live feeds, stock tickers, score updates.

INTERVIEW PATTERN:
  Chat/messaging: "WebSocket because we need bidirectional, 
   low-latency communication."
  Live feed updates: "SSE because it's server-to-client only, 
   simpler than WebSocket."
  Dashboard refresh: "Polling every 30s because the data doesn't 
   need to be real-time."
```

### 9. Pre-computed vs Computed On-the-fly

```
PRE-COMPUTED:
  Compute in advance, serve from cache/materialized view
  Pro: Instant reads
  Con: Stale data, write amplification
  Use: Feeds, leaderboards, recommendation lists

ON-THE-FLY:
  Compute at read time from raw data
  Pro: Always fresh
  Con: Slow reads, compute-heavy
  Use: Search results, ad-hoc reports

INTERVIEW PATTERN:
  "The home timeline is pre-computed and cached in Redis because 
   users load it hundreds of times a day. Search results are computed 
   on-the-fly because queries are unpredictable."
```

### 10. Sharding Strategy: Hash vs Range

```
HASH-BASED:
  shard = hash(key) % N
  Pro: Even distribution
  Con: Range queries span all shards, resharding is painful
  Use: User data, KV stores

RANGE-BASED:
  shard 1: A-M, shard 2: N-Z
  Pro: Range queries efficient within a shard
  Con: Hot spots (popular ranges get more traffic)
  Use: Time-series data, alphabetical data

CONSISTENT HASHING:
  Hash ring with virtual nodes
  Pro: Adding/removing nodes moves minimal data
  Use: Distributed caches, DynamoDB-style systems
```

### 11-15: Quick Reference

```
11. OPTIMISTIC vs PESSIMISTIC LOCKING
    Optimistic: check version on write (low contention)
    Pessimistic: lock row before read (high contention)

12. LEADER-BASED vs LEADERLESS REPLICATION
    Leader: one node handles writes (simpler, strong consistency)
    Leaderless: any node handles writes (higher availability, conflicts)

13. B-TREE vs LSM TREE
    B-Tree: read-optimized (MySQL, PostgreSQL)
    LSM Tree: write-optimized (Cassandra, RocksDB, LevelDB)

14. BATCH vs STREAM PROCESSING
    Batch: process accumulated data periodically (Spark, MapReduce)
    Stream: process events as they arrive (Flink, Kafka Streams)

15. CDN PUSH vs CDN PULL
    Push: origin pushes content to CDN proactively (predictable content)
    Pull: CDN fetches from origin on cache miss (unpredictable access)
```

---

## Trade-off Decision Matrix

```
┌──────────────────────┬───────────────────────────────────────────┐
│ If you need...        │ Choose...                                │
├──────────────────────┼───────────────────────────────────────────┤
│ Strong consistency    │ SQL (MySQL/PG), synchronous replication  │
│ High write throughput │ Cassandra, Kafka buffer, LSM Tree        │
│ Low read latency      │ Redis cache, CDN, denormalization        │
│ Real-time updates     │ WebSocket (bidirectional), SSE (one-way) │
│ Exact counting        │ SQL with transactions                    │
│ Approximate counting  │ Redis INCR, HyperLogLog                 │
│ Full-text search      │ Elasticsearch                           │
│ Time-series data      │ InfluxDB, TimescaleDB                   │
│ Geospatial queries    │ Redis GEO, PostGIS, GeoHash             │
│ File/blob storage     │ S3 + CDN                                │
│ Event ordering        │ Kafka (partitioned by key)              │
│ Distributed locking   │ Redis SETNX, ZooKeeper                  │
│ Service discovery     │ etcd, Consul, ZooKeeper                 │
│ Rate limiting         │ Redis (sliding window counter)           │
└──────────────────────┴───────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the pattern for discussing trade-offs? | 1. State the decision, 2. Compare 2-3 options, 3. Justify your choice with requirements, 4. Acknowledge what you're giving up. |
| 2 | When to use SQL vs NoSQL? | SQL: transactions, relationships, strong consistency (payments, users). NoSQL: high write throughput, flexible schema, horizontal scale (messages, logs). |
| 3 | Cache-aside vs write-through? | Cache-aside: only cache what's read (saves memory). Write-through: cache on every write (always consistent). Choose based on read pattern predictability. |
| 4 | When to use synchronous vs async processing? | Sync: user needs immediate confirmation (payment). Async: operation is slow (video transcoding) or can be batched (notifications). |
| 5 | Push vs pull model? | Push: pre-compute for fast reads, but write amplification for popular sources. Pull: no write cost, but slow reads. Hybrid: push for normal, pull for popular. |
| 6 | When is eventual consistency acceptable? | When stale data doesn't cause harm: like counts, view counts, search indices, social feeds. NOT for payments, inventory, authentication. |
| 7 | B-Tree vs LSM Tree? | B-Tree: read-optimized (in-place updates). LSM Tree: write-optimized (sequential writes, background compaction). Most KV stores use LSM. |

---

[← Previous: Requirements Deep Dive](./chapter-45-requirements-deep-dive.md) · [Next: Common Mistakes →](./chapter-47-common-mistakes.md)
