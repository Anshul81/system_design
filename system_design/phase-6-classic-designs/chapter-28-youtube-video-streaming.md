# Chapter 28: Design YouTube / Video Streaming

> **Phase 6 — Classic Design Problems** · Estimated Time: 3.5 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Chat System](./chapter-27-chat-system.md) · [Next: Instagram / Photo Sharing →](./chapter-29-instagram-photo-sharing.md)

---

## Why This Problem Matters

Video streaming tests CDN architecture, video processing pipelines, adaptive bitrate streaming, and storage at massive scale. The key insight is that YouTube's challenge is BANDWIDTH and STORAGE, not QPS. A single video can be gigabytes; multiply by billions of views and you have a fundamentally different problem from text-based systems like Twitter or URL shorteners.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Upload videos** | Users can upload videos of varying length and quality |
| **Stream/watch videos** | Adaptive quality streaming based on bandwidth |
| **Search videos** | Search by title, description, tags |
| **Like, comment, subscribe** | Social engagement features |
| **Video recommendations** | Personalized "Up Next" and home feed |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **DAU** | 1B |
| **Videos uploaded/min** | 500 hours of video |
| **Average watch time** | 40 min/user/day |
| **Availability** | 99.99% |
| **Global** | Low-latency streaming worldwide |
| **Upload latency** | Video playable within minutes of upload |

---

## Step 2: Estimation (3 min)

```
UPLOAD:
  500 hours/min = 30,000 hours/day = 720,000 hours/day
  Storage per hour (multiple qualities): ~5 GB
  Daily new storage: 720K hrs × 5 GB = 3.6 PB/day

STREAMING:
  1B users × 40 min/day × 5 Mbps avg quality ÷ 8 = 
  1B × 40 × 60s × 625KB/s = enormous bandwidth
  → 90%+ served from CDN edge, not origin

QPS:
  Video views: ~5B/day → ~58K view requests/sec
  Uploads: ~720K/day → ~8 uploads/sec (but each is heavy processing)
  Metadata reads (search, recommendations): ~100K/sec

STORAGE BREAKDOWN:
  Raw uploads: 3.6 PB/day
  After transcoding (5 qualities): ~18 PB/day total
  Per year: ~6.5 EB (exabytes!)
  → Need aggressive lifecycle policies: move old, unpopular videos 
    to cold storage (S3 Glacier)
```

---

## Step 3: High-Level Design (10 min)

### Architecture Diagram

```
UPLOAD FLOW:
┌────────┐  pre-signed  ┌──────────┐  raw video  ┌──────────────┐
│ Client  │────URL──────▶│ S3 (Raw)  │────────────▶│ Transcoding   │
│ uploads │              │           │             │ Pipeline      │
└────────┘              └──────────┘             │ (parallel)    │
                                                   │ 360p, 480p,  │
                                                   │ 720p, 1080p, │
                                                   │ 4K + HLS     │
                                                   └──────┬───────┘
                                                          │
                                                   ┌──────▼───────┐
                                                   │ S3 (Processed)│
                                                   │ + CDN Push    │
                                                   └──────────────┘

STREAMING FLOW:
┌────────┐    manifest     ┌──────────┐    video     ┌──────────┐
│ Client  │───(HLS/DASH)──▶│  CDN Edge │───segments──▶│ Client    │
│ (player)│◀──────────────│  Server   │◀────────────│ (renders) │
└────────┘                └──────────┘             └──────────┘
                              │ cache miss
                              ▼
                         ┌──────────┐
                         │ S3 Origin│
                         └──────────┘
```

### API Design

```
POST /api/v1/videos/upload-url
  Request:  { "filename": "my_video.mp4",
              "content_type": "video/mp4",
              "file_size_bytes": 524288000 }
  Response: { "video_id": "vid_abc123",
              "upload_url": "https://s3.../presigned?...",  // pre-signed URL
              "expires_in": 3600 }

POST /api/v1/videos/{video_id}/metadata
  Request:  { "title": "My Awesome Video",
              "description": "A great video about...",
              "tags": ["tutorial", "coding"],
              "category": "Education",
              "visibility": "public" }
  Response: { "video_id": "vid_abc123", "status": "processing" }

GET /api/v1/videos/{video_id}
  Response: { "video_id": "vid_abc123",
              "title": "My Awesome Video",
              "manifest_url": "https://cdn.../vid_abc123/master.m3u8",
              "thumbnail_url": "https://cdn.../vid_abc123/thumb.jpg",
              "duration_ms": 312000,
              "view_count": 1234567,
              "like_count": 45000,
              "channel": { "id": "ch_789", "name": "TechGuru" },
              "status": "ready" }

GET /api/v1/feed?cursor={cursor}&limit=20
  Response: { "videos": [...], "next_cursor": "..." }

GET /api/v1/search?q={query}&cursor={cursor}&limit=20
  Response: { "videos": [...], "next_cursor": "..." }

POST /api/v1/videos/{video_id}/like
POST /api/v1/videos/{video_id}/comments
  Request:  { "content": "Great video!" }

GET /api/v1/videos/{video_id}/comments?cursor={cursor}&limit=20
```

### Database Schema

```sql
-- Video metadata (MySQL, sharded by video_id)
CREATE TABLE videos (
    video_id    BIGINT PRIMARY KEY,    -- Snowflake ID
    user_id     BIGINT NOT NULL,
    title       VARCHAR(500),
    description TEXT,
    duration_ms INT,
    status      ENUM('uploading', 'processing', 'ready', 'failed'),
    view_count  BIGINT DEFAULT 0,
    like_count  INT DEFAULT 0,
    thumbnail_url TEXT,
    manifest_url TEXT,                 -- HLS manifest location
    category    VARCHAR(100),
    visibility  ENUM('public', 'unlisted', 'private') DEFAULT 'public',
    created_at  TIMESTAMP,
    
    INDEX idx_user (user_id, created_at DESC),
    INDEX idx_category (category, view_count DESC)
);

-- Channel / user info
CREATE TABLE channels (
    channel_id   BIGINT PRIMARY KEY,
    user_id      BIGINT UNIQUE NOT NULL,
    name         VARCHAR(255),
    subscriber_count BIGINT DEFAULT 0,
    created_at   TIMESTAMP
);

-- Subscriptions (follow relationship)
CREATE TABLE subscriptions (
    subscriber_id BIGINT NOT NULL,
    channel_id    BIGINT NOT NULL,
    created_at    TIMESTAMP,
    PRIMARY KEY (subscriber_id, channel_id),
    INDEX idx_channel (channel_id)
);

-- Video files stored in S3:
-- s3://videos/{video_id}/raw/original.mp4
-- s3://videos/{video_id}/360p/segment_001.ts
-- s3://videos/{video_id}/720p/segment_001.ts
-- s3://videos/{video_id}/1080p/segment_001.ts
-- s3://videos/{video_id}/master.m3u8
-- s3://videos/{video_id}/thumbnails/thumb_001.jpg
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: Video Processing Pipeline

```
RAW VIDEO → Processing Pipeline:

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Validate  │───▶│ Transcode │───▶│ Generate  │───▶│ Publish   │
│ (format,  │    │ (FFmpeg)  │    │ Thumbnails│    │ (update   │
│  size,    │    │ parallel: │    │ + Previews│    │  metadata │
│  content) │    │ 360p      │    │           │    │  DB, push │
│           │    │ 480p      │    │           │    │  to CDN)  │
│           │    │ 720p      │    │           │    │           │
│           │    │ 1080p     │    │           │    │           │
│           │    │ 4K        │    │           │    │           │
└──────────┘    └──────────┘    └──────────┘    └──────────┘

Each quality is a separate task → processed in parallel
Orchestrated by: message queue (Kafka/SQS)
Processing time: 10-60 min per video (async)
```

```
DETAILED PIPELINE FLOW:

1. UPLOAD COMPLETE → S3 sends event to SQS/Kafka
   
2. VALIDATION WORKER:
   → Check file format (mp4, mov, avi, etc.)
   → Check file size (reject > 128 GB)
   → Content moderation scan (ML model for NSFW/violence)
   → Extract metadata (codec, resolution, duration, framerate)
   → If invalid → mark video as "failed", notify user
   → If valid → produce transcoding tasks to Kafka

3. TRANSCODING WORKERS (Kafka consumers, GPU instances):
   → One task per (video_id, target_quality) pair
   → FFmpeg command: convert to H.264/H.265 + AAC audio
   → Split into 2-10 second HLS segments (.ts files)
   → Generate per-quality playlist (.m3u8)
   → Upload segments to S3 (processed bucket)
   
   Parallelism: 5 transcoding tasks run simultaneously
   Optimization: Start with lower resolutions first
   → 360p available in 2 min, 4K available in 30 min
   → User can start watching before all qualities are ready

4. THUMBNAIL WORKER:
   → Extract frames at key moments (scene changes)
   → Generate 3-5 thumbnail candidates
   → Run ML model to pick the most engaging thumbnail
   → Generate preview animation (6-second hover preview)

5. PUBLISH WORKER:
   → Generate master.m3u8 manifest linking all quality playlists
   → Update video status in MySQL: "processing" → "ready"
   → Push hot segments to CDN (proactive cache warming)
   → Send notification to subscribers (via Notification Service)

DAG Visualization:
  Upload Complete
       │
       ▼
    Validate ──failed──▶ Notify User (failed)
       │
       ▼
  ┌────┴─────────────────────┐
  │ Parallel Transcoding     │  Thumbnail
  │ 360p │ 480p │ 720p │ ... │  Generation
  └──────┴──────┴──────┴─────┘      │
       │                            │
       └───────────┬────────────────┘
                   ▼
               Publish
```

### Deep Dive 2: Adaptive Bitrate Streaming (ABR)

```
HLS (HTTP Live Streaming):

1. Video split into 2-10 second SEGMENTS
2. Each segment available in multiple QUALITIES
3. Client downloads a MANIFEST listing all segments and qualities
4. Client picks quality based on current bandwidth

Manifest (master.m3u8):
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
  360p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=854x480
  480p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
  720p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
  1080p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=14000000,RESOLUTION=3840x2160
  4k/playlist.m3u8

Per-quality playlist (720p/playlist.m3u8):
  #EXTM3U
  #EXT-X-TARGETDURATION:10
  #EXTINF:10.0,
  segment_001.ts
  #EXTINF:10.0,
  segment_002.ts
  #EXTINF:8.5,
  segment_003.ts

Client on fast WiFi: downloads 1080p segments
Client on slow 3G: switches down to 360p segments
Mid-stream bandwidth drop: player seamlessly switches quality

Each segment is a separate HTTP GET → perfect for CDN caching!

ABR Algorithm (Client-Side):
  1. Download segment N at current quality
  2. Measure download time vs segment duration
     → Downloaded 10s segment in 2s → bandwidth = 5× needed
     → Can upgrade quality
  3. If download took longer than segment duration → downgrade
  4. Use buffer level: if buffer > 30s, try higher quality
  5. Smooth transitions: don't switch quality on every segment
```

### Deep Dive 3: CDN Architecture

```
CDN ARCHITECTURE — The Key to YouTube's Scale

Why CDN matters:
  Without CDN: 1B users × 40 min/day → all traffic hits origin
  With CDN: 90-95% of traffic served from edge, near users

Multi-Tier CDN:

┌────────┐    ┌────────────┐    ┌────────────┐    ┌──────────┐
│ Client  │──▶│ CDN Edge    │──▶│ CDN Regional│──▶│ S3 Origin │
│         │   │ (city POP)  │   │ Shield      │   │          │
│         │   │ (100+ POPs) │   │ (10 regions)│   │          │
└────────┘   └────────────┘   └────────────┘   └──────────┘
              Cache hit: 85%    Cache hit: 10%    Origin: 5%

Edge POP (Point of Presence):
  → Located in every major city (100+ worldwide)
  → Caches the most popular video segments
  → Serves ~85% of all requests
  → Latency to user: 5-20 ms

Regional Shield:
  → Collapses duplicate requests to origin
  → If 100 edge POPs request the same segment → only 1 request to origin
  → Caches mid-tier content (popular but not viral)

Origin (S3):
  → Only handles ~5% of requests (long-tail content)
  → Store ALL video segments (every quality, every video)
  → Replicated across multiple regions for durability

CACHE WARMING:
  New viral video uploaded:
  1. Predict popularity (creator has 10M subscribers → will be popular)
  2. Proactively push segments to regional CDN nodes
  3. As views grow, segments propagate to edge nodes organically
  
  Old/unpopular video:
  → Not cached at edge, may not be cached at regional
  → First view → cache miss → fetch from S3 origin → cache at edge
  → Subsequent views from same region → cache hit

CDN KEY DESIGN:
  Cache key: {video_id}/{quality}/{segment_number}
  Example: vid_abc123/720p/segment_042.ts
  TTL: 7 days for popular content, 24 hours for long-tail
```

### Deep Dive 4: View Counting at Scale

```
CHALLENGE: 58K view events/sec → can't UPDATE a row in MySQL on every view

SOLUTION: Async aggregation with Redis + periodic flush

┌────────┐   view    ┌──────────┐  increment  ┌───────┐
│ Client  │──event──▶│ View Svc  │───────────▶│ Redis  │
└────────┘          └──────────┘            │ counter│
                                             └───┬───┘
                                                 │ flush every 30s
                                             ┌───▼───────┐
                                             │ MySQL DB   │
                                             │ (video row)│
                                             └───────────┘

Flow:
1. User watches video → client sends view event
   (with dedup: session_id + video_id, only count once per session)
2. View Service → INCR redis key "views:{video_id}" 
3. Background worker (cron, every 30s):
   → GETSET all dirty view counters → get current counts, reset to 0
   → Batch UPDATE videos SET view_count = view_count + {delta}
      WHERE video_id IN (...)
4. Display: show approximate count (exact doesn't matter for UX)

Why not Kafka for view counting?
  → Could also use Kafka → consumer aggregates → writes to DB
  → Both patterns work. Redis is simpler for pure counting.
  → Kafka better if you also need view events for analytics/ML pipeline

DEDUPLICATION:
  → Same user watching same video = 1 view (not counted on every buffer/refresh)
  → Key: user_id + video_id + time_bucket (1 view per 30-minute window)
  → Store in Redis SET: "viewed:{video_id}:{time_bucket}" → SADD user_id
  → Check before incrementing counter
```

### Deep Dive 5: Search & Recommendations

```
SEARCH (Elasticsearch):

Video metadata indexed in Elasticsearch:
  { video_id, title, description, tags, category,
    channel_name, view_count, upload_date }

Search query: "python tutorial for beginners"
  → ES full-text search on title + description + tags
  → Ranked by: relevance score × recency × view_count (popularity)
  → Return top 20 results with pagination (cursor-based)

Indexing pipeline:
  Video uploaded → metadata saved in MySQL
  → CDC (Change Data Capture) or Kafka event
  → Elasticsearch consumer indexes the document
  → Near real-time: searchable within seconds of upload

RECOMMENDATIONS (Separate ML Service):

Home feed algorithm (simplified):
  1. Candidate generation:
     → Videos from subscribed channels (not yet watched)
     → Videos similar to recently watched (collaborative filtering)
     → Trending videos in user's region
  2. Ranking:
     → ML model scores each candidate: P(click) × P(watch_time)
     → Features: user history, video metadata, channel stats, freshness
  3. Filtering:
     → Remove already watched, duplicates, blocked channels
  4. Return top N videos for the feed

"Up Next" sidebar:
  → Given current video, find related videos
  → Features: same channel, similar tags/category, co-watch patterns
  → "Users who watched X also watched Y"

Architecture:
  ┌────────┐   ┌──────────┐   ┌───────────────┐   ┌──────────┐
  │ Client  │──▶│ Feed API  │──▶│ Recommendation │──▶│ Video DB  │
  │         │   │           │   │ Service (ML)    │   │ (MySQL)   │
  │         │◀──│           │◀──│                 │   └──────────┘
  └────────┘   └──────────┘   └───────────────┘
                                        │
                                 ┌──────▼──────┐
                                 │ Feature Store│
                                 │ (Redis/      │
                                 │  BigTable)   │
                                 └─────────────┘

  Recommendation service is SEPARATE from video serving:
  → Different scaling needs (GPU-heavy for ML inference)
  → Different latency budget (can precompute feeds)
  → Different data (needs user behavior events, not video bytes)
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     YOUTUBE ARCHITECTURE                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐    ┌──────────┐    ┌──────────────────┐                 │
│  │ Mobile  │───▶│   CDN     │    │ API Gateway       │                 │
│  │ Web     │    │ (video    │    │ (auth, rate limit, │                 │
│  │ Smart TV│    │  segments)│    │  routing)          │                 │
│  └────────┘    └──────────┘    └────────┬─────────┘                 │
│       │                                  │                            │
│       │ (video bytes from CDN)          │ (metadata, APIs)           │
│       │                                  │                            │
│       │              ┌───────────────────┼────────────────┐          │
│       │              │                   │                │          │
│       │        ┌─────▼─────┐      ┌─────▼──────┐  ┌─────▼──────┐  │
│       │        │ Upload Svc │      │ Video Svc   │  │ Feed /     │  │
│       │        │ (pre-signed│      │ (metadata,  │  │ Search Svc │  │
│       │        │  URLs)     │      │  streaming) │  │            │  │
│       │        └─────┬─────┘      └──────┬─────┘  └──────┬─────┘  │
│       │              │                    │               │          │
│       │        ┌─────▼──────┐      ┌─────▼──────┐  ┌────▼──────┐  │
│       │        │ S3 (Raw)    │      │ Video DB   │  │ Elastic-  │  │
│       │        └─────┬──────┘      │ (MySQL     │  │ search    │  │
│       │              │             │  sharded)  │  └───────────┘  │
│       │        ┌─────▼──────┐      └────────────┘                  │
│       │        │ Transcoding │                                      │
│       │        │ Pipeline    │      ┌────────────┐                  │
│       │        │ (Kafka +    │      │ View Count │                  │
│       │        │  GPU workers│      │ Service    │                  │
│       │        │  + FFmpeg)  │      │ (Redis +   │                  │
│       │        └─────┬──────┘      │  batch DB) │                  │
│       │              │             └────────────┘                  │
│       │        ┌─────▼──────┐                                      │
│       │        │ S3 (Processed│     ┌─────────────┐                 │
│       │        │  + CDN push) │     │ Recom. Svc  │                 │
│       │        └─────────────┘     │ (ML models, │                 │
│       │                             │  feature    │                 │
│       ◀─────── CDN Edge Servers ───│  store)     │                 │
│         (video segments served      └─────────────┘                 │
│          from nearest POP)                                          │
│                                                                      │
│  ┌───────────┐  ┌───────────┐  ┌──────────┐  ┌─────────────┐      │
│  │ Kafka      │  │ Notif Svc │  │ Comments │  │ Analytics   │      │
│  │ (events)   │  │ (APNs/FCM)│  │ Service  │  │ (DW/Spark)  │      │
│  └───────────┘  └───────────┘  └──────────┘  └─────────────┘      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs Discussed

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Video storage | S3 + CDN | Self-hosted storage | Unlimited scale, CDN serves 90%+ traffic globally. S3 offers 11 nines durability. |
| Upload mechanism | Pre-signed URL (direct to S3) | Upload through app server | Don't route multi-GB files through app servers. Saves bandwidth and CPU. |
| Streaming protocol | HLS (HTTP Live Streaming) | DASH, RTMP | HTTP-based (CDN-friendly), adaptive bitrate, widest device support. DASH is similar but less iOS support. |
| Transcoding | Async pipeline (Kafka + GPU workers) | Synchronous processing | Video processing takes 10-60 min — must be async. DAG-based pipeline enables partial availability (watch 360p while 4K encodes). |
| View counts | Eventual consistency (Redis + batch flush) | Real-time DB updates | 58K views/sec would overwhelm DB with row-level locks. Redis INCR + batch flush every 30s is scalable. |
| Search | Elasticsearch | MySQL LIKE queries | Full-text search, relevance scoring, handles 100K+ QPS. MySQL LIKE doesn't scale for text search. |
| Recommendations | Separate ML service | Rule-based | Different scaling (GPU), different data needs, can iterate ML models independently of video serving. |
| Thumbnails | ML-selected from extracted frames | User-uploaded only | Auto-generated thumbnails improve CTR. ML picks the most engaging frame. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the main challenge of a video platform? | Bandwidth and storage, not QPS. Videos are large (GB each) and must be served globally with low latency via CDN. |
| 2 | What is adaptive bitrate streaming? | Video is encoded at multiple qualities. The player dynamically switches quality based on available bandwidth. Uses HLS or DASH protocol. |
| 3 | What is HLS? | HTTP Live Streaming. Video split into segments (2-10s), each available at multiple qualities. Manifest file lists all options. HTTP-based → CDN-friendly. |
| 4 | Why is video transcoding done asynchronously? | Transcoding to multiple qualities takes 10-60 minutes. Users can't wait. Jobs queued in Kafka, processed by GPU worker pools. Lower resolutions finish first for early availability. |
| 5 | How are view counts handled at scale? | Eventual consistency. INCR in Redis on each view event, batch flush to MySQL every 30s. Deduplication via (user_id, video_id, time_bucket) in Redis SET. |
| 6 | How do you upload large videos efficiently? | Pre-signed URLs: client uploads directly to S3, bypassing app servers. App server only generates the URL and stores metadata. |
| 7 | What is a CDN edge POP? | Point of Presence — a server in a city close to users. Caches popular video segments. ~85% of video traffic served from edge, 5-20ms latency to user. |
| 8 | How does YouTube's multi-tier CDN work? | Edge POPs (85% hits) → Regional Shield (collapses duplicate requests, 10% hits) → S3 Origin (5%). Only long-tail/unpopular videos hit origin. |
| 9 | Why separate the recommendation service? | Different scaling needs (GPU for ML), different data (user behavior, not video bytes), and independent iteration on ML models without affecting video serving. |
| 10 | How do you make a video watchable faster after upload? | Process lower resolutions first (360p ready in ~2 min, 4K in ~30 min). Users can start watching before all qualities are encoded. |

---

[← Previous: Chat System](./chapter-27-chat-system.md) · [Next: Instagram / Photo Sharing →](./chapter-29-instagram-photo-sharing.md)
