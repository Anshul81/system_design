# Chapter 41: Design Ad Click Aggregation

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3 hours · Difficulty: 🟡 Medium · Type: 🏗️ DESIGN

[← Previous: Payment System](./chapter-40-payment-system.md) · [Next: Key-Value Store →](./chapter-42-key-value-store.md)

---

## Why This Problem Matters

Ad click aggregation is the revenue engine behind Google, Meta, and every ad-supported platform. It tests real-time stream processing, data aggregation at massive scale, and time-windowed counting. The core challenge: counting billions of clicks per day in near-real-time while handling duplicates, late arrivals, and providing accurate billing to advertisers.

---

## Step 1: Requirements

### Functional
| Feature | Description |
|---------|-------------|
| **Click tracking** | Record every ad click with metadata |
| **Real-time aggregation** | Clicks aggregated per ad, per minute |
| **Query API** | Advertisers query click counts for their campaigns |
| **Filtering** | Filter by ad_id, campaign_id, time range, region |
| **Billing** | Accurate click counts for advertiser billing |

### Non-Functional
| Requirement | Target |
|-------------|--------|
| **Click events/day** | 10B |
| **Peak QPS** | 200K clicks/sec |
| **Aggregation latency** | < 1 minute from click to queryable |
| **Accuracy** | No double-counting (deduplication) |
| **Data retention** | Raw: 7 days, aggregated: years |

---

## Step 2: Estimation

```
CLICK EVENTS:
  10B clicks/day ÷ 86,400 ≈ 115K clicks/sec
  Peak: ~200K clicks/sec

EVENT SIZE:
  Each click event: ~200 bytes
  (ad_id, user_id, timestamp, ip, device, referrer, geo)
  Daily raw data: 10B × 200B = 2 TB/day

AGGREGATED DATA:
  10M unique ads × 1440 minutes/day = 14.4B aggregated rows/day
  But most ads have 0 clicks in most minutes → sparse
  Realistic: ~500M non-zero aggregated rows/day × 50 bytes = 25 GB/day
  
QUERY LOAD:
  Advertiser dashboards: ~10K queries/sec
```

---

## Step 3: High-Level Design

### Architecture

```
┌────────────┐     ┌──────────────┐     ┌──────────────────────────┐
│ Ad on       │     │ Click         │     │ Kafka                     │
│ Publisher   │────▶│ Receiver      │────▶│ (click events topic)      │
│ Page        │     │ Service       │     │ (partitioned by ad_id)    │
└────────────┘     │ (validate,    │     └────────────┬─────────────┘
                    │  dedup, log)  │                   │
                    └──────────────┘                   │
                                            ┌─────────┼──────────┐
                                            ▼         ▼          ▼
                                     ┌───────────┐ ┌──────────┐ ┌──────────┐
                                     │ Aggregation│ │ Raw Data │ │ Fraud    │
                                     │ Service    │ │ Archiver │ │ Detection│
                                     │ (Flink /   │ │ (→ S3)   │ │ Service  │
                                     │  Spark     │ │          │ │          │
                                     │  Streaming)│ └──────────┘ └──────────┘
                                     └─────┬─────┘
                                           │
                                     ┌─────▼──────────┐
                                     │ Aggregation DB   │
                                     │ (OLAP: ClickHouse│
                                     │  or Druid)       │
                                     └─────┬──────────┘
                                           │
                                     ┌─────▼──────────┐
                                     │ Query Service    │
                                     │ (advertiser      │
                                     │  dashboard API)  │
                                     └────────────────┘
```

### API Design

```
POST /api/v1/click
  Request:  { "ad_id": "ad_123",
              "impression_id": "imp_456",     // for dedup
              "user_id": "u_789",
              "timestamp": 1695600000,
              "ip": "203.0.113.1",
              "user_agent": "...",
              "referrer": "https://publisher.com/article" }
  Response: HTTP 204 No Content (fire-and-forget)

GET /api/v1/ads/{ad_id}/clicks?from=2024-01-01T00:00&to=2024-01-01T23:59&granularity=1h
  Response: { "ad_id": "ad_123",
              "data": [
                { "timestamp": "2024-01-01T00:00", "clicks": 15234 },
                { "timestamp": "2024-01-01T01:00", "clicks": 12891 },
                ...
              ],
              "total": 324567 }

GET /api/v1/campaigns/{campaign_id}/clicks?from=...&to=...&group_by=region
  Response: { "campaign_id": "camp_456",
              "data": [
                { "region": "US", "clicks": 234567 },
                { "region": "EU", "clicks": 189012 },
                ...
              ] }
```

---

## Step 4: Detailed Design

### Deep Dive 1: Stream Processing Pipeline

```
AGGREGATION WITH TUMBLING WINDOWS:

Raw click events flow through Kafka → Flink/Spark Streaming

Flink job:
  1. Consume from Kafka topic (partitioned by ad_id)
  2. Deduplicate: check impression_id against Redis SET
     → If seen → drop (duplicate click)
     → If new → SADD to Redis SET with 1-hour TTL
  3. Tumbling window: aggregate per (ad_id, 1-minute window)
  4. Emit: { ad_id, window_start, click_count, unique_users }
  5. Write aggregated row to ClickHouse

WINDOW TYPES:
  Tumbling: [0:00-0:01) [0:01-0:02) [0:02-0:03) ...
  Each event belongs to exactly one window.
  Window closes at end → emits aggregate.

  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │ Window 1 │ │ Window 2 │ │ Window 3 │
  │ 0:00-0:01│ │ 0:01-0:02│ │ 0:02-0:03│
  │ clicks: 5│ │ clicks: 3│ │ clicks: 7│
  └─────────┘ └─────────┘ └─────────┘
       ↓            ↓           ↓
  Write to DB   Write to DB  Write to DB

LATE ARRIVALS:
  What if a click at 0:00:58 arrives at 0:01:03?
  → Watermark: allow 5-second late arrivals
  → Events arriving within watermark update the correct window
  → Events arriving after watermark → dropped or sent to DLQ
```

### Deep Dive 2: Deduplication

```
CLICK FRAUD & DUPLICATE PREVENTION:

Layer 1: Client-side dedup (impression_id)
  Each ad impression generates a unique impression_id.
  Click carries this ID. If same impression_id clicked twice → dedup.

Layer 2: Redis-based dedup (streaming layer)
  Redis SET per time bucket:
    Key: "clicks:{minute_bucket}" 
    SADD impression_id → returns 1 (new) or 0 (duplicate)
    TTL: 1 hour (clicks older than 1 hour can't be dupes)

Layer 3: Fraud detection (async)
  ML model flags suspicious patterns:
    - Same user clicking same ad 100 times
    - Same IP generating thousands of clicks
    - Click-through rate anomalies
    - Known bot user agents
  
  Flagged clicks → quarantined, not counted for billing
  Advertisers not charged for fraudulent clicks

DEDUP ACCURACY:
  At 200K clicks/sec, Redis SET memory:
  200K × 60 sec × 60 min × 36 bytes/key ≈ 2.6 GB per hour bucket
  → Fits easily in a single Redis instance
```

### Deep Dive 3: Storage & Query Optimization

```
STORAGE: ClickHouse (Column-Oriented OLAP Database)

Why ClickHouse?
  - Column-oriented: "SUM(clicks) WHERE ad_id=X AND date BETWEEN..." is fast
  - Compression: 10:1 compression ratio on time-series data
  - MergeTree engine: optimized for time-series inserts
  - Handles 200K+ inserts/sec easily

TABLE SCHEMA:
  CREATE TABLE ad_clicks_1min (
      ad_id        UInt64,
      campaign_id  UInt64,
      window_start DateTime,
      click_count  UInt32,
      unique_users UInt32,
      region       String
  ) ENGINE = MergeTree()
  PARTITION BY toYYYYMMDD(window_start)
  ORDER BY (ad_id, window_start);

MULTI-RESOLUTION ROLLUPS:
  1-minute → 1-hour → 1-day → 1-month

  Materialized views auto-aggregate:
    1-min data → retained 7 days
    1-hour rollup → retained 90 days  
    1-day rollup → retained forever

  Dashboard query for "last 30 days" hits 1-hour rollup (720 rows)
  Dashboard query for "last hour" hits 1-min data (60 rows)
  → Always fast, regardless of time range

RAW DATA ARCHIVAL:
  Raw click events (Kafka) → S3 in Parquet format
  Retained for 7 days for debugging / reprocessing
  If aggregation pipeline has a bug → replay from S3
```

### Deep Dive 4: Exactly-Once Semantics

```
THE CHALLENGE: Flink/Spark crashes mid-aggregation

Without exactly-once:
  Window emits aggregate → writes to ClickHouse → Flink crashes
  Flink restarts → replays Kafka from last checkpoint
  → Same aggregate written to ClickHouse AGAIN → double-counted!

SOLUTION: Flink Checkpointing + Idempotent Writes

1. Flink checkpoints:
   Periodically snapshot state (window contents) to durable storage
   On crash → restore from last checkpoint → replay only unprocessed events

2. Idempotent writes to ClickHouse:
   Each aggregate has a deterministic key: (ad_id, window_start)
   Use ReplacingMergeTree → deduplicates rows with same key
   Even if written twice, only one copy retained

3. Kafka consumer offsets committed WITH Flink checkpoint:
   → Ensures no events are lost or double-processed
   → End-to-end exactly-once semantics

GUARANTEE:
  Every click counted exactly once in aggregated data.
  Critical for billing accuracy.
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   AD CLICK AGGREGATION SYSTEM                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐   click    ┌───────────────┐                       │
│  │ Publisher    │──event────▶│ Click Receiver │                       │
│  │ (ad on page)│            │ (validate,     │                       │
│  └─────────────┘            │  basic dedup)  │                       │
│                              └───────┬───────┘                       │
│                                      │                               │
│                              ┌───────▼───────┐                       │
│                              │ Kafka          │                       │
│                              │ (partitioned   │                       │
│                              │  by ad_id)     │                       │
│                              └───┬───────┬───┘                       │
│                                  │       │                           │
│                    ┌─────────────┘       └──────────────┐            │
│                    ▼                                     ▼            │
│             ┌──────────────┐                    ┌──────────────┐     │
│             │ Flink/Spark   │                    │ Raw Archiver  │     │
│             │ Streaming     │                    │ → S3 Parquet  │     │
│             │               │                    └──────────────┘     │
│             │ • Dedup (Redis)│                                        │
│             │ • 1-min window │                    ┌──────────────┐     │
│             │ • Aggregate    │                    │ Fraud ML      │     │
│             └──────┬────────┘                    │ Service       │     │
│                    │                              └──────────────┘     │
│             ┌──────▼────────┐                                        │
│             │ ClickHouse     │    ┌──────────────┐                    │
│             │ (1-min, 1-hr,  │───▶│ Query Service │                    │
│             │  1-day rollups)│    │ (advertiser   │                    │
│             └───────────────┘    │  dashboard)   │                    │
│                                   └──────────────┘                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Streaming engine | Flink/Spark Streaming | Custom consumers | Flink provides windowing, watermarks, exactly-once, and state management built-in. |
| Aggregation DB | ClickHouse (column OLAP) | PostgreSQL, Cassandra | Column-oriented is 10-100x faster for aggregation queries. Built for analytics. |
| Deduplication | Redis SET + impression_id | Bloom filter | Redis SET is exact (no false positives). Critical for billing accuracy. Bloom filter would have false positives. |
| Partitioning | By ad_id in Kafka | By timestamp | Same ad's clicks go to same partition → easy per-ad aggregation without shuffles. |
| Multi-resolution | Materialized rollups | Query-time aggregation | Pre-computed rollups make dashboard queries O(rows in rollup), not O(all raw events). |
| Raw data retention | 7 days in S3 Parquet | Don't keep raw | Enables reprocessing if aggregation pipeline has a bug. Parquet is cheap and queryable. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you aggregate 200K clicks/sec in near-real-time? | Stream processing (Flink/Spark): consume from Kafka, tumbling 1-minute windows, aggregate per ad_id, write to column OLAP DB (ClickHouse). |
| 2 | How do you handle duplicate clicks? | Impression_id dedup: Redis SET per time bucket. SADD returns 0 if already seen → drop duplicate. TTL auto-cleans old entries. |
| 3 | What is a tumbling window? | A fixed-size, non-overlapping time window. Each event belongs to exactly one window. Window closes at end and emits the aggregate. |
| 4 | How do you handle late-arriving events? | Watermark: allow events up to N seconds late. Late events update the correct window. Events past the watermark are dropped or sent to a dead letter queue. |
| 5 | Why use ClickHouse over MySQL for aggregation queries? | ClickHouse is column-oriented: "SUM clicks WHERE ad_id=X" only reads the clicks column, not entire rows. 10-100x faster for analytics. |
| 6 | How do you achieve exactly-once counting? | Flink checkpointing + idempotent writes. Flink snapshots state periodically. On crash, replay from checkpoint. ClickHouse ReplacingMergeTree deduplicates writes. |
| 7 | What is multi-resolution rollup? | Pre-aggregate 1-min → 1-hour → 1-day. Dashboard queries hit the appropriate resolution. "Last hour" queries 60 rows, "last month" queries 720 rows. Always fast. |
| 8 | How is click fraud detected? | Async ML model flags suspicious patterns: same user clicking repeatedly, same IP generating thousands of clicks, bot user agents, CTR anomalies. Flagged clicks not billed. |

---

[← Previous: Payment System](./chapter-40-payment-system.md) · [Next: Key-Value Store →](./chapter-42-key-value-store.md)
