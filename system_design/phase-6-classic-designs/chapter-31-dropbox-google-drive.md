# Chapter 31: Design Dropbox / Google Drive

> **Phase 6 — Classic Design Problems** · Estimated Time: 4 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Uber / Ride Sharing](./chapter-30-uber-ride-sharing.md) · [Next: Notification System →](./chapter-32-notification-system.md)

---

## Why This Problem Matters

File storage introduces chunking, deduplication, sync conflicts, and efficient upload/download of large files. The key insight is splitting files into chunks for delta sync — only uploading changed parts of a file, not the entire file every time. This is fundamentally different from the "small data, high QPS" pattern of social media systems.

---

## Step 1: Requirements (5 min)

### Functional Requirements

| Feature | Description |
|---------|-------------|
| **Upload/download files** | Support files up to 10 GB |
| **Sync across devices** | Changes appear on all devices in near real-time |
| **File versioning** | View and restore previous versions |
| **Share files/folders** | Share with specific users or via link |
| **Offline support** | Edit offline, sync when reconnected |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Users** | 500M registered, 100M DAU |
| **Avg files/user** | 200 files |
| **Avg file size** | 1 MB |
| **Storage** | 500M × 200 × 1MB = 100 PB total |
| **Sync latency** | < 5 seconds after file change |
| **Durability** | 99.999999999% (11 nines, S3 level) |

---

## Step 2: Estimation (3 min)

```
TOTAL STORAGE: 100 PB (with dedup: ~50 PB)
UPLOADS/DAY: 100M DAU × 2 file changes = 200M uploads/day
UPLOAD QPS: 200M / 86,400 ≈ 2,300/sec
BANDWIDTH: 2,300/sec × 1 MB avg = 2.3 GB/sec upload

METADATA:
  500M users × 200 files × 500B metadata = 50 TB metadata
  Metadata fits in sharded MySQL

NOTIFICATIONS:
  100M DAU → need to notify ~50M devices of sync changes/day
```

---

## Step 3: High-Level Design (10 min)

### Architecture

```
┌─────────────┐     ┌─────────────┐     ┌───────────────────┐
│ Desktop      │     │ API Gateway  │     │ Metadata Service   │
│ Client       │────▶│              │────▶│ (file tree, perms, │
│ (sync agent) │     └─────────────┘     │  versions, sharing)│
└──────┬──────┘                          └────────┬──────────┘
       │                                          │
       │ chunk upload                      ┌──────▼──────┐
       │ (direct to S3                     │ Metadata DB  │
       │  via pre-signed URL)              │ (MySQL)      │
       │                                    └─────────────┘
       ▼
┌─────────────┐     ┌─────────────┐
│ S3 (chunks)  │     │ Notification │  Real-time sync
│ Block Server │     │ Service      │  (other devices get
└─────────────┘     │ (WebSocket)  │   notified of changes)
                     └─────────────┘
```

### API Design

```
POST /api/v1/files/upload-url
  Request:  { "file_name": "report.docx",
              "parent_folder_id": "folder_123",
              "chunk_hashes": ["abc...", "def...", "ghi..."],
              "chunk_sizes": [4194304, 4194304, 2097152],
              "total_size": 10485760 }
  Response: { "file_id": "file_456",
              "upload_urls": {
                "abc...": "https://s3.../presigned-1",  // only for NEW chunks
                "ghi...": "https://s3.../presigned-2"
              },
              "existing_chunks": ["def..."],  // already in S3 (dedup!)
              "version": 1 }

POST /api/v1/files/{file_id}/commit
  Request:  { "chunk_hashes": ["abc...", "def...", "ghi..."],
              "version": 1 }
  Response: { "file_id": "file_456", "version": 1, "status": "synced" }

GET /api/v1/files/{file_id}
  Response: { "file_id": "file_456",
              "file_name": "report.docx",
              "size": 10485760,
              "version": 3,
              "chunks": ["abc...", "def...", "ghi..."],
              "download_urls": { "abc...": "...", ... },
              "modified_at": "2024-01-15T10:30:00Z" }

GET /api/v1/files/{file_id}/versions
  Response: { "versions": [
    { "version": 3, "size": 10485760, "modified_at": "...", "modified_by": "user_1" },
    { "version": 2, "size": 10200000, "modified_at": "..." },
    { "version": 1, "size": 9800000, "modified_at": "..." }
  ] }

POST /api/v1/files/{file_id}/restore?version=2

POST /api/v1/shares
  Request:  { "file_id": "file_456", "user_ids": ["u_789"],
              "permission": "edit" }

GET /api/v1/sync/changes?since={timestamp}
  Response: { "changes": [
    { "file_id": "file_456", "action": "modified", "version": 3 },
    { "file_id": "file_789", "action": "created", "version": 1 },
    { "file_id": "file_111", "action": "deleted" }
  ], "timestamp": 1695600000 }
```

---

## Step 4: Detailed Design (20 min)

### Deep Dive 1: File Chunking & Delta Sync

```
PROBLEM: User edits a 10 MB file. Re-upload entire 10 MB?
SOLUTION: Split into 4 MB chunks. Only upload CHANGED chunks.

Original file (10 MB):
  [Chunk 1: 4MB] [Chunk 2: 4MB] [Chunk 3: 2MB]
       hash: abc       hash: def       hash: ghi

User edits middle of file (changes in Chunk 2 only):
  [Chunk 1: 4MB] [Chunk 2: 4MB] [Chunk 3: 2MB]
       hash: abc       hash: xyz       hash: ghi
                         ↑ CHANGED!

Delta sync: Only upload Chunk 2 (4 MB instead of 10 MB)
  → 60% bandwidth savings for this edit!

CHUNKING ALGORITHM:
  Fixed-size: split at every 4 MB boundary
    Simple but insertion at beginning shifts ALL chunks

  Content-defined (Rabin fingerprint):  ← Better
    Use rolling hash to find chunk boundaries based on content
    Inserting bytes at the beginning only affects the first chunk
    Subsequent chunks stay the same → even better delta sync

CHUNK SIZE TRADE-OFF:
  Smaller chunks (e.g., 1 MB):
    ✅ Finer-grained delta sync (upload less data on small edits)
    ❌ More metadata per file (more chunk hashes to store)
    ❌ More S3 operations
  
  Larger chunks (e.g., 16 MB):
    ✅ Less metadata overhead
    ❌ Coarser delta sync (larger uploads for small edits)
  
  4 MB: practical balance (what Dropbox uses)
```

### Deep Dive 2: Deduplication (Content-Addressable Storage)

```
DEDUPLICATION:
  If two users upload the same file, chunks have the same hash.
  Store only ONE copy of each unique chunk.

  User A: file → [chunk_abc, chunk_def, chunk_ghi]
  User B: same file → [chunk_abc, chunk_def, chunk_ghi]
  Storage: 3 chunks, not 6. (content-addressable storage)

HOW IT WORKS:
  1. Client computes SHA-256 hash of each chunk locally
  2. Sends chunk hashes to server: "Here are the hashes I have"
  3. Server checks: which hashes already exist in S3?
     → "abc and ghi already exist. Only upload def."
  4. Client uploads only NEW chunks
  5. Server stores mapping: file → [abc, def, ghi]

  File in DB references chunk hashes, not actual data.
  Same chunk hash → same S3 object. Zero duplication.

DEDUP RATIO:
  Enterprise: many users share same documents → 50-60% dedup
  Consumer: lower overlap → 20-30% dedup
  Either way: massive storage savings at 100 PB scale

REFERENCE COUNTING:
  Each chunk has a ref_count (how many file versions reference it)
  When ref_count drops to 0 → chunk can be garbage collected
  
  chunks table:
    chunk_hash | s3_key | size_bytes | ref_count
    abc...     | s3://.. | 4194304   | 3  (used by 3 file versions)

  GC job: DELETE FROM chunks WHERE ref_count = 0
          Then delete from S3
```

### Deep Dive 3: Sync Across Devices

```
USER EDITS FILE on Device A:

1. Sync client detects file change (via file system watcher)
   OS APIs: inotify (Linux), FSEvents (macOS), ReadDirectoryChangesW (Windows)
   
2. Split file into chunks, compute hashes
3. Compare chunk hashes with server-side metadata
4. Upload only changed chunks to S3 (via pre-signed URLs)
5. Commit: update metadata (new chunk list, new version)
6. Notification Service pushes change to Device B (via WebSocket)

Device B receives sync notification:
1. Fetch updated file metadata (new chunk list)
2. Compare with local chunks (which ones do I already have?)
3. Download only new/changed chunks from S3
4. Reassemble file locally

NOTIFICATION SERVICE:
  ┌──────────────────────────────────────────────┐
  │  Device A edits file                          │
  │       │                                       │
  │       ▼                                       │
  │  Upload chunks + commit metadata              │
  │       │                                       │
  │       ▼                                       │
  │  Notification Service                         │
  │       │                                       │
  │       ├──▶ WebSocket push to Device B (online)│
  │       │    "file_456 changed, version 3"      │
  │       │                                       │
  │       └──▶ Push notification to Device C       │
  │            (offline/mobile)                    │
  └──────────────────────────────────────────────┘

OFFLINE SUPPORT:
  Client maintains local changelog:
    [edit file_456 at T1, create file_789 at T2]
  When connection restored:
    Replay changelog against server
    Handle conflicts if same file changed on both sides
```

### Deep Dive 4: Conflict Resolution

```
CONFLICT SCENARIO:
  Device A edits file while Device B also edits the same file:
  
  Server version: V1
  Device A (offline): edits → local V2a
  Device B (offline): edits → local V2b
  Both come online and try to sync.

OPTION 1: Last-write-wins (Google Drive approach)
  Whichever device syncs last, their version wins.
  Simple but LOSES DATA (Device A's changes overwritten).

OPTION 2: Keep both versions (Dropbox approach) ✅
  Server stores BOTH as versions:
    V2a: "report.docx" (from Device A)
    V2b: "report (conflicted copy).docx" (from Device B)
  User manually resolves by choosing one or merging.
  
  More complex but NO DATA LOSS.

OPTION 3: Automatic merge (Google Docs approach)
  Use Operational Transformation to merge edits automatically.
  Only works for structured formats (text documents, spreadsheets).
  Doesn't work for binary files (images, PDFs, videos).

DETECTION:
  Each edit includes the base version it was derived from.
  Device A: "I edited V1 → V2a"
  Device B: "I edited V1 → V2b"
  Server sees: both based on V1 → CONFLICT!
  If Device B said "I edited V2a → V3" → no conflict (sequential).
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DROPBOX / GOOGLE DRIVE ARCHITECTURE              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────┐     ┌────────────┐     ┌──────────────────┐        │
│  │ Desktop     │────▶│ API Gateway │────▶│ Metadata Service  │        │
│  │ Sync Client │     │ (auth,      │     │ (file tree, perms │        │
│  │ (file       │     │  rate limit)│     │  versions, shares)│        │
│  │  watcher)   │     └────────────┘     └────────┬─────────┘        │
│  └──────┬─────┘                                  │                   │
│         │ chunk upload                    ┌───────▼────────┐         │
│         │ (pre-signed                     │ Metadata DB     │         │
│         │  URL direct                     │ (MySQL sharded) │         │
│         │  to S3)                         └────────────────┘         │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────┐    ┌──────────────────┐                           │
│  │ S3 (chunks)   │    │ Notification Svc  │                           │
│  │ Content-      │    │ (WebSocket +      │                           │
│  │ addressable   │    │  push notifs)     │                           │
│  │ by hash       │    └──────────────────┘                           │
│  └──────────────┘                                                    │
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           │
│  │ Sharing Svc   │    │ GC Worker     │    │ Search Svc   │           │
│  │ (permissions, │    │ (clean unused │    │ (full-text   │           │
│  │  link sharing)│    │  chunks)      │    │  file search)│           │
│  └──────────────┘    └──────────────┘    └──────────────┘           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Data Model

```sql
CREATE TABLE files (
    file_id     BIGINT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    file_name   VARCHAR(255),
    parent_folder_id BIGINT,
    file_size   BIGINT,
    is_folder   BOOLEAN DEFAULT FALSE,
    latest_version INT DEFAULT 1,
    created_at  TIMESTAMP,
    updated_at  TIMESTAMP,
    
    INDEX idx_user_parent (user_id, parent_folder_id)
);

CREATE TABLE file_versions (
    file_id     BIGINT,
    version     INT,
    chunk_hashes JSON,    -- ["abc...", "def...", "ghi..."]
    file_size   BIGINT,
    modified_by BIGINT,   -- user who made this version
    created_at  TIMESTAMP,
    PRIMARY KEY (file_id, version)
);

CREATE TABLE chunks (
    chunk_hash  CHAR(64) PRIMARY KEY,  -- SHA-256
    s3_key      VARCHAR(255),
    size_bytes  INT,
    ref_count   INT DEFAULT 1  -- dedup: how many file versions reference this
);

CREATE TABLE shares (
    share_id    BIGINT PRIMARY KEY,
    file_id     BIGINT NOT NULL,
    shared_with_user_id BIGINT,        -- NULL if link-based share
    share_link  VARCHAR(255) UNIQUE,   -- NULL if user-based share
    permission  ENUM('view', 'edit'),
    created_at  TIMESTAMP,
    
    INDEX idx_file (file_id),
    INDEX idx_user (shared_with_user_id)
);
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Chunk size | 4 MB | 1 MB, 16 MB | Balance between dedup efficiency and metadata overhead. 4 MB is industry standard (Dropbox). |
| Chunk storage | S3 (content-addressable by hash) | Self-hosted storage | Unlimited, 11 nines durability, enables dedup via hash-based addressing. |
| Chunking algorithm | Content-defined (rolling hash) | Fixed-size | Content-defined is resilient to insertions at file start — only first chunk changes. |
| Real-time sync | WebSocket notifications | Polling | Push-based for instant sync across devices. Polling would add 5-30 second latency. |
| Conflict resolution | Keep both versions (Dropbox) | Last-write-wins (Drive) | No data loss. More complex but user can always recover both edits. |
| Deduplication | Content-addressable (hash-based) | No dedup | Same chunk hash → same data → store once. 20-60% storage savings at petabyte scale. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is file chunking? | Splitting files into fixed-size chunks (e.g., 4 MB). Only changed chunks are uploaded on edits, saving bandwidth (delta sync). |
| 2 | How does deduplication work? | Chunks are stored by content hash (SHA-256). If two files have identical chunks, only one copy is stored. Reference count tracks usage. |
| 3 | How does real-time sync work across devices? | When Device A uploads changes, Notification Service pushes the change via WebSocket to Device B. Device B downloads only changed chunks. |
| 4 | How are sync conflicts handled? | Dropbox: keep both versions, user resolves. Google Drive: last-write-wins. Conflicts detected when two edits are based on the same base version. |
| 5 | Why 4 MB chunk size? | Trade-off: smaller = better dedup, smaller uploads, but more metadata. Larger = less metadata but larger uploads. 4 MB is the practical balance. |
| 6 | What is content-defined chunking? | Using a rolling hash (Rabin fingerprint) to find chunk boundaries based on content, not fixed offsets. Resilient to insertions at file start. |
| 7 | How is the upload optimized? | Pre-signed URLs (direct client → S3). Client sends chunk hashes first; server says which already exist (dedup). Only new chunks uploaded. |
| 8 | How does offline sync work? | Client maintains local changelog. When reconnected, replays changes against server. If conflicts exist, keeps both versions for user resolution. |

---

[← Previous: Uber / Ride Sharing](./chapter-30-uber-ride-sharing.md) · [Next: Notification System →](./chapter-32-notification-system.md)
