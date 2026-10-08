# Chapter 7: NoSQL Databases — Types & Trade-offs

> **Phase 2 — Databases & Storage** · Estimated Time: 3 hours · Difficulty: 🟡 Medium

[← Previous: SQL Databases](./chapter-06-sql-databases.md) · [Next: Database Scaling →](./chapter-08-replication-sharding.md)

---

## Learning Objectives

- ✅ Identify the 4 main types of NoSQL databases and their use cases
- ✅ Explain the data model and access patterns for each type
- ✅ Choose the right NoSQL database for a given system design problem
- ✅ Understand the trade-offs NoSQL makes compared to SQL
- ✅ Know when Redis, MongoDB, Cassandra, and DynamoDB are appropriate

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 6 (SQL Databases) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 2 hrs |
| Comparison exercises | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** SQL databases are like filing cabinets with strictly organized folders — great for structured data, but rigid. NoSQL databases are like different storage solutions for different needs: a bookshelf (document DB), a dictionary (key-value), a spreadsheet with unlimited columns (wide-column), and a spider web (graph). Each is optimized for specific access patterns.

**Interview relevance:** In every system design, you'll choose at least one database. The interviewer wants to hear WHY you chose it, not just what it is. "I'd use Redis for caching because it gives sub-millisecond reads from memory, and Cassandra for the message store because it handles high write throughput with tunable consistency" — that's what gets you points.

---

## Visual Diagram — NoSQL Database Types

```
┌─────────────────────────────────────────────────────────────────┐
│                     NoSQL DATABASE TYPES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  KEY-VALUE                    DOCUMENT                           │
│  ─────────                    ────────                           │
│  key → value                  key → JSON document                │
│                                                                  │
│  "user:123" → {blob}          "user:123" → {                    │
│  "session:abc" → {blob}         "name": "John",                 │
│  "cache:page:1" → {blob}        "email": "j@x.com",            │
│                                   "posts": [                     │
│  Examples:                          {"title": "Hello"},          │
│  Redis, Memcached,                  {"title": "World"}           │
│  DynamoDB                         ]                              │
│                                 }                                │
│                                                                  │
│                               Examples:                          │
│                               MongoDB, CouchDB,                 │
│                               Firestore                          │
│                                                                  │
│  ────────────────────────────────────────────────────────        │
│                                                                  │
│  WIDE-COLUMN                  GRAPH                              │
│  ───────────                  ─────                              │
│  row_key → {col_family: {     (Alice)──[FOLLOWS]──▶(Bob)        │
│    col1: val, col2: val}}       │                    │           │
│                                  └──[LIKES]──▶(Post1)            │
│  row: "user:123"                                                 │
│    profile: {name, email}     Examples:                          │
│    activity: {last_login}     Neo4j, Amazon Neptune,             │
│    metrics: {tweet_count}     JanusGraph                         │
│                                                                  │
│  Examples:                                                       │
│  Cassandra, HBase,                                               │
│  Google Bigtable                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### 1. Key-Value Stores

The simplest NoSQL model: a giant hash map. Every piece of data has a unique key and an opaque value.

```
Operations:
  GET(key) → value
  SET(key, value)
  DELETE(key)

That's it. No queries, no joins, no filtering by value.
```

| Database | Specialty | Used By |
|----------|-----------|---------|
| **Redis** | In-memory, sub-ms latency, data structures (lists, sets, sorted sets, hashes) | Twitter, GitHub, Stack Overflow, Instagram |
| **Memcached** | Pure in-memory cache, simpler than Redis, multi-threaded | Facebook, Wikipedia |
| **DynamoDB** | Managed, serverless, single-digit ms latency, auto-scaling | Amazon, Lyft, Samsung |
| **etcd** | Distributed key-value for config/service discovery | Kubernetes |

**Redis Data Structures (Important for Interviews):**
```
STRING:  SET user:123:name "John"
         GET user:123:name → "John"

HASH:    HSET user:123 name "John" email "j@x.com"
         HGET user:123 name → "John"

LIST:    LPUSH user:123:feed tweet_456 tweet_789
         LRANGE user:123:feed 0 19 → [latest 20 tweets]

SET:     SADD user:123:followers 456 789
         SISMEMBER user:123:followers 456 → true

SORTED SET: ZADD leaderboard 1500 "player_1" 2000 "player_2"
            ZRANGE leaderboard 0 9 → [top 10 players]

Use cases in system design:
  • Cache (STRING/HASH)
  • Session store (HASH)
  • Rate limiting (STRING with INCR + EXPIRE)
  • Leaderboards (SORTED SET)
  • Real-time analytics (sorted sets, HyperLogLog)
  • Message queue (LIST with LPUSH/BRPOP)
  • Pub/Sub for WebSocket routing
```

**When to use key-value:**
- ✅ Caching (most common use)
- ✅ Session storage
- ✅ Rate limiting counters
- ✅ Leaderboards and rankings
- ✅ Real-time data that needs sub-ms access
- ❌ Complex queries (no SQL-like filtering)
- ❌ Relationships between data (no JOINs)

---

### 2. Document Databases

Store data as JSON-like documents. Each document can have a different structure (flexible schema).

```
MongoDB Document Example:

{
  "_id": ObjectId("64a1b2c3d4e5f6"),
  "username": "johndoe",
  "email": "john@example.com",
  "profile": {
    "bio": "Software engineer",
    "location": "San Francisco",
    "avatar_url": "https://..."
  },
  "posts": [
    {
      "title": "My First Post",
      "content": "Hello world!",
      "tags": ["intro", "tech"],
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "followers_count": 1234,
  "created_at": "2023-06-01T00:00:00Z"
}
```

| Database | Specialty | Used By |
|----------|-----------|---------|
| **MongoDB** | General-purpose, flexible schema, rich query language | Uber, eBay, Coinbase |
| **CouchDB** | Multi-master replication, eventual consistency | IBM, Apple |
| **Firestore** | Real-time sync, managed, mobile-first | Google, mobile apps |

**When to use document DB:**
- ✅ Flexible/evolving schemas (startup MVPs, content management)
- ✅ Hierarchical data (nested objects, arrays)
- ✅ Data accessed as whole documents (user profiles, product catalogs)
- ❌ Highly relational data requiring JOINs
- ❌ Multi-document transactions (improving but weaker than SQL)

---

### 3. Wide-Column Databases

Think of it as a two-dimensional key-value store. Data is organized into rows and column families, but each row can have different columns.

```
Cassandra Data Model:

Table: user_tweets
Partition Key: user_id
Clustering Key: created_at (DESC)

┌───────────┬────────────┬──────────────┬──────────────────┐
│ user_id   │ created_at │ tweet_id     │ content           │
│ (PK)      │ (CK DESC)  │              │                   │
├───────────┼────────────┼──────────────┼──────────────────┤
│ user_123  │ 2024-01-15 │ tweet_789    │ "Hello world"     │
│ user_123  │ 2024-01-14 │ tweet_788    │ "Good morning"    │
│ user_123  │ 2024-01-13 │ tweet_787    │ "Weekend vibes"   │
├───────────┼────────────┼──────────────┼──────────────────┤
│ user_456  │ 2024-01-15 │ tweet_790    │ "First tweet"     │
└───────────┴────────────┴──────────────┴──────────────────┘

Query: SELECT * FROM user_tweets
       WHERE user_id = 'user_123'
       ORDER BY created_at DESC
       LIMIT 20;

→ Returns user_123's latest 20 tweets. FAST because
  data is physically sorted by created_at on disk.
```

| Database | Specialty | Used By |
|----------|-----------|---------|
| **Cassandra** | Write-heavy, high availability, tunable consistency, linear scalability | Netflix, Apple, Instagram (DMs), Discord |
| **HBase** | Hadoop integration, strong consistency per row | Facebook (Messages, pre-2020), Yahoo |
| **Google Bigtable** | Managed, integrates with Google Cloud, petabyte-scale | Google (Search, Maps, Gmail), Spotify |

**When to use wide-column:**
- ✅ Write-heavy workloads (time-series data, event logging, IoT)
- ✅ Data with a clear partition key (user_id, device_id)
- ✅ Time-ordered data (messages, activity feeds, metrics)
- ✅ Need to scale writes linearly by adding nodes
- ❌ Complex ad-hoc queries (must design tables around query patterns)
- ❌ Multi-row transactions
- ❌ Data that needs flexible querying

**Cassandra's Key Insight: Model tables around queries, not entities.**
```
In SQL: Design tables first, then write queries
In Cassandra: Define queries first, then design tables

Need "get tweets by user"? → Table: user_tweets (PK: user_id)
Need "get tweets by hashtag"? → Table: hashtag_tweets (PK: hashtag)
Same data, different tables, different access patterns.
This is called "query-driven modeling."
```

---

### 4. Graph Databases

Store data as nodes (entities) and edges (relationships). Optimized for traversing relationships.

```
Social Network Graph:

    (Alice)──[FOLLOWS]──▶(Bob)──[FOLLOWS]──▶(Charlie)
       │                    │
       └──[LIKES]──▶(Post1) └──[POSTED]──▶(Post2)
                       │
                   [TAGGED]──▶(#tech)

Query: "Find all friends of Alice's friends"
  Cypher (Neo4j): MATCH (alice:User {name:'Alice'})
                  -[:FOLLOWS]->()-[:FOLLOWS]->(fof)
                  RETURN DISTINCT fof.name

This query traverses two relationship hops.
In SQL, this would require recursive JOINs — very expensive.
In a graph DB, it's a natural, efficient traversal.
```

| Database | Specialty | Used By |
|----------|-----------|---------|
| **Neo4j** | Most popular, Cypher query language, ACID | LinkedIn, NASA, Walmart |
| **Amazon Neptune** | Managed, supports Gremlin and SPARQL | AWS-based social networks |
| **JanusGraph** | Distributed, scales horizontally, open-source | Uber, IBM |

**When to use graph DB:**
- ✅ Social networks (friends-of-friends, mutual connections)
- ✅ Recommendation engines ("people who liked X also liked Y")
- ✅ Fraud detection (detecting patterns in transaction graphs)
- ✅ Knowledge graphs (entity relationships)
- ❌ Simple CRUD operations (overkill)
- ❌ High-throughput write workloads
- ❌ Tabular data with straightforward queries

---

### 5. Time-Series Databases

Specialized for time-stamped data — metrics, logs, IoT sensor data.

| Database | Used For |
|----------|----------|
| **InfluxDB** | Application metrics, IoT data |
| **TimescaleDB** | PostgreSQL extension, SQL compatibility with time-series optimization |
| **Prometheus** | Monitoring and alerting metrics |

**When to use:** Metrics dashboards, monitoring systems, IoT sensor data, financial data.

---

## Comparison Table — NoSQL Database Types

| Type | Data Model | Best For | Scalability | Query Flexibility | Example |
|------|-----------|----------|-------------|-------------------|---------|
| **Key-Value** | key → blob | Caching, sessions | Excellent | None (get/set only) | Redis, DynamoDB |
| **Document** | key → JSON doc | Flexible schemas, catalogs | Good | Moderate (filter on fields) | MongoDB |
| **Wide-Column** | row → column families | Time-series, write-heavy | Excellent | Limited (query-driven) | Cassandra |
| **Graph** | nodes + edges | Relationships, social | Moderate | Excellent for traversals | Neo4j |

---

## Real-World Example — Discord's Database Architecture

```
Discord handles 150M+ MAU with heavy read AND write loads:

┌─────────────────────────────────────────────────────┐
│                    Discord's Data Layer              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Messages:     Cassandra → ScyllaDB (2M writes/sec) │
│  (write-heavy, time-ordered, partition by channel)   │
│                                                     │
│  User Data:    PostgreSQL (ACID for accounts)       │
│  (structured, relational, needs consistency)         │
│                                                     │
│  Real-time:    Redis (presence, typing indicators)  │
│  (ephemeral, sub-ms latency needed)                  │
│                                                     │
│  Search:       Elasticsearch (message search)       │
│  (full-text search, inverted index)                  │
│                                                     │
└─────────────────────────────────────────────────────┘

Key insight: POLYGLOT PERSISTENCE.
Different data, different access patterns → different databases.
This is the standard approach in modern system design.
```

---

## Common Interview Questions

### Q1: "When would you use Redis vs Cassandra vs MongoDB?"

**Model answer:**
"Redis when I need sub-millisecond reads for caching, sessions, or real-time features — it's in-memory so it's blazing fast but data size is limited by RAM. Cassandra when I have a write-heavy workload with time-ordered data — like a message store or event log — because it's optimized for sequential writes and scales linearly by adding nodes. MongoDB when I need a flexible schema with rich querying — like a product catalog where each product might have different attributes — because it stores JSON documents and supports field-level queries."

### Q2: "What is polyglot persistence?"

**Model answer:**
"Polyglot persistence means using different database technologies for different data needs within the same system. For example, in a chat application, I'd use PostgreSQL for user accounts (ACID), Cassandra for messages (write-heavy, time-ordered), Redis for online presence and caching (low-latency), and Elasticsearch for message search (full-text). Each database is optimized for its specific access pattern. The trade-off is operational complexity — you're running 4 database systems instead of 1."

### Q3: "Why can't you just use MongoDB for everything?"

**Model answer:**
"MongoDB is versatile, but it makes trade-offs. It doesn't provide the same ACID guarantees as PostgreSQL across multiple documents, so it's risky for financial transactions. It's not as fast as Redis for caching because it reads from disk. It doesn't handle write-heavy time-series data as efficiently as Cassandra, which is optimized for sequential writes. And it doesn't handle relationship traversals as efficiently as a graph database. Every NoSQL database sacrifices something — the key is knowing what you're giving up."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "NoSQL is always faster than SQL" | NoSQL is faster for specific access patterns. SQL with proper indexing can be faster for complex queries. |
| "Redis replaces a database" | Redis is primarily an in-memory cache/store. It can persist data but isn't a primary database for most use cases. |
| Using MongoDB for write-heavy time-series data | Use Cassandra or a time-series DB. MongoDB's document model isn't optimized for append-heavy sequential writes. |
| "We'll use Cassandra because it scales" | Cassandra requires query-driven data modeling. If your access patterns don't fit, it's painful. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 7 CHEAT SHEET                         │
│                  NoSQL Databases                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  KEY-VALUE (Redis, DynamoDB):                                    │
│  → Caching, sessions, rate limiting, leaderboards               │
│  → Sub-ms reads, limited by RAM                                 │
│                                                                  │
│  DOCUMENT (MongoDB):                                             │
│  → Flexible schemas, product catalogs, CMS                      │
│  → Rich querying on JSON fields                                 │
│                                                                  │
│  WIDE-COLUMN (Cassandra, HBase):                                │
│  → Write-heavy, time-series, messages, events                   │
│  → Model tables around queries, not entities                    │
│  → Scales writes linearly                                       │
│                                                                  │
│  GRAPH (Neo4j):                                                  │
│  → Social networks, recommendations, fraud detection            │
│  → Efficient relationship traversal                             │
│                                                                  │
│  CAPACITY (per node):                                            │
│  Redis: 100K-200K ops/sec, limited by RAM                       │
│  Cassandra: 10K-50K ops/sec, scales linearly                    │
│  MongoDB: 10K-50K ops/sec depending on queries                  │
│                                                                  │
│  INTERVIEW DEFAULT:                                              │
│  Cache → Redis    Messages → Cassandra   Profiles → PostgreSQL  │
│  Search → Elasticsearch   Files → S3 + CDN                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 4 types of NoSQL databases? | Key-value (Redis), Document (MongoDB), Wide-column (Cassandra), Graph (Neo4j) |
| 2 | What Redis data structure would you use for a leaderboard? | Sorted Set (ZADD for scores, ZRANGE for top-N) |
| 3 | What is Cassandra's key design principle? | Model tables around queries, not entities. Design your table structure based on your access patterns. |
| 4 | When would you use a graph database? | Social networks (friends-of-friends), recommendation engines, fraud detection — anything requiring efficient relationship traversal |
| 5 | What is polyglot persistence? | Using different database technologies for different data needs within the same system |
| 6 | Redis vs Memcached — when to use each? | Redis: data structures, persistence, pub/sub, single-threaded. Memcached: pure cache, simpler, multi-threaded, slightly faster for simple get/set |
| 7 | Why is Cassandra good for write-heavy workloads? | Writes go to an in-memory table (memtable), then flush to disk as sorted files (SSTables). No random I/O for writes. |
| 8 | What's the main limitation of key-value stores? | No query flexibility — you can only get/set by key. Can't filter, sort, or query by value fields. |
| 9 | MongoDB vs PostgreSQL — when to choose MongoDB? | When schema is evolving rapidly, data is naturally hierarchical (JSON), and you don't need strong multi-document transactions |
| 10 | How many ops/sec can a single Redis instance handle? | 100K-200K operations per second (in-memory, single-threaded) |

---

[← Previous: SQL Databases](./chapter-06-sql-databases.md) · [Next: Database Scaling →](./chapter-08-replication-sharding.md)
