# Chapter 9: SQL vs NoSQL — The Decision Framework

> **Phase 2 — Databases & Storage** · Estimated Time: 2 hours · Difficulty: 🟡 Medium

[← Previous: Replication & Sharding](./chapter-08-replication-sharding.md) · [Next: Blob Storage & Data Lakes →](./chapter-10-blob-storage.md)

---

## Learning Objectives

- ✅ Apply the CAP theorem to real system design decisions
- ✅ Distinguish ACID vs BASE consistency models
- ✅ Use a practical decision matrix to choose between SQL and NoSQL
- ✅ Understand polyglot persistence — using multiple databases in one system
- ✅ Defend your database choice in an interview with clear trade-offs

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 6 (SQL) & Chapter 7 (NoSQL) | ✅ Yes |
| Chapter 8 (Replication & Sharding) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Interview relevance:** "Why did you choose PostgreSQL over Cassandra?" is one of the most common follow-up questions. A weak answer is "because it's what I know." A strong answer discusses data model, access patterns, consistency requirements, and scaling needs. This chapter gives you the framework to make and defend that choice every time.

---

## Core Concepts

### 1. The CAP Theorem

In a distributed system, when a network partition occurs, you must choose between **Consistency** and **Availability**. You can't have both during a partition.

```
┌────────────────────────────────────────────────┐
│              THE CAP THEOREM                    │
├────────────────────────────────────────────────┤
│                                                │
│           Consistency (C)                      │
│              ╱╲                                │
│             ╱  ╲                               │
│            ╱    ╲                              │
│           ╱  CP  ╲     CP Systems:             │
│          ╱        ╲    • Traditional RDBMS     │
│         ╱    ╱╲    ╲   • HBase, MongoDB (default)│
│        ╱    ╱  ╲    ╲  • Redis (single node)   │
│       ╱    ╱    ╲    ╲                         │
│      ╱    ╱  CA   ╲    ╲                       │
│     ╱    ╱(only in ╲    ╲                      │
│    ╱    ╱ no-       ╲    ╲                     │
│   ╱    ╱ partition)  ╲    ╲                    │
│  ╱    ╱               ╲    ╲                   │
│ ╱   AP                 ╲    ╲                  │
│╱                         ╲                     │
│ Availability (A)     Partition               │
│                      Tolerance (P)            │
│ AP Systems:                                    │
│ • Cassandra, DynamoDB                          │
│ • CouchDB, Riak                               │
│                                                │
│ REALITY: P is not optional in distributed      │
│ systems. Networks WILL partition. So the real   │
│ choice is: CP or AP.                           │
│                                                │
│ CP = Reject requests during partition           │
│      (data is always correct)                   │
│ AP = Serve requests during partition            │
│      (data may be stale/inconsistent)           │
└────────────────────────────────────────────────┘
```

**Interview tip:** Don't just say "CAP theorem." Say: "Since network partitions are inevitable in distributed systems, we have to choose between consistency and availability. For a banking system, I'd choose CP because incorrect balances are unacceptable. For a social media feed, I'd choose AP because showing a slightly stale feed is better than showing nothing."

### 2. ACID vs BASE

| ACID (SQL) | BASE (NoSQL) |
|-----------|-------------|
| **A**tomicity — all-or-nothing transactions | **B**asically **A**vailable — system is always available |
| **C**onsistency — data is always valid | **S**oft state — state may change over time (without input) |
| **I**solation — transactions don't interfere | **E**ventual consistency — data will converge eventually |
| **D**urability — committed data survives crashes | |
| **Trade-off:** Slower, harder to distribute | **Trade-off:** Faster, easier to scale, but weaker guarantees |

### 3. The Decision Matrix

```
┌──────────────────────────────────────────────────────────────┐
│              SQL vs NoSQL DECISION MATRIX                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  CHOOSE SQL WHEN:                                            │
│  ✅ Data is structured with clear relationships              │
│  ✅ Need ACID transactions (payments, inventory)             │
│  ✅ Complex queries with JOINs, aggregations                 │
│  ✅ Schema is well-defined and stable                        │
│  ✅ Data integrity is critical                               │
│  ✅ Read-heavy with complex queries                          │
│                                                              │
│  CHOOSE NoSQL WHEN:                                          │
│  ✅ Schema evolves frequently                                │
│  ✅ Need horizontal write scaling                            │
│  ✅ Data is semi-structured or hierarchical                  │
│  ✅ Simple access patterns (key-value, time-series)          │
│  ✅ Need very high write throughput                          │
│  ✅ Eventual consistency is acceptable                       │
│  ✅ Need low-latency reads from memory (caching)            │
│                                                              │
│  COMMON INTERVIEW PATTERNS:                                  │
│  User accounts, orders, payments → SQL (PostgreSQL/MySQL)   │
│  Caching, sessions, rate limiting → Key-Value (Redis)       │
│  Messages, events, time-series → Wide-Column (Cassandra)    │
│  Product catalogs, CMS → Document (MongoDB)                  │
│  Social graphs, recommendations → Graph (Neo4j)             │
│  Search, autocomplete → Search Engine (Elasticsearch)        │
│  Files, images, videos → Object Storage (S3)                │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4. Polyglot Persistence in Practice

```
E-Commerce System (Amazon-like):

┌────────────┐
│   Client    │
└─────┬──────┘
      ▼
┌────────────────┐
│  API Gateway    │
└───┬───┬───┬────┘
    │   │   │
    ▼   │   ▼
┌──────┐│┌──────────┐
│ User ││| Product  │
│ Svc  │││ Catalog  │
│      │││ Svc      │
│MySQL │││MongoDB   │ ← Flexible product attributes
└──────┘│└──────────┘
        │
    ┌───▼───┐  ┌────────┐  ┌──────────┐  ┌───────────┐
    │ Order  │  │ Cart   │  │ Search   │  │ Analytics │
    │ Svc    │  │ Svc    │  │ Svc      │  │ Svc       │
    │        │  │        │  │          │  │           │
    │MySQL   │  │Redis   │  │Elastic   │  │Cassandra  │
    │(ACID!) │  │(fast!) │  │search    │  │(write-    │
    └────────┘  └────────┘  └──────────┘  │ heavy)    │
                                           └───────────┘
```

---

## Common Interview Questions

### Q1: "SQL or NoSQL for a chat application?"

**Model answer:**
"Both. I'd use PostgreSQL for user accounts and group metadata — structured data that needs ACID. For messages, I'd use Cassandra because it's optimized for write-heavy, time-ordered data. The partition key would be `(chat_id, bucket)` with clustering by `created_at`, so fetching recent messages in a chat is a single-partition query. Redis for online presence and message delivery tracking because it needs sub-millisecond reads. This is polyglot persistence — each database is chosen for its access pattern."

### Q2: "Explain the CAP theorem with an example."

**Model answer:**
"The CAP theorem says during a network partition in a distributed system, you choose consistency or availability. Example: we have 2 database nodes. The network between them goes down. A client writes to Node A. Now if another client reads from Node B, do we return stale data (AP — available but inconsistent) or reject the read until nodes sync (CP — consistent but unavailable)? DynamoDB defaults to AP — it serves reads even if they might be stale, because availability matters more for shopping carts. A banking database would choose CP — it's better to reject a request than return a wrong balance."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "NoSQL is always better for scaling" | SQL scales well with replicas, caching, and sharding (Vitess, Citus). NoSQL has different trade-offs, not universally better scaling. |
| "CAP theorem means pick 2 of 3" | In practice, P is mandatory. The choice is CP or AP during a partition. Most of the time (no partition), you have all three. |
| Choosing a database without stating the access pattern | Always state: "The primary access pattern is X, so I chose Y because Z." |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 9 CHEAT SHEET                         │
│                  SQL vs NoSQL Decision                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CAP: During partition, choose Consistency or Availability      │
│  CP: Banking, inventory (reject requests vs wrong data)         │
│  AP: Social feeds, shopping carts (stale data OK)               │
│                                                                  │
│  ACID (SQL): Strong guarantees, complex queries, JOINs          │
│  BASE (NoSQL): Eventual consistency, simple access, scale       │
│                                                                  │
│  QUICK DECISION:                                                 │
│  Structured + relationships + ACID → SQL (PostgreSQL/MySQL)     │
│  Caching + sessions → Redis                                     │
│  Write-heavy + time-ordered → Cassandra                         │
│  Flexible schema → MongoDB                                       │
│  Full-text search → Elasticsearch                                │
│  Files/media → S3                                                │
│                                                                  │
│  POLYGLOT PERSISTENCE = multiple DBs, each for its strength     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What does the CAP theorem state? | In a distributed system during a network partition, you must choose between consistency (all reads return latest write) and availability (all requests get a response). |
| 2 | What does ACID stand for? | Atomicity, Consistency, Isolation, Durability |
| 3 | What does BASE stand for? | Basically Available, Soft state, Eventually consistent |
| 4 | CP vs AP — which would you choose for a payment system? | CP — incorrect balances are unacceptable. Better to reject a request than process it with stale data. |
| 5 | What is polyglot persistence? | Using multiple database technologies in one system, each chosen for its specific access pattern and strengths. |
| 6 | When is eventual consistency acceptable? | Social media feeds, like counts, follower counts, analytics — where being slightly stale doesn't cause harm. |
| 7 | What's the default interview choice for user account data? | PostgreSQL or MySQL — structured, relational, needs ACID for account operations. |
| 8 | Why might you use both SQL and NoSQL in one system? | Different data has different access patterns. User accounts need ACID (SQL), messages need high write throughput (Cassandra), caching needs low latency (Redis). |

---

[← Previous: Replication & Sharding](./chapter-08-replication-sharding.md) · [Next: Blob Storage & Data Lakes →](./chapter-10-blob-storage.md)
