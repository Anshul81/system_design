# Chapter 4: Back-of-the-Envelope Estimation

> **Phase 1 — Scalability Fundamentals** · Estimated Time: 3 hours · Difficulty: 🟡 Medium

[← Previous: Scaling 101](./chapter-03-scaling-101.md) · [Next: The System Design Interview Framework →](./chapter-05-interview-framework.md)

---

## Learning Objectives

After this chapter, you will be able to:

- ✅ Calculate QPS (queries per second) for any system from DAU
- ✅ Estimate storage requirements for data-intensive systems
- ✅ Calculate bandwidth and memory needs
- ✅ Use powers of 2 and latency numbers confidently
- ✅ Perform estimation under interview time pressure (2-3 minutes)
- ✅ Know when to estimate and when to skip

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 3 (Scaling 101) | ✅ Yes |
| Basic math (multiplication, division) | ✅ Yes |
| [Latency Reference](../LATENCY_REFERENCE.md) | ✅ Review |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Worked examples | 1 hr |
| Practice problems | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** Before building a bridge, engineers estimate the weight it needs to support. They don't need exact numbers — they need to know if it's 100 tons or 100,000 tons. That order-of-magnitude difference completely changes the design. System design estimation works the same way.

**Interview relevance:** Interviewers expect you to spend 2-3 minutes on estimation BEFORE drawing architecture. It shows you can think quantitatively: "We need 10K QPS and 5 TB of storage" is very different from "We need 1M QPS and 500 TB of storage." The first might work with a single server. The second requires serious distributed systems design. Getting the order of magnitude right shapes your entire architecture.

---

## Visual Diagram — The Estimation Process

```
┌──────────────────────────────────────────────────────────────────┐
│                     ESTIMATION WORKFLOW                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. START WITH USERS                                             │
│     │                                                            │
│     ├── Total registered users: 500M                             │
│     ├── DAU (Daily Active Users): 250M (50% of registered)      │
│     └── Peak concurrent users: 25M (10% of DAU)                 │
│                                                                  │
│  2. CALCULATE QPS                                                │
│     │                                                            │
│     ├── Actions per user per day: 10 tweets read, 0.5 write     │
│     ├── Daily actions: 250M × 10 = 2.5B reads/day              │
│     ├── QPS = 2.5B / 86,400 ≈ 29,000 reads/sec                 │
│     └── Peak QPS = QPS × 2 = ~58,000 reads/sec                 │
│                                                                  │
│  3. CALCULATE STORAGE                                            │
│     │                                                            │
│     ├── New data per day: 125M tweets × 560 bytes = 70 GB/day  │
│     ├── With metadata + indexes: 70 GB × 3 = 210 GB/day        │
│     ├── Per year: 210 GB × 365 = ~77 TB/year                   │
│     └── 5-year retention: ~385 TB                                │
│                                                                  │
│  4. CALCULATE BANDWIDTH                                          │
│     │                                                            │
│     ├── Read bandwidth: 29K QPS × 5 KB avg = 145 MB/s          │
│     └── Write bandwidth: 1.4K QPS × 1 KB avg = 1.4 MB/s       │
│                                                                  │
│  5. CALCULATE MEMORY (Cache)                                     │
│     │                                                            │
│     ├── 80/20 rule: 20% of data generates 80% of traffic       │
│     └── Cache 20% of daily reads: 0.2 × 70 GB = 14 GB          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### 1. The QPS Calculation Formula

This is the most important formula in system design:

```
                   DAU × actions_per_user_per_day
Base QPS  =  ──────────────────────────────────────
                         86,400

Peak QPS  =  Base QPS × peak_multiplier (2x to 5x)
```

**Step-by-step:**

```python
# QPS Calculation Template

DAU = 250_000_000          # 250M daily active users
actions_per_day = 10       # average actions per user per day

# Total daily actions
total_daily = DAU * actions_per_day
# = 2,500,000,000 (2.5 billion)

# Seconds in a day
seconds_per_day = 86_400   # ~100K (use 100K for quick math)

# Base QPS
qps = total_daily / seconds_per_day
# = 2.5B / 86,400 ≈ 29,000 QPS

# Peak QPS (2x for steady traffic, 3-5x for spiky)
peak_qps = qps * 2
# ≈ 58,000 QPS
```

**Quick estimation shortcut:**
```
DAU × actions / 100K = QPS (approximate)

Example: 250M × 10 / 100K = 25,000 QPS
(Close enough to the exact 29,000 for interview purposes)
```

---

### 2. The Storage Calculation Formula

```
                                               avg_size
Daily Storage = new_items_per_day × ──────────────────────
                                     (text + metadata + media)

Yearly Storage = Daily × 365
Total Storage = Yearly × retention_years × replication_factor
```

**Worked example — Twitter tweets:**

```python
# Storage Estimation: Twitter Tweets

# Text storage
tweets_per_day = 500_000_000          # 500M tweets/day
avg_tweet_size = 280 * 2              # 280 chars × 2 bytes (UTF-8 avg) = 560 bytes

text_per_day = tweets_per_day * avg_tweet_size
# = 500M × 560 = 280 GB/day

# Metadata per tweet (user_id, timestamp, tweet_id, indexes)
metadata_per_tweet = 200               # bytes
metadata_per_day = tweets_per_day * metadata_per_tweet
# = 500M × 200 = 100 GB/day

# Media (assume 10% of tweets have images, avg 200KB)
media_tweets = tweets_per_day * 0.10   # 50M
avg_image_size = 200_000               # 200 KB
media_per_day = media_tweets * avg_image_size
# = 50M × 200KB = 10 TB/day

# Total daily storage
total_daily = text_per_day + metadata_per_day + media_per_day
# ≈ 280 GB + 100 GB + 10 TB ≈ 10.4 TB/day

# 5-year storage (with 3x replication)
total_5yr = total_daily * 365 * 5 * 3
# ≈ 10.4 TB × 365 × 5 × 3 ≈ 57 PB

# Key insight: MEDIA DOMINATES STORAGE.
# Text is tiny compared to images/video.
```

---

### 3. The Bandwidth Calculation Formula

```
Bandwidth = QPS × average_response_size

Incoming (write): write_QPS × avg_request_size
Outgoing (read):  read_QPS × avg_response_size
```

**Worked example:**

```python
# Bandwidth Estimation: Twitter Read

read_qps = 29_000                     # from QPS calculation
avg_timeline_response = 5_000         # 5 KB (20 tweets with metadata)

read_bandwidth = read_qps * avg_timeline_response
# = 29,000 × 5 KB = 145 MB/sec = 1.16 Gbps

# With images in timeline:
avg_with_images = 50_000              # 50 KB (with thumbnails)
rich_bandwidth = read_qps * avg_with_images
# = 29,000 × 50 KB = 1.45 GB/sec = 11.6 Gbps
```

---

### 4. The Cache Memory Calculation

The **80/20 rule** (Pareto principle) is your friend: 20% of data generates 80% of traffic.

```
Cache Size = daily_read_data × 0.20

Or more precisely:
Cache Size = unique_daily_requests × avg_response_size × 0.20
```

**Worked example:**

```python
# Cache Estimation: Twitter Timeline Cache

# How many unique timelines are read per day?
unique_users_reading = 250_000_000     # 250M DAU
avg_timeline_cache_size = 5_000        # 5 KB (latest 20 tweets)

# If we cache every active user's timeline:
full_cache = unique_users_reading * avg_timeline_cache_size
# = 250M × 5 KB = 1.25 TB

# With 80/20 rule (cache 20% of hot data):
hot_cache = full_cache * 0.20
# = 250 GB

# This fits in ~4-5 Redis instances with 64 GB RAM each.
```

---

### 5. Common System Estimates (Cheat Sheet)

Use these as starting points, then adjust for the specific problem:

```
┌──────────────────────────────────────────────────────────────┐
│              COMMON SYSTEM ESTIMATES                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SOCIAL MEDIA (Twitter/Instagram):                           │
│  DAU: 250-500M    Read:Write = 100:1 to 1000:1              │
│  Text tweet: 560 bytes    Photo: 200 KB    Video: 5 MB      │
│                                                              │
│  MESSAGING (WhatsApp/Slack):                                 │
│  DAU: 500M-2B    Messages/day: 50-100B                      │
│  Avg message size: 100 bytes    Avg group: 10-50 members    │
│                                                              │
│  URL SHORTENER (TinyURL):                                    │
│  New URLs/day: 100M    Reads/day: 10B                       │
│  Read:Write = 100:1    URL entry: ~500 bytes                │
│                                                              │
│  VIDEO STREAMING (YouTube):                                  │
│  DAU: 1B    Videos uploaded/day: 720K hrs                    │
│  Video storage: 50 MB/min (compressed, multiple qualities)  │
│                                                              │
│  E-COMMERCE (Amazon):                                        │
│  Orders/day: 50M    Product catalog: 500M items             │
│  Peak QPS during sales: 10x normal                          │
│                                                              │
│  FILE STORAGE (Dropbox):                                     │
│  Users: 500M    Avg files/user: 200    Avg file: 1 MB      │
│  Daily uploads: 1B files    Storage: 500 PB total           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### 6. Quick Math Tricks for Interviews

These shortcuts save precious interview minutes:

```
SHORTCUT 1: Seconds in a day
  86,400 ≈ 100,000 ≈ 10^5
  (Round up to 100K for quick division)

SHORTCUT 2: Seconds in a year
  ~31.5M ≈ 30M ≈ 3 × 10^7

SHORTCUT 3: Million/Billion division
  1 billion / 100K = 10,000
  (1B daily actions → 10K QPS)

SHORTCUT 4: Storage unit conversion
  1 KB = 10^3 bytes
  1 MB = 10^6 bytes
  1 GB = 10^9 bytes
  1 TB = 10^12 bytes
  1 PB = 10^15 bytes

SHORTCUT 5: Data size estimates
  Character: 1-2 bytes
  Integer: 4-8 bytes
  UUID: 16 bytes (128 bits)
  Timestamp: 8 bytes (64-bit epoch)
  Short text (tweet): 500 bytes
  JSON API response: 1-10 KB
  Photo (compressed): 200 KB - 2 MB
  1 min HD video: 100 MB
  1 min 4K video: 350 MB

SHORTCUT 6: The 2-5 rule for peak
  Normal traffic → peak = 2-3x
  Spiky traffic (viral/events) → peak = 5-10x
```

---

### 7. Full Worked Example — Design a URL Shortener (Estimation)

This is the most common estimation exercise in interviews:

```python
# URL Shortener Estimation

# ─── ASSUMPTIONS ───
# 100M new short URLs created per day
# Read:Write ratio = 100:1 (redirects vs creations)
# Average URL length: 100 characters (original URL)
# Short URL: 7 characters
# Data retention: 10 years

# ─── QPS ───
write_qps = 100_000_000 / 86_400     # ≈ 1,160 writes/sec
read_qps = write_qps * 100            # ≈ 116,000 reads/sec
peak_read_qps = read_qps * 2          # ≈ 232,000 reads/sec

print(f"Write QPS: ~1,200/sec")
print(f"Read QPS: ~116K/sec")
print(f"Peak Read QPS: ~232K/sec")

# ─── STORAGE ───
# Each URL record: short_url(7B) + original_url(100B) + 
#                  created_at(8B) + user_id(8B) + 
#                  expiry(8B) + misc(69B) ≈ 200 bytes
records_per_day = 100_000_000
bytes_per_record = 200

daily_storage = records_per_day * bytes_per_record
# = 100M × 200B = 20 GB/day

yearly_storage = daily_storage * 365
# = 20 GB × 365 = 7.3 TB/year

total_10yr = yearly_storage * 10
# = 73 TB over 10 years

print(f"Daily storage: ~20 GB")
print(f"Yearly storage: ~7.3 TB")
print(f"10-year storage: ~73 TB")

# ─── BANDWIDTH ───
# Write: 1,200 req/sec × 200 bytes = 240 KB/sec (negligible)
# Read: 116K req/sec × 200 bytes = 23.2 MB/sec = ~186 Mbps
print(f"Read bandwidth: ~23 MB/sec")

# ─── CACHE ───
# Cache hot URLs (80/20 rule: 20% of URLs get 80% of traffic)
# Daily read requests: 116K QPS × 86,400 = ~10B reads/day
# Unique URLs accessed per day (estimate): ~200M
# Cache 20% of them: 40M × 200 bytes = 8 GB
print(f"Cache memory: ~8 GB (fits in a single Redis instance)")

# ─── SUMMARY ───
# ┌────────────────────────────────────────┐
# │ URL Shortener Estimation Summary       │
# ├────────────────────────────────────────┤
# │ Write QPS:       ~1,200/sec            │
# │ Read QPS:        ~116K/sec (peak 232K) │
# │ Daily storage:   ~20 GB               │
# │ 10-year storage: ~73 TB               │
# │ Read bandwidth:  ~23 MB/sec            │
# │ Cache needed:    ~8 GB                 │
# │                                        │
# │ Key insight: Read-heavy. Cache will    │
# │ handle most reads. Storage is modest.  │
# │ The challenge is read QPS, not storage.│
# └────────────────────────────────────────┘
```

---

## Comparison Table — How Estimation Shapes Design

| Metric | Small Scale | Large Scale | Design Implication |
|--------|-------------|-------------|-------------------|
| QPS < 1K | Single server | — | No load balancer needed |
| QPS 1K-10K | 2-5 servers | — | Simple load balancer |
| QPS 10K-100K | — | 10-50 servers | Need caching, CDN |
| QPS > 100K | — | 50+ servers | Need sharding, heavy caching |
| Storage < 1 TB | Single database | — | No sharding needed |
| Storage 1-10 TB | — | Large instance | Consider read replicas |
| Storage > 10 TB | — | Sharded DB | Must design sharding strategy |
| Bandwidth > 1 Gbps | — | — | Need CDN, edge caching |

---

## Real-World Example — YouTube Estimation

```
YouTube Daily Estimation:

Users:
  DAU: 1 billion
  Avg watch time: 40 minutes/day
  Uploads: 500 hours of video per minute

QPS:
  Video views: 5 billion/day
  View QPS = 5B / 86,400 ≈ 58,000/sec
  Search QPS ≈ 10,000/sec
  Upload QPS ≈ 500 hrs/min ÷ 60 ≈ 8 hrs/sec of new content

Storage (new uploads per day):
  500 hrs/min × 60 min/hr × 24 hrs = 720,000 hours/day
  Each hour at multiple qualities: ~5 GB
  720K hrs × 5 GB = 3.6 PB/day in new uploads

Bandwidth:
  1B users × 40 min × 5 Mbps (avg quality) / 8 bits
  = 1B × 40 × 60sec × 625KB/sec
  ≈ insanely high → that's why YouTube uses CDN edge
    servers in every major city

Key insight: YouTube's challenge isn't QPS — it's
BANDWIDTH and STORAGE. They store PETABYTES per day
and serve video at enormous bandwidth. The entire
design revolves around the CDN and video processing
pipeline.
```

---

## Common Interview Questions

### Q1: "Estimate the QPS and storage for a chat application like WhatsApp."

**Model answer:**
"WhatsApp has about 2 billion users with roughly 1 billion DAU. Each active user sends about 50 messages per day on average. That gives us 50 billion messages per day. QPS is 50B divided by roughly 100K seconds per day, which is about 580K messages per second. At peak, let's say 1.2 million per second. For storage, the average message is about 100 bytes of text. That's 50B × 100 bytes = 5 TB of text per day. With metadata, indexes, and media (10% of messages have images at 200KB each), daily storage is about 5 TB text + 1 PB media. The key insight is that media dominates storage but text dominates QPS."

### Q2: "How would you estimate if a single server can handle the load?"

**Model answer:**
"A single modern server can handle roughly 1K-10K QPS depending on the complexity of each request. A simple key-value lookup might handle 100K QPS on a single Redis instance, while a complex API call with database queries might handle only 1K QPS. I'd compare my estimated QPS to these benchmarks. If I need 5K QPS with simple operations, one server might work. If I need 50K QPS, I need at least 5-10 servers with a load balancer. I'd also check memory — if the working set fits in RAM (< 64-256 GB), caching will be effective."

### Q3: "Walk me through estimation for a system you haven't seen before."

**Model answer:**
"The process is always the same. First, estimate the number of users — DAU is the key metric. Second, estimate actions per user per day. Third, calculate QPS by dividing total daily actions by 86,400. Fourth, estimate the size of each piece of data and multiply by daily volume for storage. Fifth, calculate bandwidth as QPS times average response size. Sixth, estimate cache needs using the 80/20 rule — cache 20% of hot data. Finally, compare these numbers to single-server benchmarks to estimate the number of servers needed. The goal isn't exact numbers — it's the right order of magnitude."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| Spending 10 minutes on estimation | Spend 2-3 minutes max. Round aggressively. |
| Precise calculations (1,234,567 QPS) | Round to significant figures (1.2M QPS). Orders of magnitude matter, not decimals. |
| Forgetting peak vs average QPS | Always state both. Peak = average × 2-5x depending on traffic pattern. |
| Ignoring media when estimating storage | Media (images, video) usually dominates storage by 10-1000x over text. |
| Not stating assumptions | Always state assumptions explicitly: "Assuming 250M DAU and 10 tweets per user per day..." |
| Skipping estimation entirely | Even a quick "we're looking at roughly 50K QPS and 10 TB storage" gives important design context. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 4 CHEAT SHEET                         │
│                  Back-of-the-Envelope Estimation                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  QPS FORMULA:                                                    │
│  QPS = DAU × actions_per_day / 86,400                           │
│  Quick: DAU × actions / 100K                                    │
│  Peak QPS = QPS × 2 (or ×3-5 for spiky)                        │
│                                                                  │
│  STORAGE FORMULA:                                                │
│  Daily = new_items × avg_size                                   │
│  Yearly = Daily × 365                                           │
│  Total = Yearly × years × replication_factor                    │
│                                                                  │
│  BANDWIDTH: QPS × avg_response_size                              │
│  CACHE: daily_data × 0.20 (80/20 rule)                          │
│                                                                  │
│  QUICK SIZES:                                                    │
│  Tweet: 560B   Photo: 200KB   1min HD video: 100MB              │
│  JSON response: 1-10KB   UUID: 16B   Timestamp: 8B             │
│                                                                  │
│  QUICK CONVERSION:                                               │
│  86,400 sec/day ≈ 100K                                          │
│  31.5M sec/year ≈ 30M                                           │
│  1B / 100K = 10K QPS                                            │
│                                                                  │
│  SERVER CAPACITY:                                                │
│  Web server: 1K-10K QPS                                         │
│  Redis: 100K-200K ops/sec                                       │
│  MySQL: 1K-10K QPS                                              │
│  Kafka: 100K-200K msgs/sec                                      │
│                                                                  │
│  REMEMBER: State assumptions. Round aggressively.               │
│  The goal is ORDER OF MAGNITUDE, not precision.                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How many seconds in a day (approx)? | 86,400 ≈ 100,000 (10^5) |
| 2 | What's the formula for QPS? | QPS = DAU × actions_per_user_per_day / 86,400 |
| 3 | What's the 80/20 rule for caching? | 20% of data generates 80% of traffic. Cache 20% of hot data. |
| 4 | How much storage does a tweet take? | ~560 bytes (280 chars × 2 bytes UTF-8) + ~200 bytes metadata = ~760 bytes |
| 5 | What's the typical peak multiplier for QPS? | 2x for steady traffic, 3-5x for spiky traffic (events, viral content) |
| 6 | How many QPS can a single web server handle? | 1K-10K QPS depending on request complexity |
| 7 | When does media dominate storage over text? | Almost always. A single 200KB photo = 400 tweets of text. Always estimate media separately. |
| 8 | What's the quick formula for QPS? | DAU × actions_per_day / 100K (using 100K ≈ seconds in a day) |
| 9 | How much RAM does a Redis instance typically have? | 64-256 GB. A single instance can cache millions of keys. |
| 10 | How long should estimation take in an interview? | 2-3 minutes max. State assumptions, round aggressively, get the order of magnitude right. |

---

[← Previous: Scaling 101](./chapter-03-scaling-101.md) · [Next: The System Design Interview Framework →](./chapter-05-interview-framework.md)
