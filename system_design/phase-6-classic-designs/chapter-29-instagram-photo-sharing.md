# Chapter 29: Design Instagram / Photo Sharing

> **Phase 6 — Classic Design Problems** · Estimated Time: 3 hours · Difficulty: 🟡 Medium · Type: 🏗️ DESIGN

[← Previous: YouTube / Video Streaming](./chapter-28-youtube-video-streaming.md) · [Next: Uber / Ride Sharing →](./chapter-30-uber-ride-sharing.md)

---

## Why This Problem Matters

Instagram combines photo storage, news feed generation, and social graph — reinforcing Twitter design patterns while adding a media-heavy dimension. It's a great "second design" to practice after Twitter because the feed mechanics are similar but the upload pipeline and CDN requirements differ.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Upload photos** | Upload with filters, captions, and tags |
| **View photo feed** | Feed from followed users, reverse-chronological |
| **Follow/unfollow** | Social graph management |
| **Like & comment** | Engagement features |
| **Explore/discover** | Discover popular content |
| **Stories** | Ephemeral 24-hour content (mention, don't design) |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **DAU** | 500M |
| **Photos uploaded/day** | 100M |
| **Feed reads/day** | 5B |
| **Average photo size** | 200KB (after compression) |
| **Availability** | 99.99% |
| **Feed latency** | < 200ms |

---

## Step 2: Estimation (3 min)

```
WRITE QPS: 100M photos/day ÷ 86,400 ≈ 1,160 uploads/sec
READ QPS:  5B feed reads ÷ 86,400 ≈ 58,000/sec

STORAGE:
  Photos: 100M × 200KB = 20 TB/day
  Multiple sizes (thumbnail, medium, full): 20 TB × 3 = 60 TB/day
  Per year: ~22 PB

BANDWIDTH:
  Feed: 58K req/sec × 10 photos × 200KB = 116 GB/sec
  → 90%+ served from CDN, not origin

CACHE:
  Feed metadata: 500M users × 5KB = 2.5 TB total
  Active 20%: ~500 GB → Redis Cluster
```

---

## Step 3: High-Level Design (10 min)

### Architecture

```
UPLOAD FLOW:
┌────────┐  1.metadata  ┌──────────┐  2.pre-signed URL  ┌──────┐
│ Client  │─────────────▶│ App Server│──────────────────▶│  S3   │
│         │              │          │                    │      │
│         │──3.upload────────────────────────────────────▶│      │
│         │   directly   │          │                    │      │
└────────┘  to S3        └──────┬──┘                    └──┬───┘
                                │ 4.save metadata           │
                         ┌──────▼──────┐             ┌─────▼─────┐
                         │  Photo DB    │             │ Image      │
                         │  (MySQL)     │             │ Processing │
                         └─────────────┘             │ (resize,   │
                                                      │  thumbnail)│
                                                      └───────────┘

FEED FLOW:
┌────────┐    ┌──────────┐    ┌──────────┐    ┌───────────┐
│ Client  │───▶│ Feed Svc  │───▶│ Redis     │───▶│ Photo DB   │
│         │◀──│ (hybrid   │◀──│ (timeline │    │ (on miss)  │
│         │   │  fan-out) │    │  cache)   │    └───────────┘
└────────┘    └──────────┘    └──────────┘
                                                 Images from CDN
                                                 (not from feed API)
```

### API Design

```
POST /api/v1/photos/upload-url
  Request:  { "filename": "sunset.jpg", "content_type": "image/jpeg" }
  Response: { "photo_id": "ph_abc123",
              "upload_url": "https://s3.../presigned?...",
              "expires_in": 3600 }

POST /api/v1/photos/{photo_id}/metadata
  Request:  { "caption": "Beautiful sunset! 🌅",
              "tags": ["sunset", "nature"],
              "filter": "clarendon",
              "location": { "lat": 37.77, "lng": -122.41 } }
  Response: { "photo_id": "ph_abc123", "status": "processing" }

GET /api/v1/feed?cursor={cursor}&limit=10
  Response: { "posts": [
    { "photo_id": "ph_abc123",
      "user": { "id": "u_789", "username": "traveler" },
      "caption": "Beautiful sunset!",
      "image_urls": {
        "thumbnail": "https://cdn.../ph_abc123/thumb.jpg",
        "medium": "https://cdn.../ph_abc123/medium.jpg",
        "full": "https://cdn.../ph_abc123/full.jpg"
      },
      "like_count": 1234,
      "comment_count": 56,
      "liked_by_me": true,
      "created_at": "2024-01-15T10:30:00Z" },
    ...
  ], "next_cursor": "..." }

POST /api/v1/photos/{photo_id}/like
DELETE /api/v1/photos/{photo_id}/like

POST /api/v1/photos/{photo_id}/comments
  Request:  { "content": "Amazing shot! 📸" }

POST /api/v1/users/{user_id}/follow
DELETE /api/v1/users/{user_id}/follow
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: Photo Upload & Processing Pipeline

```
UPLOAD FLOW (Pre-signed URL pattern):

1. Client → App Server: "I want to upload a 5MB JPEG"
2. App Server:
   a. Generate photo_id (Snowflake)
   b. Create S3 pre-signed URL for upload
   c. Insert photo record in DB (status: 'uploading')
   d. Return pre-signed URL to client
3. Client uploads directly to S3 (bypasses app server entirely)
4. S3 sends event (via S3 Event Notification → SQS)
5. Image Processing Worker:
   a. Download original from S3
   b. Validate (format, dimensions, file integrity)
   c. Apply filter (if selected)
   d. Generate 3 sizes:
      - Thumbnail: 150×150 px (~10KB)
      - Medium: 640×640 px (~50KB)
      - Full: 1080×1080 px (~200KB)
   e. Upload all sizes to S3
   f. Push to CDN (cache warming for creator's followers)
   g. Update DB: status → 'ready', add image URLs
6. Fan-out: push photo_id to followers' timeline caches

WHY PRE-SIGNED URLs?
  Without: 100M photos × 200KB = 20 TB/day through app servers
  With: 0 bytes through app servers (direct client → S3)
  → App servers handle only metadata (< 1KB per request)
  → 1000× less bandwidth on app servers
```

### Deep Dive 2: Feed Generation (Hybrid Fan-out)

```
SAME PATTERN AS TWITTER — hybrid fan-out:

WHEN USER POSTS A PHOTO:
  1. Save photo to DB
  2. Check: is user a celebrity (> 10K followers)?
     → NO (normal user): Fan-out on write
       For each follower: ZADD timeline:{follower_id} {timestamp} {photo_id}
       Workers process this via Kafka
     → YES (celebrity): Skip fan-out. Photo only in Photo DB.

WHEN USER OPENS FEED:
  1. Read pre-computed timeline from Redis (normal user posts)
     ZREVRANGE timeline:{user_id} 0 9
  2. Fetch celebrity posts (from users with > 10K followers that I follow)
     For each celebrity: get latest 5 photos from Photo DB
  3. Merge both lists, sort by timestamp, return top 10
  4. Client fetches actual images from CDN using the URLs

  Total queries: 1 Redis read + ~3 celebrity queries = 4 queries
  (vs. 500 queries if pure fan-out-on-read with 500 followees)

FEED API RETURNS METADATA ONLY:
  The feed response contains photo URLs (CDN paths), captions, and counts.
  The CLIENT fetches images from CDN in parallel.
  This is crucial: feed API serves < 1KB per post,
  CDN serves the actual 50KB-200KB images.
```

### Deep Dive 3: Like Deduplication & Counting

```
LIKE SYSTEM:

Table: likes
  PRIMARY KEY (user_id, photo_id)  ← prevents double-likes at DB level

FLOW:
  User taps "like":
    1. INSERT INTO likes (user_id, photo_id) ON DUPLICATE KEY IGNORE
       → If already liked: silently ignored
       → If new: inserted
    2. UPDATE photos SET like_count = like_count + 1 WHERE photo_id = ?
       (only if step 1 succeeded)

PROBLEM: At 58K reads/sec, SELECT COUNT(*) FROM likes is too slow.

SOLUTION: Denormalized like_count on the photos table.
  Increment on like, decrement on unlike.
  Eventual consistency is fine (showing 1,234 vs 1,235 is acceptable).

"LIKED BY" Feature:
  "Liked by alice, bob, and 1,232 others"
  → Query: SELECT user_id FROM likes WHERE photo_id = ? LIMIT 3
  → Plus: photos.like_count for the total
  → Cache the first 3 likers in Redis for hot photos

UNLIKE FLOW:
  1. DELETE FROM likes WHERE user_id = ? AND photo_id = ?
  2. UPDATE photos SET like_count = like_count - 1 WHERE photo_id = ?
```

### Deep Dive 4: Explore/Discover Page

```
EXPLORE PAGE: Show popular/trending content personalized to user

APPROACH:
  1. TRENDING POOL:
     Background job computes trending photos every 10 minutes:
     → Photos with high like velocity (likes per minute)
     → Weighted by recency (newer photos ranked higher)
     → Store top 10,000 trending photos in Redis

  2. PERSONALIZATION:
     For each user, filter trending pool by:
     → Categories the user engages with (food, travel, fashion)
     → Users similar to this user (collaborative filtering)
     → Content from the user's region/language

  3. SERVING:
     Explore API → fetch personalized trending photos from cache
     → CDN serves the actual images

  This is a simplified view. Real Instagram uses a full ML 
  recommendation pipeline (similar to YouTube Ch. 28).
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     INSTAGRAM ARCHITECTURE                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐    ┌──────────┐    ┌──────────────────┐                 │
│  │ Mobile  │───▶│   CDN     │    │ API Gateway       │                 │
│  │ App     │    │ (images)  │    │ (auth, rate limit) │                 │
│  └────────┘    └──────────┘    └────────┬─────────┘                 │
│       │                                  │                            │
│       │ (images from CDN)               │ (metadata APIs)            │
│       │                                  │                            │
│       │              ┌───────────────────┼─────────────────┐         │
│       │              │                   │                 │         │
│       │        ┌─────▼─────┐      ┌─────▼──────┐   ┌─────▼─────┐  │
│       │        │ Upload Svc │      │ Feed Svc    │   │ User Svc  │  │
│       │        │ (pre-signed│      │ (hybrid     │   │ (follow,  │  │
│       │        │  URL, meta)│      │  fan-out)   │   │  profile) │  │
│       │        └─────┬─────┘      └──────┬─────┘   └───────────┘  │
│       │              │                    │                          │
│       │        ┌─────▼──────┐      ┌─────▼──────┐                  │
│       │        │ S3 (photos) │      │ Redis       │                  │
│       │        │ + Image     │      │ (timeline   │                  │
│       │        │ Processing  │      │  cache)     │                  │
│       │        │ Workers     │      └────────────┘                  │
│       │        └────────────┘                                       │
│       │                              ┌────────────┐                  │
│       ◀──────── CDN Edge Servers ───│ Photo DB   │                  │
│         (images served from          │ (MySQL     │                  │
│          nearest POP)                │  sharded)  │                  │
│                                      └────────────┘                  │
│                                                                      │
│  ┌───────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │ Kafka      │  │ Notification │  │ Explore /    │                 │
│  │ (fan-out   │  │ Service      │  │ Trending Svc │                 │
│  │  events)   │  │ (APNs/FCM)  │  │ (ML-based)   │                 │
│  └───────────┘  └──────────────┘  └──────────────┘                 │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Photo storage | S3 (3 sizes: thumb, medium, full) + CDN | Store in DB BLOB | Unlimited, cheap, CDN serves globally. Never store binary in a DB. |
| Upload | Pre-signed URL (direct to S3) | Upload through app server | Don't route 200KB × 100M/day through app servers. Saves 20 TB/day bandwidth. |
| Feed | Hybrid fan-out (push + pull) | Pure push or pull | Push: fast reads but celebrity problem. Pull: slow reads. Hybrid gets best of both. |
| Like dedup | Composite PK (user_id, photo_id) | Application-level check | DB enforces uniqueness atomically. No race conditions. |
| Like counts | Denormalized on photos table | COUNT(*) on likes | COUNT at 58K QPS would kill the DB. Pre-computed counter is O(1). |
| Feed API | Returns metadata only (URLs, not images) | Inline images | Client fetches images from CDN in parallel. Feed API stays small (< 1KB/post). |
| Image sizes | 3 fixed sizes per photo | Dynamic resizing | Pre-computed is CDN-cacheable. Dynamic resizing = compute on every request. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you handle photo uploads at scale? | Pre-signed URLs: client uploads directly to S3, bypassing app servers. App server only generates the URL and stores metadata. Saves 20 TB/day bandwidth. |
| 2 | How many photo sizes does Instagram store? | At least 3: thumbnail (~150px, 10KB), medium (~640px, 50KB), full resolution (~1080px, 200KB). Each stored as a separate S3 object. |
| 3 | How is the Instagram feed generated? | Hybrid fan-out (same as Twitter): push for normal users (< 10K followers), pull for celebrities. Timeline cached in Redis Sorted Set. Merge at read time. |
| 4 | How do you prevent double-likes? | Composite primary key (user_id, photo_id) on the likes table. DB rejects duplicate inserts atomically. |
| 5 | Does the feed API return actual images? | No. It returns photo metadata (CDN URLs, captions, counts). Client fetches images from CDN separately in parallel. |
| 6 | How is like_count maintained? | Denormalized counter on the photos table. Incremented on like, decremented on unlike. Avoids expensive COUNT(*) queries. |
| 7 | How does the Explore page work? | Background job computes trending photos (high like velocity + recency). Personalized per user via collaborative filtering on engagement categories. |
| 8 | Why use 3 fixed image sizes instead of dynamic resizing? | Pre-computed sizes are CDN-cacheable. Dynamic resizing requires compute on every request and produces un-cacheable responses. |

---

[← Previous: YouTube / Video Streaming](./chapter-28-youtube-video-streaming.md) · [Next: Uber / Ride Sharing →](./chapter-30-uber-ride-sharing.md)
