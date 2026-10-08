# 🔢 Latency Numbers & Quick Reference

> **This reference is used throughout the course. Bookmark it.**
> These numbers are approximate and meant for back-of-the-envelope estimation, not exact benchmarking.

---

## ⚡ Latency Numbers Every Programmer Should Know

```
┌─────────────────────────────────────────────────────────────────┐
│                 LATENCY NUMBERS (Approximate, 2024)             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MEMORY & CPU                                                   │
│  ─────────────────────────────────────────────────────          │
│  L1 cache reference .......................... 0.5 ns           │
│  Branch mispredict ........................... 5 ns             │
│  L2 cache reference .......................... 7 ns             │
│  Mutex lock/unlock ........................... 25 ns            │
│  Main memory reference ....................... 100 ns           │
│                                                                 │
│  STORAGE                                                        │
│  ─────────────────────────────────────────────────────          │
│  Read 4 KB randomly from SSD ................. 150,000 ns      │
│  Read 1 MB sequentially from memory .......... 250,000 ns      │
│  Read 1 MB sequentially from SSD ............. 1,000,000 ns    │
│  Read 1 MB sequentially from HDD ............. 20,000,000 ns   │
│  HDD seek ................................... 10,000,000 ns     │
│                                                                 │
│  NETWORK                                                        │
│  ─────────────────────────────────────────────────────          │
│  Send 1 KB over 1 Gbps network ............... 10,000 ns       │
│  Round trip within same datacenter ............ 500,000 ns      │
│  Round trip CA → Netherlands ................. 150,000,000 ns   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Latency in Human-Readable Terms

| Operation | Latency | Analogy (if 1 ns = 1 second) |
|-----------|---------|------------------------------|
| L1 cache ref | 0.5 ns | Half a second |
| L2 cache ref | 7 ns | 7 seconds |
| Main memory ref | 100 ns | ~2 minutes |
| SSD random read | 150 μs | ~2 days |
| HDD seek | 10 ms | ~4 months |
| Intra-datacenter round trip | 500 μs | ~6 days |
| CA → Netherlands round trip | 150 ms | ~5 years |

**Key insight:** Memory is ~1000x faster than SSD. SSD is ~10x faster than HDD. Network within a datacenter is comparable to SSD. Cross-continent adds 150ms — this is why CDNs exist.

---

## 📐 Powers of 2

```
┌────────────────────────────────────────────────────┐
│                    POWERS OF 2                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  2^7  = 128                                        │
│  2^8  = 256                                        │
│  2^10 = 1,024           ≈ 1 Thousand   = 1 KB      │
│  2^16 = 65,536          ≈ 65 Thousand               │
│  2^20 = 1,048,576       ≈ 1 Million    = 1 MB      │
│  2^30 = 1,073,741,824   ≈ 1 Billion   = 1 GB      │
│  2^32 = 4,294,967,296   ≈ 4 Billion                │
│  2^40 = 1,099,511,627,776 ≈ 1 Trillion = 1 TB      │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Quick Size Reference

| Data Type | Typical Size |
|-----------|-------------|
| A character (ASCII) | 1 byte |
| A character (Unicode/UTF-8 avg) | 2-4 bytes |
| A UUID | 16 bytes |
| A tweet (280 chars) | ~560 bytes |
| A typical JSON API response | 1-10 KB |
| A photo (compressed JPEG) | 200 KB - 2 MB |
| A minute of HD video | ~100 MB |
| A minute of 4K video | ~350 MB |

---

## 📅 Time Estimates

```
┌────────────────────────────────────────────────────┐
│                 TIME ESTIMATES                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  1 second   = 1 second                             │
│  1 minute   = 60 seconds                           │
│  1 hour     = 3,600 seconds                        │
│  1 day      = 86,400 seconds    ≈ 100K seconds     │
│  1 week     = 604,800 seconds   ≈ 600K seconds     │
│  1 month    = 2,592,000 seconds ≈ 2.5M seconds     │
│  1 year     = 31,536,000 seconds ≈ 30M seconds     │
│                                                    │
│  SHORTCUT: seconds in a day ≈ 10^5                 │
│  SHORTCUT: seconds in a year ≈ 3 × 10^7           │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 📊 Common Scale Numbers

These are order-of-magnitude estimates for common services. Use them as starting points for estimation problems:

```
┌──────────────────────────────────────────────────────────────┐
│                    SCALE REFERENCE                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Twitter:                                                    │
│    DAU: ~250M    Tweets/day: ~500M    Tweet reads/day: ~50B  │
│    Read:Write = 100:1                                        │
│                                                              │
│  YouTube:                                                    │
│    DAU: ~1B    Videos uploaded/min: 500 hrs                   │
│    Video views/day: ~5B                                      │
│                                                              │
│  WhatsApp:                                                   │
│    DAU: ~2B    Messages/day: ~100B                            │
│                                                              │
│  Google Search:                                              │
│    Queries/day: ~8.5B    Queries/second: ~100K               │
│                                                              │
│  Instagram:                                                  │
│    DAU: ~500M    Photos uploaded/day: ~100M                   │
│                                                              │
│  Uber:                                                       │
│    Rides/day: ~20M    Active drivers: ~5M                    │
│                                                              │
│  Netflix:                                                    │
│    Subscribers: ~250M    Hours streamed/day: ~450M           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 🧮 Quick QPS Calculation Template

```
Given:
  DAU = X million
  Each user does Y actions/day

Step 1: Total daily actions = X × Y
Step 2: QPS = Total daily actions / 86,400
Step 3: Peak QPS = QPS × 2 (or ×3 for spiky workloads)

Example — Twitter Tweets:
  DAU = 250M, avg tweets/user/day = 2
  Total tweets/day = 500M
  QPS = 500M / 86,400 ≈ 5,800 tweets/sec
  Peak QPS ≈ 12,000 tweets/sec
```

---

## 🧮 Quick Storage Calculation Template

```
Given:
  New items/day = N
  Avg item size = S bytes
  Retention = R years

Storage needed:
  Daily = N × S
  Yearly = Daily × 365
  Total = Yearly × R

Example — Twitter Tweets (text only):
  500M tweets/day × 560 bytes = 280 GB/day
  280 GB × 365 = ~100 TB/year
  5 years = ~500 TB (text only, before media)
```

---

## 🧮 Quick Bandwidth Calculation Template

```
Given:
  QPS = Q requests/sec
  Avg response size = S bytes

Bandwidth = Q × S bytes/sec

Example — Twitter read API:
  Read QPS = 500,000 req/sec
  Avg response = 5 KB (timeline fetch is larger)
  Bandwidth = 500K × 5 KB = 2.5 GB/sec = 20 Gbps
```

---

## 📏 Rule of Thumb — Machine Capacity

```
┌────────────────────────────────────────────────────┐
│           SINGLE SERVER CAPACITY (2024)             │
├────────────────────────────────────────────────────┤
│                                                    │
│  Web server: 1K–10K QPS (depends on logic)         │
│  Redis: 100K–200K ops/sec                          │
│  MySQL: 1K–10K QPS (depends on queries)            │
│  Cassandra: 10K–50K ops/sec per node               │
│  Kafka: 100K–200K msgs/sec per broker              │
│  Elasticsearch: 10K–50K queries/sec per node       │
│                                                    │
│  Typical RAM: 64–256 GB                            │
│  Typical SSD: 1–4 TB                               │
│  Typical network: 10–25 Gbps                       │
│                                                    │
│  RULE OF THUMB:                                    │
│  If you need > 10K QPS → multiple servers          │
│  If you need > 1 TB RAM → sharding/partitioning    │
│  If you need < 1ms latency → caching               │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## 🔗 Availability Quick Reference

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|-------------|--------------|----------------|---------------|
| 99% (two 9s) | 3.65 days | 7.3 hrs | 1.68 hrs |
| 99.9% (three 9s) | 8.77 hrs | 43.8 min | 10.1 min |
| 99.99% (four 9s) | 52.6 min | 4.38 min | 1.01 min |
| 99.999% (five 9s) | 5.26 min | 26.3 sec | 6.05 sec |

**Interview tip:** Most systems target 99.9% (three 9s) or 99.99% (four 9s). Five 9s is extremely expensive and only for critical infrastructure (DNS, payment processing).

---

[← Back to Course Home](./README.md)
