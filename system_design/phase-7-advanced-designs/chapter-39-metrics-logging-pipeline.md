# Chapter 39: Design a Metrics & Logging Pipeline

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3 hours · Difficulty: 🟡 Medium · Type: 🏗️ DESIGN

[← Previous: Collaborative Editing](./chapter-38-collaborative-editing.md) · [Next: Payment System →](./chapter-40-payment-system.md)

---

## Why This Problem Matters

Tests your ability to design a write-heavy, time-series system. The key challenge: ingesting millions of events per second with eventual queryability. This is Datadog/New Relic in a box.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Metrics ingestion** | 10M metrics/sec |
| **Log ingestion** | 1M log lines/sec |
| **Query latency** | < 5 sec for dashboard queries |
| **Retention** | 30 days (hot), 1 year (cold) |
| **Alerting** | < 1 minute alert trigger latency |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  METRICS & LOGGING PIPELINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Data Sources:                                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐             │
│  │Service A │ │Service B │ │Service C │ │Infra     │             │
│  │(metrics) │ │(logs)    │ │(traces)  │ │(host     │             │
│  │          │ │          │ │          │ │ metrics) │             │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘            │
│       │            │            │            │                   │
│       └────────────┼────────────┼────────────┘                   │
│                    ▼                                             │
│           ┌──────────────────┐                                   │
│           │  Kafka / Kinesis  │  (buffering & ordering)          │
│           │  (10M+ events/s) │                                   │
│           └────────┬─────────┘                                   │
│                    │                                             │
│        ┌───────────┼───────────┐                                │
│        ▼           ▼           ▼                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ Stream    │ │ Indexer   │ │ Alert     │                        │
│  │ Processor │ │ (logs →   │ │ Evaluator │                        │
│  │ (aggregate│ │  Elastic  │ │ (compare  │                        │
│  │  metrics) │ │  search)  │ │  vs       │                        │
│  └─────┬────┘ └─────┬────┘ │  thresholds│                        │
│        │            │      └──────┬────┘                        │
│        ▼            ▼             ▼                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ Time-     │ │ Elastic  │ │ PagerDuty │                        │
│  │ Series DB │ │ search   │ │ / Slack   │                        │
│  │ (metrics) │ │ (logs)   │ │ (alerts)  │                        │
│  │ InfluxDB  │ └──────────┘ └──────────┘                        │
│  │ or        │                                                   │
│  │ Prometheus│     ┌──────────────┐                              │
│  └─────┬────┘     │   Grafana     │  Dashboards & visualization │
│        └──────────▶│   / Kibana    │                              │
│                    └──────────────┘                              │
│                                                                  │
│  COLD STORAGE:                                                   │
│  After 30 days → downsample metrics → S3 (Parquet)             │
│  After 30 days → archive logs → S3 (compressed)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

**1. Time-Series Aggregation:**
```
Raw: 10M data points/sec (too much to query directly)
Aggregation:
  1-second resolution → 5-minute resolution → 1-hour resolution

Raw:     10M points/sec → stored for 24 hours
5-min:   Aggregated (avg, p50, p99, max) → stored for 30 days
1-hour:  Further aggregated → stored for 1 year

Dashboard queries hit pre-aggregated data → fast!
```

**2. Write Path Optimization:**
```
10M metrics/sec → can't write each individually to DB

Batch writes:
  Agent collects metrics locally for 10 seconds
  Sends batch of 1000+ metrics in one request
  Kafka buffers → Stream processor aggregates → batch write to TSDB

Write amplification reduced by 1000x.
```

**3. Hot vs Cold Storage:**
```
Hot (< 30 days): Time-series DB (InfluxDB, Prometheus)
  Fast queries for dashboards and alerting

Cold (> 30 days): S3 in Parquet format
  Cheap storage, queryable via Spark/Presto for ad-hoc analysis
  Downsampled (1-hour resolution instead of 1-second)
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Ingestion buffer | Kafka | Absorbs write spikes, decouples producers from consumers |
| Metrics storage | Time-series DB (InfluxDB) | Optimized for time-range queries with aggregation |
| Log storage | Elasticsearch | Full-text search on log content, good for debugging |
| Aggregation | Pre-computed (stream processing) | Real-time dashboards need pre-aggregated data. Can't scan 10M raw events per query. |
| Cold storage | S3 + Parquet | 100x cheaper than keeping everything in hot storage. Columnar format for analytics. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | Why pre-aggregate metrics? | 10M raw metrics/sec is too much to query directly. Pre-aggregate to 5-min and 1-hour resolutions for fast dashboard queries. |
| 2 | How do you handle 10M metrics/sec write throughput? | Batch at the agent (10s windows), buffer in Kafka, stream process for aggregation, batch write to TSDB. |
| 3 | Hot vs cold storage for metrics? | Hot (< 30 days): Time-series DB for fast queries. Cold (> 30 days): downsampled data in S3 Parquet for cheap archival. |
| 4 | Metrics vs logs storage? | Metrics: time-series DB (InfluxDB, Prometheus) — numeric, aggregatable. Logs: Elasticsearch — text, full-text searchable. |

---

[← Previous: Collaborative Editing](./chapter-38-collaborative-editing.md) · [Next: Payment System →](./chapter-40-payment-system.md)
