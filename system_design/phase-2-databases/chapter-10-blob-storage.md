# Chapter 10: Blob Storage & Data Lakes

> **Phase 2 — Databases & Storage** · Estimated Time: 1.5 hours · Difficulty: 🟢 Beginner

[← Previous: SQL vs NoSQL](./chapter-09-sql-vs-nosql.md) · [Next: Load Balancing →](../phase-3-load-balancing-caching/chapter-11-load-balancing.md)

---

## Learning Objectives

- ✅ Compare file storage, block storage, and object storage
- ✅ Know when and why to use object storage (e.g., AWS S3, GCP Cloud Storage) in system design
- ✅ Understand CDN integration with blob storage for serving media
- ✅ Describe data lake basics and when they're relevant

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapters 6-9 (Databases) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1 hr |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Interview relevance:** Every system that handles media (images, videos, files) needs blob storage. When designing Instagram, YouTube, Dropbox, or any system with user-uploaded content, you'll say "store the file in object storage (e.g., S3) and store the metadata (filename, size, owner) in the database." This separation is fundamental.

---

## Core Concepts

### 1. Storage Types Compared

```
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE TYPES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  BLOCK STORAGE (EBS, Azure Disk)                                │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐                                   │
│  │Blk1│ │Blk2│ │Blk3│ │Blk4│  Raw disk blocks                  │
│  └────┘ └────┘ └────┘ └────┘  Attached to ONE server            │
│  • Lowest latency, like a hard drive                             │
│  • Used for: OS disks, databases                                │
│                                                                  │
│  FILE STORAGE (EFS, NFS)                                        │
│  ├── /home/user/                                                │
│  │   ├── documents/                                             │
│  │   │   ├── report.pdf                                         │
│  │   │   └── notes.txt                                          │
│  │   └── photos/                                                │
│  • Hierarchical directory structure                              │
│  • Shared across multiple servers                                │
│  • Used for: shared file systems, NAS                           │
│                                                                  │
│  OBJECT STORAGE (S3, GCS, Azure Blob)                           │
│  ┌──────────────────────────────────────┐                       │
│  │  Bucket: "my-app-images"             │                       │
│  │  ┌───────────────────────────────┐   │                       │
│  │  │ Key: "users/123/avatar.jpg"   │   │                       │
│  │  │ Value: [binary image data]    │   │                       │
│  │  │ Metadata: {size, type, date}  │   │                       │
│  │  └───────────────────────────────┘   │                       │
│  │  ┌───────────────────────────────┐   │                       │
│  │  │ Key: "posts/456/photo.png"    │   │                       │
│  │  │ Value: [binary image data]    │   │                       │
│  │  └───────────────────────────────┘   │                       │
│  └──────────────────────────────────────┘                       │
│  • Flat namespace (key → blob), no directories                  │
│  • HTTP API (PUT/GET/DELETE via REST)                            │
│  • Virtually unlimited storage                                   │
│  • Used for: images, videos, backups, static assets             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Feature | Block | File | Object |
|---------|-------|------|--------|
| **Access** | Low-level (blocks) | File system (paths) | HTTP API (keys) |
| **Scalability** | Single server | Moderate | Virtually unlimited |
| **Latency** | Lowest (~μs) | Low (~ms) | Higher (~10-100ms) |
| **Sharing** | Single server | Multiple servers | Global (via HTTP) |
| **Cost** | $$$ | $$ | $ |
| **Best for** | Databases, OS | Shared workloads | Media, backups, static |

### 2. Object Storage in System Design

The standard pattern for media-heavy systems:

```
Upload Flow:
┌────────┐  1. Upload    ┌──────────┐  2. Store    ┌──────────┐
│ Client  │────metadata──▶│ App       │────file────▶│ S3       │
│         │              │ Server    │             │ (Object  │
│         │  3. Return   │           │  Return URL │ Storage) │
│         │◀────URL──────│           │◀────────────│          │
└────────┘              └──────────┘             └──────────┘

Read Flow:
┌────────┐  1. Request   ┌──────────┐
│ Client  │────image────▶│ CDN Edge  │  (cache hit = fast!)
│         │◀────image────│ Server    │
└────────┘              └─────┬────┘
                              │ cache miss
                              ▼
                        ┌──────────┐
                        │ S3       │
                        │ (Origin) │
                        └──────────┘
```

**Key properties of object storage (S3):**
- **Durability:** 99.999999999% (11 nines) — data virtually never lost
- **Availability:** 99.99%
- **Capacity:** Unlimited — pay for what you use
- **Access:** HTTP REST API, not file system
- **Cost:** ~$0.023/GB/month (S3 Standard) — extremely cheap at scale

**Interview pattern:** "Store the file in S3, store the metadata (URL, size, owner, timestamps) in the relational database, and serve through a CDN."

### 3. Pre-signed URLs

For secure direct uploads without routing through your server:

```
Secure Upload with Pre-signed URL:

┌────────┐  1. Request      ┌──────────┐  2. Generate    ┌──────────┐
│ Client  │────upload URL──▶│ App       │────pre-signed──▶│ S3       │
│         │                 │ Server    │    URL           │          │
│         │◀──pre-signed────│           │                 │          │
│         │   URL            └──────────┘                 │          │
│         │                                                │          │
│         │  3. Upload file DIRECTLY to S3                 │          │
│         │──────────────────────────────────────────────▶│          │
└────────┘                                                └──────────┘

Benefits:
  • No file data flows through your server (saves bandwidth)
  • URL expires after N minutes (secure)
  • Scales: S3 handles the upload, not your servers
```

### 4. Data Lakes (Brief)

A data lake stores raw, unprocessed data at massive scale for analytics:

```
Data Sources → Data Lake → Analytics/ML

┌───────────┐
│ App DB     │──┐
├───────────┤  │
│ Log files  │──┤     ┌──────────────┐     ┌──────────────┐
├───────────┤  ├────▶│  Data Lake     │────▶│  Analytics   │
│ API events │──┤     │  (S3 + format │     │  Spark, Presto│
├───────────┤  │     │   Parquet/ORC) │     │  Redshift     │
│ IoT data   │──┘     └──────────────┘     └──────────────┘
```

**When to mention in interviews:** Only when designing analytics platforms, metrics systems, or ML pipelines. Not for typical CRUD applications.

---

## Comparison Table

| Storage Type | Use in System Design | Example Service |
|-------------|---------------------|-----------------|
| **Object Storage** | Images, videos, file uploads, backups | AWS S3, GCS, Azure Blob |
| **Block Storage** | Database volumes, high-IOPS workloads | AWS EBS, Azure Disk |
| **File Storage** | Shared file systems across servers | AWS EFS, Azure Files |
| **CDN** | Serving static assets close to users | CloudFront, Cloudflare, Akamai |

---

## Common Interview Questions

### Q1: "Where would you store user-uploaded images?"

**Model answer:**
"In object storage like S3. I'd store the image binary in S3 with a key like `users/{user_id}/photos/{photo_id}.jpg`, and store the metadata (S3 URL, dimensions, upload time, user_id) in the relational database. For serving, I'd put a CDN like CloudFront in front of S3 so images are cached at edge locations worldwide, giving users low-latency access. For uploads, I'd use pre-signed URLs so the client uploads directly to S3, avoiding routing large files through my application servers."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 10 CHEAT SHEET                        │
│                  Blob Storage & Data Lakes                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  OBJECT STORAGE (S3):                                            │
│  • Store files, serve via CDN                                    │
│  • 11 nines durability, virtually unlimited                     │
│  • HTTP API (PUT/GET/DELETE)                                    │
│  • Use pre-signed URLs for direct client uploads                │
│                                                                  │
│  THE PATTERN:                                                    │
│  Binary data → S3                                                │
│  Metadata → PostgreSQL/MySQL                                     │
│  Serving → CDN in front of S3                                   │
│                                                                  │
│  BLOCK STORAGE: Database disks, high IOPS                       │
│  FILE STORAGE: Shared file systems                               │
│  DATA LAKE: Raw data for analytics (S3 + Parquet)               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 3 types of cloud storage? | Block (EBS), File (EFS), Object (S3) |
| 2 | What is the standard pattern for storing user uploads? | Binary data in S3, metadata in the relational DB, serve through CDN |
| 3 | What is a pre-signed URL? | A temporary, signed URL that allows a client to upload directly to S3 without going through your server |
| 4 | What is S3's durability? | 99.999999999% (11 nines) — data is virtually never lost |
| 5 | When would you use block storage vs object storage? | Block for databases (low latency, attached to one server). Object for files/media (HTTP access, unlimited scale, cheap). |
| 6 | What is a data lake? | A centralized repository of raw, unprocessed data stored at scale (typically in S3) for analytics and ML |

---

[← Previous: SQL vs NoSQL](./chapter-09-sql-vs-nosql.md) · [Next: Load Balancing →](../phase-3-load-balancing-caching/chapter-11-load-balancing.md)
