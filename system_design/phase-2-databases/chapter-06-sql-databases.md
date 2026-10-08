# Chapter 6: SQL Databases — Relational Modeling

> **Phase 2 — Databases & Storage** · Estimated Time: 3 hours · Difficulty: 🟡 Medium

[← Previous: Interview Framework](../phase-1-scalability/chapter-05-interview-framework.md) · [Next: NoSQL Databases →](./chapter-07-nosql-databases.md)

---

## Learning Objectives

- ✅ Explain ACID properties and why they matter for data integrity
- ✅ Design normalized database schemas for system design problems
- ✅ Choose the right indexing strategy (B-tree vs hash index)
- ✅ Understand transactions, isolation levels, and connection pooling
- ✅ Know when SQL is the right choice (and when it isn't)

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Phase 0-1 completed | ✅ Yes |
| Basic SQL query knowledge | Helpful |

| Component | Time |
|-----------|------|
| Reading & concepts | 2 hrs |
| Schema design practice | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** A relational database is like a well-organized spreadsheet system where every table is a sheet, every row is a record, and relationships between sheets are explicitly defined. You can't have an order without a customer. You can't have a comment without a post. These relationships are enforced by the database.

**Interview relevance:** Most system design interviews involve designing a data model. Whether it's Twitter, URL shortener, or e-commerce, the interviewer expects you to sketch a schema with tables, columns, primary keys, and indexes. Knowing ACID properties, indexing strategies, and when normalization helps vs hurts is essential for the "detailed design" step.

---

## Visual Diagram — How SQL Databases Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    SQL DATABASE ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Application                                                     │
│       │                                                          │
│       │  SQL Query: SELECT * FROM users WHERE id = 123           │
│       ▼                                                          │
│  ┌──────────────────────────────────────────────────┐           │
│  │                 Query Processor                   │           │
│  │  ┌─────────────┐  ┌────────────┐  ┌───────────┐ │           │
│  │  │   Parser     │→│  Optimizer  │→│ Executor   │ │           │
│  │  │  (SQL→AST)   │  │(pick best  │  │(run plan) │ │           │
│  │  │              │  │ index/join │  │           │ │           │
│  │  └─────────────┘  │ strategy)  │  └─────┬─────┘ │           │
│  │                    └────────────┘        │       │           │
│  └──────────────────────────────────────────┼───────┘           │
│                                              │                   │
│  ┌──────────────────────────────────────────┼───────┐           │
│  │              Storage Engine               │       │           │
│  │  ┌──────────────┐    ┌──────────────┐    │       │           │
│  │  │  Buffer Pool  │    │  Transaction  │    │       │           │
│  │  │  (in-memory   │    │  Log (WAL)    │◄───┘       │           │
│  │  │   page cache) │    │              │            │           │
│  │  └──────┬───────┘    └──────────────┘            │           │
│  │         │                                         │           │
│  │  ┌──────▼────────────────────────────────────┐   │           │
│  │  │           B-Tree Indexes                   │   │           │
│  │  │  ┌───┐                                     │   │           │
│  │  │  │ R │         Root node                   │   │           │
│  │  │  └┬─┬┘                                     │   │           │
│  │  │   │ │                                      │   │           │
│  │  │  ┌▼┐┌▼┐       Internal nodes               │   │           │
│  │  │  │ ││ │                                    │   │           │
│  │  │  └┬┘└┬┘                                    │   │           │
│  │  │  ┌▼┐┌▼┐┌▼┐    Leaf nodes (data pointers)  │   │           │
│  │  │  │ ││ ││ │                                 │   │           │
│  │  │  └─┘└─┘└─┘                                 │   │           │
│  │  └────────────────────────────────────────────┘   │           │
│  │                                                    │           │
│  │  ┌───────────────────────────────────────────┐    │           │
│  │  │              Data Files (Disk)             │    │           │
│  │  │  [Page 1] [Page 2] [Page 3] ... [Page N]  │    │           │
│  │  └───────────────────────────────────────────┘    │           │
│  └────────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### 1. ACID Properties

ACID is the guarantee that SQL databases provide for every transaction. This is why you choose SQL when data integrity is critical.

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All-or-nothing. Either the entire transaction succeeds or nothing changes. | Bank transfer: debit Account A AND credit Account B. If credit fails, debit is rolled back. |
| **Consistency** | Data always moves from one valid state to another. Constraints are enforced. | Foreign keys, unique constraints, NOT NULL — the database rejects invalid data. |
| **Isolation** | Concurrent transactions don't interfere with each other. | Two users buying the last item simultaneously — only one succeeds. |
| **Durability** | Once committed, data survives server crashes. | Data is written to disk (WAL) before confirming the transaction. |

**Interview tip:** When the interviewer asks "Why SQL for this?", the answer is usually "We need ACID guarantees — specifically [atomicity/consistency] for [specific reason]." For example: "We need atomicity for payment processing — we can't debit without crediting."

### 2. Normalization vs Denormalization

**Normalization** = eliminate data redundancy by splitting data into related tables.
**Denormalization** = intentionally add redundancy for read performance.

```
NORMALIZED (3NF):
┌─────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│ users             │     │ tweets                │     │ likes              │
├─────────────────┤     ├─────────────────────┤     ├──────────────────┤
│ id (PK)          │     │ id (PK)              │     │ id (PK)           │
│ username         │◄────│ user_id (FK)         │     │ user_id (FK)      │
│ display_name     │     │ content              │◄────│ tweet_id (FK)     │
│ bio              │     │ created_at           │     │ created_at        │
│ created_at       │     │ media_url            │     └──────────────────┘
└─────────────────┘     └─────────────────────┘

To get "tweets with author names" → JOIN users and tweets
Pro: No duplicate data, easy updates
Con: JOINs are expensive at scale


DENORMALIZED:
┌────────────────────────────────┐
│ tweets_with_user                │
├────────────────────────────────┤
│ tweet_id (PK)                  │
│ user_id                        │
│ username         ← duplicated  │
│ display_name     ← duplicated  │
│ content                        │
│ created_at                     │
│ like_count       ← pre-computed│
│ media_url                      │
└────────────────────────────────┘

No JOIN needed — all data in one table
Pro: Fast reads (single table scan)
Con: Username update → must update ALL their tweets
```

**When to denormalize:**
- Read-heavy workloads (>>100:1 read:write ratio)
- Need to avoid expensive JOINs at scale
- Data that changes rarely (usernames don't change often)

**Interview tip:** "I'd start normalized for data integrity, then selectively denormalize the hot read paths. For example, I'd store `author_username` in the tweets table to avoid JOINing on every timeline fetch."

### 3. Indexing — B-Tree vs Hash

Indexes are the #1 tool for improving query performance. Without an index, the database does a full table scan (reading every row).

```
WITHOUT INDEX (Full Table Scan):
  Query: SELECT * FROM users WHERE email = 'john@x.com'
  
  [Row 1] → check → no match
  [Row 2] → check → no match
  [Row 3] → check → no match
  ...
  [Row 999,999] → check → MATCH!
  
  Time: O(n) — must scan ALL rows

WITH B-TREE INDEX:
  Query: SELECT * FROM users WHERE email = 'john@x.com'
  
  B-Tree traversal: Root → Internal → Leaf → Row pointer
  
  Time: O(log n) — typically 3-4 disk reads for millions of rows
```

| Index Type | Best For | How It Works | Limitations |
|-----------|----------|-------------|-------------|
| **B-Tree** (default) | Range queries, sorting, equality | Balanced tree, ordered keys | Slower writes (must maintain tree) |
| **Hash Index** | Exact equality lookups | Hash function → bucket | No range queries, no sorting |
| **Composite Index** | Multi-column queries | B-tree on (col1, col2, ...) | Order matters (leftmost prefix) |
| **Full-text Index** | Text search | Inverted index | Specialized, not for exact match |

**Composite index ordering matters:**
```
INDEX on (user_id, created_at):

✅ WHERE user_id = 123                    (uses index)
✅ WHERE user_id = 123 AND created_at > X  (uses index)
❌ WHERE created_at > X                   (can't use index — leftmost prefix rule)
```

**Interview tip:** When designing schemas, always mention your indexing strategy. "I'd create a composite index on `(user_id, created_at)` for the timeline query, since we always fetch tweets for a specific user ordered by time."

### 4. Transactions and Isolation Levels

Isolation levels control what concurrent transactions can see:

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Performance |
|-------|-------------|---------------------|---------------|-------------|
| **Read Uncommitted** | ✅ Possible | ✅ Possible | ✅ Possible | Fastest |
| **Read Committed** | ❌ Prevented | ✅ Possible | ✅ Possible | Fast |
| **Repeatable Read** | ❌ Prevented | ❌ Prevented | ✅ Possible | Medium |
| **Serializable** | ❌ Prevented | ❌ Prevented | ❌ Prevented | Slowest |

**For interviews:** Most systems use **Read Committed** (PostgreSQL default) or **Repeatable Read** (MySQL InnoDB default). Only mention Serializable for critical sections like payment processing.

### 5. Connection Pooling

```
WITHOUT Connection Pool:
Each request creates a new DB connection (~50ms overhead):

Request 1 → [Connect] → [Query] → [Close]
Request 2 → [Connect] → [Query] → [Close]
Request 3 → [Connect] → [Query] → [Close]

WITH Connection Pool:
Connections are pre-created and reused:

Pool: [Conn 1] [Conn 2] [Conn 3] [Conn 4] [Conn 5]

Request 1 → borrow Conn 1 → [Query] → return Conn 1
Request 2 → borrow Conn 2 → [Query] → return Conn 2
Request 3 → borrow Conn 1 → [Query] → return Conn 1
```

**Pool sizing rule of thumb:** connections = (CPU cores × 2) + effective_disk_spindles. For most systems, 20-50 connections per application instance is a good start.

### 6. Common SQL Databases

| Database | Strengths | Used By |
|----------|-----------|---------|
| **PostgreSQL** | Advanced features, JSON support, extensible, strong consistency | Instagram, Reddit, Stripe |
| **MySQL** | Simple, fast reads, proven at scale, wide ecosystem | Facebook, Twitter, Uber, Airbnb |
| **Amazon Aurora** | MySQL/PostgreSQL compatible, auto-scaling, managed, 5x MySQL throughput | Many AWS-based companies |

---

## Real-World Example — Instagram's Database

```
Instagram uses PostgreSQL for core data:

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ users          │  │ photos        │  │ follows       │
├──────────────┤  ├──────────────┤  ├──────────────┤
│ id (PK)       │  │ id (PK)       │  │ id (PK)       │
│ username (UQ) │  │ user_id (FK)  │  │ follower_id   │
│ email (UQ)    │  │ caption       │  │ followee_id   │
│ password_hash │  │ image_url     │  │ created_at    │
│ bio           │  │ filter_type   │  └──────────────┘
│ avatar_url    │  │ created_at    │
│ created_at    │  │ like_count    │  Index: (follower_id, followee_id) UNIQUE
└──────────────┘  │ comment_count │  Index: (followee_id) for "who follows me"
                   └──────────────┘

Key decisions:
• PostgreSQL for ACID guarantees (double-likes prevention)
• Denormalized like_count/comment_count on photos (avoid COUNT queries)
• Sharded by user_id (all of a user's photos on same shard)
• Read replicas for feed generation queries
```

---

## Common Interview Questions

### Q1: "When would you choose SQL over NoSQL?"

**Model answer:**
"I'd choose SQL when I need ACID transactions — for example, payment processing where I need atomicity, or inventory management where I need strong consistency. SQL is also better when I have structured, relational data with well-defined schemas — users, orders, products with clear relationships. SQL's JOINs are powerful for complex queries. I'd lean toward NoSQL when I have unstructured data, need flexible schemas, or need to scale writes horizontally with less friction."

### Q2: "How would you design a schema for a URL shortener?"

**Model answer:**
```sql
CREATE TABLE urls (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code  VARCHAR(7) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at  TIMESTAMP,
    click_count BIGINT DEFAULT 0
);

-- Index for redirect lookups (most frequent query)
CREATE INDEX idx_short_code ON urls(short_code);

-- Index for user's URLs
CREATE INDEX idx_user_id ON urls(user_id);
```

### Q3: "What is the N+1 query problem?"

**Model answer:**
"The N+1 problem occurs when you fetch a list of N items and then make a separate query for each item's related data. For example, fetching 20 tweets then making 20 separate queries to get each author's name — that's 1 + 20 = 21 queries instead of 1 query with a JOIN or an IN clause. The fix is to batch: `SELECT * FROM users WHERE id IN (1, 2, 3, ...)` or use a JOIN. In system design, this matters because N+1 queries multiply latency and overload the database."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| No indexes mentioned | Always specify indexes for your most common queries |
| Over-normalizing everything | Normalize for writes, denormalize for reads. Pragmatic > pure. |
| "SQL can't scale" | SQL scales very well with read replicas, connection pooling, and careful indexing. Sharding is harder but doable (Vitess, Citus). |
| Ignoring connection pooling | At scale, connection management is critical. Always mention pooling. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 6 CHEAT SHEET                         │
│                  SQL Databases                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ACID: Atomicity, Consistency, Isolation, Durability            │
│  Use SQL when: structured data, relationships, ACID needed      │
│                                                                  │
│  INDEXES:                                                        │
│  • B-Tree: range queries, sorting, equality (default)           │
│  • Hash: exact lookups only (O(1))                              │
│  • Composite: (col1, col2) — leftmost prefix rule              │
│  • Always index your WHERE/JOIN/ORDER BY columns                │
│                                                                  │
│  NORMALIZATION vs DENORMALIZATION:                                │
│  • Normalize: no duplicate data, complex JOINs                  │
│  • Denormalize: duplicate data, fast reads, harder updates      │
│  • Interview: "Start normalized, selectively denormalize"       │
│                                                                  │
│  COMMON DATABASES:                                               │
│  PostgreSQL: advanced features, JSON, extensible                │
│  MySQL: simple, fast reads, proven at massive scale             │
│  Aurora: managed MySQL/PostgreSQL, auto-scaling, 5x perf        │
│                                                                  │
│  CAPACITY: Single MySQL: 1K-10K QPS, up to ~10TB               │
│  Connection pool: 20-50 connections per app instance             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What does ACID stand for? | Atomicity, Consistency, Isolation, Durability |
| 2 | What's the difference between normalization and denormalization? | Normalization eliminates data redundancy (clean but slow JOINs). Denormalization adds redundancy for read performance (fast but harder updates). |
| 3 | What index type is best for range queries? | B-Tree index — it stores data in sorted order, making range scans efficient |
| 4 | What is the leftmost prefix rule? | A composite index on (A, B, C) can be used for queries on (A), (A, B), or (A, B, C), but NOT (B) or (C) alone |
| 5 | What's the N+1 query problem? | Fetching N items then making N additional queries for related data. Fix with JOINs or batched IN queries. |
| 6 | When should you denormalize? | Read-heavy workloads, avoiding expensive JOINs at scale, data that changes rarely |
| 7 | What is connection pooling? | Pre-creating database connections and reusing them across requests, avoiding the overhead of creating a new connection per request |
| 8 | PostgreSQL vs MySQL — when to use each? | PostgreSQL for complex queries, JSON support, and advanced features. MySQL for simple, high-throughput read-heavy workloads. |
| 9 | What isolation level do most systems use? | Read Committed (PostgreSQL default) or Repeatable Read (MySQL default) |
| 10 | How many QPS can a single MySQL instance handle? | 1K-10K QPS depending on query complexity, indexing, and hardware |

---

[← Previous: Interview Framework](../phase-1-scalability/chapter-05-interview-framework.md) · [Next: NoSQL Databases →](./chapter-07-nosql-databases.md)
