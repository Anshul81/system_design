# Chapter 26: Design Twitter / News Feed

> **Phase 6 — Classic Design Problems** · Estimated Time: 4 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: URL Shortener](./chapter-25-url-shortener.md) · [Next: Chat System →](./chapter-27-chat-system.md)

---

## Why This Problem Matters

This is THE most frequently asked system design question. It tests everything: database design, caching, fan-out strategies, real-time delivery, and scaling. The fan-out-on-write vs fan-out-on-read trade-off is the centerpiece discussion that separates strong candidates from average ones.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Post tweet** | Users can post text tweets (280 chars) with optional media |
| **Home timeline** | View a feed of tweets from people you follow, reverse-chronological |
| **Follow/Unfollow** | Follow other users to see their tweets |
| **Like & Retweet** | Engage with tweets |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **DAU** | 250M |
| **Tweets/day** | 500M (avg 2 tweets per active user) |
| **Timeline reads/day** | 25B (avg 100 reads per active user) |
| **Read:Write** | 50:1 |
| **Timeline latency** | < 200ms |
| **Availability** | 99.99% |

---

## Step 2: Estimation (3 min)

```
WRITE QPS:
  500M tweets/day ÷ 86,400 ≈ 5,800 tweets/sec
  Peak: ~12,000/sec

READ QPS:
  25B timeline reads/day ÷ 86,400 ≈ 290,000/sec
  Peak: ~580,000/sec

STORAGE:
  Text: 500M tweets × 560 bytes = 280 GB/day
  Media (10% with images): 50M × 200KB = 10 TB/day
  5 years: ~18 PB (media dominates)

CACHE:
  Timeline cache: 250M users × 5KB (top 20 tweets) = 1.25 TB
  Hot 20%: ~250 GB → Redis cluster
```

---

## Step 3: High-Level Design (10 min)

### Architecture Diagram

```
┌────────────┐     ┌─────────────┐
│   Client    │────▶│ API Gateway  │
└────────────┘     └──────┬──────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │ Tweet       │ │ Timeline   │ │ User       │
    │ Service     │ │ Service    │ │ Service    │
    └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
           │              │              │
    ┌──────▼─────┐ ┌──────▼─────┐ ┌──────▼─────┐
    │ Tweet DB    │ │ Timeline   │ │ User DB     │
    │ (MySQL)     │ │ Cache      │ │ (MySQL)     │
    └────────────┘ │ (Redis)    │ └────────────┘
                   └────────────┘
           │
    ┌──────▼──────────┐     ┌──────────────┐
    │ Fan-out Service  │────▶│ Message Queue │
    │ (async workers)  │     │ (Kafka)       │
    └─────────────────┘     └──────────────┘
```

### API Design

```
POST /api/v1/tweets
  Body: { "content": "Hello world!", "media_ids": ["img_123"] }
  Response: { "tweet_id": "tw_456", "created_at": "..." }

GET /api/v1/timeline?cursor={cursor}&limit=20
  Response: { "tweets": [...], "next_cursor": "tw_440" }

POST /api/v1/follow
  Body: { "followee_id": "user_789" }

GET /api/v1/users/{id}/tweets?cursor={cursor}&limit=20
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: Fan-out Strategy (THE Key Decision)

```
OPTION A: FAN-OUT ON WRITE (Push Model)
──────────────────────────────────────
When User A posts a tweet:
1. Write tweet to Tweet DB
2. Look up all of User A's followers
3. For EACH follower, insert tweet_id into their timeline cache

User A posts tweet → Fan-out Service:
  ├── Push to Follower 1's timeline cache
  ├── Push to Follower 2's timeline cache
  ├── Push to Follower 3's timeline cache
  └── Push to Follower N's timeline cache

Timeline read: Just read from cache → O(1) read, super fast!

PRO: Reads are very fast (pre-computed timeline)
CON: Celebrity problem — user with 50M followers → 50M cache writes!
     Write amplification is massive for popular users.
     Wasted writes for inactive followers who never check timeline.


OPTION B: FAN-OUT ON READ (Pull Model)
──────────────────────────────────────
When User B opens timeline:
1. Look up all users B follows
2. Fetch recent tweets from each followed user
3. Merge and sort by time
4. Return top N tweets

User B opens timeline:
  ├── Fetch User X's recent tweets
  ├── Fetch User Y's recent tweets
  ├── Fetch User Z's recent tweets
  └── Merge + sort + return top 20

PRO: No write amplification. Simple writes.
CON: Reads are SLOW (must query many users and merge).
     If B follows 500 users → 500 queries per timeline load.


OPTION C: HYBRID (What Twitter Actually Does) ✅
──────────────────────────────────────────────
• Fan-out on WRITE for normal users (< 10K followers)
  → Pre-compute their followers' timelines

• Fan-out on READ for celebrities (> 10K followers)
  → Fetch celebrity tweets at read time and merge with cached timeline

User A (500 followers) posts:
  → Push tweet to all 500 followers' timeline caches ✅

Celebrity (50M followers) posts:
  → Just write to Tweet DB. Don't fan out. ✅

When User B opens timeline:
  → Read pre-computed timeline from cache
  → Fetch recent tweets from celebrities B follows
  → Merge and return
```

```
HYBRID FLOW:

┌──────────────────────────────────────────────────────────┐
│ User B opens timeline:                                    │
│                                                          │
│ 1. Read pre-computed timeline from Redis                 │
│    (contains tweets from normal users B follows)         │
│    → [tweet_1, tweet_3, tweet_5, tweet_7, ...]          │
│                                                          │
│ 2. Fetch celebrity tweets (B follows 3 celebrities)      │
│    Celebrity A → latest 5 tweets                         │
│    Celebrity B → latest 5 tweets                         │
│    Celebrity C → latest 5 tweets                         │
│                                                          │
│ 3. Merge all tweets, sort by time, return top 20         │
│    → [tweet_1, celeb_tweet, tweet_3, celeb_tweet, ...]  │
│                                                          │
│ Total queries: 1 (cache) + 3 (celebrities) = 4 queries  │
│ vs. fan-out-on-read: 500 queries (one per followed user) │
└──────────────────────────────────────────────────────────┘
```

### Deep Dive 2: Data Model

```sql
-- Tweet Storage (MySQL, sharded by user_id)
CREATE TABLE tweets (
    tweet_id    BIGINT PRIMARY KEY,   -- Snowflake ID
    user_id     BIGINT NOT NULL,
    content     VARCHAR(560) NOT NULL, -- 280 chars × 2 bytes
    media_url   TEXT,
    created_at  TIMESTAMP NOT NULL,
    like_count  INT DEFAULT 0,
    retweet_count INT DEFAULT 0,
    
    INDEX idx_user_time (user_id, created_at DESC)
);

-- Follow Graph (MySQL, sharded by follower_id)
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    created_at  TIMESTAMP NOT NULL,
    
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)  -- "Who follows me?"
);

-- Timeline Cache (Redis)
-- Key: timeline:{user_id}
-- Value: Sorted Set of tweet_ids (scored by timestamp)
-- ZADD timeline:user_123 1634567890 tweet_456
-- ZREVRANGE timeline:user_123 0 19 → latest 20 tweet IDs
```

### Deep Dive 3: Fan-out Service Pipeline

```
┌──────────┐   tweet    ┌──────────┐   fan-out   ┌──────────────┐
│ Tweet     │──created──▶│  Kafka    │───tasks────▶│ Fan-out       │
│ Service   │   event    │  Topic    │            │ Workers       │
└──────────┘            └──────────┘            └──────┬───────┘
                                                        │
                                                        │ For each
                                                        │ follower:
                                                        ▼
                                                 ┌──────────────┐
                                                 │ Timeline Cache│
                                                 │ (Redis)       │
                                                 │               │
                                                 │ ZADD          │
                                                 │ timeline:{uid}│
                                                 │ {timestamp}   │
                                                 │ {tweet_id}    │
                                                 └──────────────┘

Fan-out worker logic:
1. Receive tweet event from Kafka
2. Check: is author a celebrity (>10K followers)?
   → Yes: Skip fan-out. Tweet stored in DB only.
   → No: Continue.
3. Fetch author's follower list
4. For each follower:
   a. ZADD tweet to follower's timeline in Redis
   b. ZREMRANGEBYRANK to keep only latest 800 tweets
5. Acknowledge Kafka offset
```

### Deep Dive 4: Scaling

```
SCALING CHALLENGES AND SOLUTIONS:

Timeline Cache (Redis):
  250M users × 5KB = 1.25 TB total
  → Redis Cluster with ~20 nodes (64 GB each)
  → Consistent hashing for key distribution
  → Not all users cached — only recently active

Tweet DB:
  Write QPS: ~6K/sec → manageable for single primary
  But storage grows (10 TB/day with media) → shard by user_id
  Shard benefit: user's tweets on same shard → fast user profile

Fan-out Bottleneck:
  500M tweets/day × avg 200 followers = 100B cache writes/day
  Peak: ~2M cache writes/sec
  → Multiple fan-out worker pools, partitioned by follower_id range
  → Kafka partitions align with worker pools

Media:
  Images → S3 + CDN (CloudFront)
  Upload via pre-signed URLs
  Serve via CDN edge servers
```

---

## Complete Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     TWITTER ARCHITECTURE                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────┐    ┌──────────┐    ┌─────────────┐               │
│  │ Mobile  │───▶│   CDN     │    │ API Gateway  │               │
│  │ Web     │    │ (images)  │    │ (auth, rate  │               │
│  └────────┘    └──────────┘    │  limiting)   │               │
│                                 └──────┬──────┘               │
│                                        │                       │
│              ┌─────────────────────────┼──────────────┐       │
│              │                         │              │       │
│        ┌─────▼─────┐           ┌──────▼──────┐ ┌─────▼─────┐│
│        │ Tweet Svc  │           │Timeline Svc │ │ User Svc  ││
│        └─────┬─────┘           └──────┬──────┘ └─────┬─────┘│
│              │                        │              │       │
│        ┌─────▼─────┐           ┌──────▼──────┐ ┌─────▼─────┐│
│        │Tweet DB    │           │Timeline     │ │User DB    ││
│        │(MySQL      │           │Cache (Redis │ │(MySQL)    ││
│        │ sharded)   │           │ Cluster)    │ └───────────┘│
│        └─────┬─────┘           └─────────────┘              │
│              │                                                │
│        ┌─────▼──────┐    ┌────────────┐                      │
│        │  Kafka      │───▶│ Fan-out     │                      │
│        │  (events)   │    │ Workers     │──▶ Timeline Cache    │
│        └────────────┘    └────────────┘                      │
│                                                                │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                 │
│  │ S3 (media)│  │Search (ES)│  │Analytics  │                 │
│  └───────────┘  └───────────┘  │(Kafka→DW) │                 │
│                                 └───────────┘                 │
└────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs Discussed

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Fan-out strategy | Hybrid (push for normal, pull for celebrities) | Pure push or pull | Push is fast for reads but celebrity fan-out is prohibitive. Pull is slow for reads. Hybrid gets best of both. |
| Timeline storage | Redis Sorted Set | Cassandra | Redis gives sub-ms reads for timeline. Sorted sets naturally order by timestamp. |
| Tweet DB | MySQL sharded by user_id | Cassandra | User's tweets on one shard → fast profile. ACID for dedup. |
| Pagination | Cursor-based (tweet_id) | Offset-based | Feed changes constantly. Offset pagination gives duplicates. |
| Media | S3 + CDN | DB BLOB | S3 is cheap, unlimited, and CDN serves media globally with low latency. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is fan-out-on-write? | When a user posts, push the tweet into all followers' pre-computed timeline caches. Fast reads, expensive writes. |
| 2 | What is the celebrity problem? | A user with millions of followers causes millions of cache writes on every post. Solved by fan-out-on-read for celebrities only. |
| 3 | How does Twitter's hybrid fan-out work? | Push for normal users (< 10K followers), pull for celebrities (> 10K). Timeline = cached tweets + freshly-fetched celebrity tweets, merged. |
| 4 | Why cursor-based pagination for the timeline? | The feed constantly changes (new tweets). Offset-based would skip or duplicate tweets. Cursor (last seen tweet_id) gives a stable reading position. |
| 5 | What data structure stores the timeline in Redis? | Sorted Set. Score = timestamp, member = tweet_id. ZREVRANGE for latest N tweets. |
| 6 | Why shard the tweet DB by user_id? | All of a user's tweets on one shard → fast "view user profile" queries. Aligns with the most common access pattern. |

---

[← Previous: URL Shortener](./chapter-25-url-shortener.md) · [Next: Chat System →](./chapter-27-chat-system.md)
