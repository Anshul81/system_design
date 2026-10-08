# Chapter 25: Design a URL Shortener (TinyURL)

> **Phase 6 — Classic Design Problems** · Estimated Time: 3 hours · Difficulty: 🟡 Medium · Type: 🏗️ DESIGN

[← Previous: Availability Patterns](../phase-5-distributed-systems/chapter-24-availability-patterns.md) · [Next: Twitter / News Feed →](./chapter-26-twitter-news-feed.md)

---

## Why This Problem Matters

URL shortener is the #1 warmup system design question. It's simple enough to complete in 35 minutes, but deep enough to test estimation, database design, caching, and scaling. If you nail this, you build confidence for harder problems.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Shorten URL** | Given a long URL, generate a short unique URL |
| **Redirect** | Given a short URL, redirect to the original URL |
| **Custom aliases** | Optionally allow user-chosen short codes |
| **Expiration** | URLs can expire after a configurable time |
| **Analytics** | Track click counts per URL |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Scale** | 100M new URLs/day |
| **Read:Write ratio** | 100:1 (redirects >> creations) |
| **Redirect latency** | < 100ms |
| **Availability** | 99.99% |
| **URL length** | 7 characters (short code) |
| **Retention** | 10 years |

---

## Step 2: Estimation (3 min)

```
WRITES:
  100M new URLs/day
  Write QPS = 100M / 86,400 ≈ 1,200/sec

READS:
  Read:Write = 100:1
  Read QPS = 1,200 × 100 = 120,000/sec
  Peak Read QPS = 120K × 2 = 240,000/sec

STORAGE:
  Each URL record ≈ 200 bytes (short_code + original_url + metadata)
  Daily: 100M × 200B = 20 GB/day
  10 years: 20 GB × 365 × 10 = 73 TB

CACHE:
  80/20 rule: cache 20% of daily reads
  Daily unique URLs accessed ≈ 200M
  Cache: 200M × 0.2 × 200B = 8 GB → fits in 1 Redis instance

URL SPACE:
  7-character Base62: 62^7 = 3.5 trillion possible URLs
  10 years of URLs: 100M × 365 × 10 = 365B URLs
  3.5T >> 365B → plenty of space
```

---

## Step 3: High-Level Design (10 min)

### Architecture Diagram

```
┌────────────┐     ┌─────────────┐     ┌──────────────┐
│   Client    │────▶│ Load Balancer│────▶│  App Servers  │
└────────────┘     └─────────────┘     │  (Stateless)  │
                                        └───────┬──────┘
                                           │    │
                                    ┌──────▼─┐ ┌▼──────────┐
                                    │ Cache   │ │ Database   │
                                    │ (Redis) │ │ (MySQL)    │
                                    └────────┘ └────────────┘

WRITE PATH (Shorten):
Client → POST /api/v1/shorten {long_url}
  → App Server generates short_code
  → Store in DB: (short_code, long_url, created_at, expires_at)
  → Return: https://short.ly/{short_code}

READ PATH (Redirect):
Client → GET /{short_code}
  → App Server checks Redis cache
  → Cache HIT: return 301 redirect to original_url
  → Cache MISS: query DB → cache result → return 301 redirect
```

### API Design

```
POST /api/v1/shorten
  Request:  { "long_url": "https://example.com/very/long/path",
              "custom_alias": "mylink",    // optional
              "expiry_days": 365 }         // optional
  Response: { "short_url": "https://short.ly/abc1234",
              "expires_at": "2025-01-01T00:00:00Z" }

GET /{short_code}
  Response: HTTP 301 Redirect
            Location: https://example.com/very/long/path

GET /api/v1/stats/{short_code}
  Response: { "short_url": "https://short.ly/abc1234",
              "original_url": "https://example.com/...",
              "click_count": 12345,
              "created_at": "2024-01-01T..." }
```

### Database Schema

```sql
CREATE TABLE urls (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_code  CHAR(7) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id     BIGINT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at  TIMESTAMP,
    click_count BIGINT DEFAULT 0,

    INDEX idx_short_code (short_code),   -- Primary lookup
    INDEX idx_user_id (user_id)          -- User's URLs
);
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: URL Generation Strategy

**Option A: Hash + Truncate**
```
hash = MD5(long_url)               → 128-bit hash
take first 43 bits                 → truncate
base62_encode(43_bit_number)       → 7-character string

Problem: Hash collisions (two URLs get same short code)
Solution: Check DB for collision. If exists, append counter and re-hash.
```

**Option B: Counter-Based (Snowflake-like) ✅ Recommended**
```
1. Generate unique 64-bit ID (Snowflake or auto-increment)
2. Base62 encode the ID → short code

ID: 1234567890
Base62: "1LY7VK"
Pad to 7 chars: "01LY7VK"

Pro: No collisions (IDs are unique by definition)
Pro: Predictable, monotonically increasing
Con: Predictable (someone could guess next URL)
Fix: If unpredictability needed, use random ID or hash-based approach
```

**Option C: Pre-generated Key Service**
```
┌──────────────┐
│ Key Generation│  Pre-generates millions of unique 7-char codes
│ Service (KGS) │  Stores in a separate DB
└───────┬──────┘
        │
  When app needs a short code:
  1. Fetch next unused code from KGS
  2. Mark it as used
  3. Assign to the new URL

Pro: Simple, guaranteed unique, no collision checks
Con: KGS is a SPOF (need redundancy)
```

**Interview recommendation:** Counter-based (Option B) is simplest and scales. Mention hash-based as alternative and explain the collision trade-off.

### Deep Dive 2: Caching Strategy

```
Cache: Redis with cache-aside pattern

Write: Store in DB → Don't cache immediately (write-around)
Read:  Check Redis → Miss → Query DB → Cache result → Return

Cache key: short_code
Cache value: original_url
TTL: 24 hours (hot URLs stay cached)
Eviction: LRU

Why this works:
  • 120K reads/sec → most hit cache (90%+ hit rate)
  • DB handles only ~12K QPS (cache misses)
  • 8 GB cache fits in a single Redis instance
  • Popular URLs (viral links) served entirely from cache
```

### Deep Dive 3: Scaling

```
At 240K peak QPS:

App Servers:
  Each handles ~10K QPS → need ~24 app servers
  Stateless → behind a load balancer
  Auto-scaling: scale up/down with traffic

Database:
  Write QPS: 1,200/sec → single MySQL primary handles this
  73 TB over 10 years → need sharding eventually
  Shard key: hash(short_code) % N
  Start with single DB + read replicas, shard later

Cache:
  8 GB → single Redis instance
  At higher scale: Redis Cluster with consistent hashing

301 vs 302 Redirect:
  301 (Permanent): Browser caches redirect, fewer server hits
  302 (Temporary): Browser always comes to server (better for analytics)
  → Use 302 if you need click tracking, 301 for pure redirection
```

---

## Complete Architecture

```
┌────────────┐
│   Client    │
└──────┬─────┘
       │
┌──────▼──────────┐
│   DNS            │ short.ly → IP of load balancer
└──────┬──────────┘
       │
┌──────▼──────────┐
│  Load Balancer   │ L7, distributes across app servers
└──────┬──────────┘
       │
┌──────┼──────────────────────┐
│      │                      │
│ ┌────▼────┐  ┌────────┐  ┌─┴───────┐
│ │App Srv 1│  │App Srv 2│  │App Srv N│  Stateless, auto-scaling
│ └────┬────┘  └────┬────┘  └────┬────┘
│      │            │            │
└──────┼────────────┼────────────┘
       │            │
  ┌────▼────┐  ┌───▼────────┐
  │  Redis   │  │  MySQL      │
  │  Cache   │  │  Primary    │
  │  8 GB    │  │  + Replicas │
  └─────────┘  └────────────┘
```

---

## Key Trade-offs Discussed

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| ID generation | Counter + Base62 | Hash + truncate | No collisions, simpler |
| Database | MySQL | DynamoDB | ACID for dedup, familiar, handles write load |
| Cache strategy | Cache-aside | Write-through | Don't cache URLs that may never be read |
| Redirect code | 302 Temporary | 301 Permanent | Enables click tracking (analytics) |
| Short code length | 7 chars | 6 or 8 | 62^7 = 3.5T URLs — enough for 10 years |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How many URLs can a 7-char Base62 code represent? | 62^7 ≈ 3.5 trillion |
| 2 | What's the read:write ratio for a URL shortener? | ~100:1 (many more redirects than URL creations) |
| 3 | 301 vs 302 redirect for a URL shortener? | 301 (permanent): browser caches, fewer server hits. 302 (temporary): server sees every click (better for analytics). |
| 4 | How do you generate the short code? | Counter-based: generate unique 64-bit ID → Base62 encode → 7-char string. No collision possible. |
| 5 | Where do you store the mapping? | MySQL for persistence (short_code → original_url). Redis for caching hot URLs (~8 GB cache). |
| 6 | How do you handle collisions with hash-based generation? | Check DB for existing short_code. If collision, append counter and re-hash. Or use counter-based generation (no collisions). |

---

[← Previous: Availability Patterns](../phase-5-distributed-systems/chapter-24-availability-patterns.md) · [Next: Twitter / News Feed →](./chapter-26-twitter-news-feed.md)
