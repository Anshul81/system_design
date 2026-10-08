# Chapter 33: Design a Web Crawler

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Notification System](../phase-6-classic-designs/chapter-32-notification-system.md) · [Next: Ticket Booking →](./chapter-34-ticket-booking.md)

---

## Why This Problem Matters

Web crawlers test distributed task scheduling, deduplication, politeness constraints, and handling billions of URLs. It's a common Google/Amazon question.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Pages to crawl** | 1B pages/month |
| **Politeness** | Max 1 request/sec per domain |
| **Deduplication** | Don't re-crawl unchanged pages |
| **Depth** | Crawl from seed URLs, following links |
| **Extensible** | Support different content types (HTML, PDF, images) |

**Estimation:**
```
1B pages/month ÷ 30 days ÷ 86,400 sec = ~390 pages/sec
Peak: ~800 pages/sec
Avg page: 500 KB → 390 × 500 KB = 195 MB/sec bandwidth
Storage: 1B × 500 KB = 500 TB/month
```

---

## Architecture

```
┌──────────┐
│ Seed URLs │
└─────┬────┘
      ▼
┌────────────────┐     ┌──────────────┐
│  URL Frontier    │────▶│ URL Dedup     │  (Bloom filter:
│  (priority queue │     │ Service       │   "Have we seen
│   + politeness   │     │               │    this URL?")
│   scheduler)     │     └──────────────┘
└────────┬───────┘
         │ dispatch URLs
         ▼
┌────────────────────────────────────────┐
│         Crawler Workers (100+)          │
│                                        │
│  1. Respect robots.txt                 │
│  2. Fetch page (HTTP GET)              │
│  3. Parse HTML → extract links         │
│  4. Content → storage pipeline         │
│  5. New links → URL Frontier           │
└────────┬──────────────────────┬────────┘
         │                      │
         ▼                      ▼
┌──────────────┐        ┌──────────────┐
│ Content Store │        │ URL Frontier  │
│ (S3 / HDFS)  │        │ (new URLs     │
│              │        │  to crawl)    │
└──────────────┘        └──────────────┘
```

### Key Components

**1. URL Frontier (Priority Queue + Politeness):**
```
Two concerns:
  Priority: Important pages (high PageRank, news sites) first
  Politeness: Max 1 request/sec per domain

Implementation:
  Per-domain queues with rate limiting:
    example.com  → [url1, url2, url3]  (1 req/sec)
    wikipedia.org → [url4, url5]       (1 req/sec)

  Priority selector picks from which domain queue to dequeue next.
  Uses separate threads per domain to enforce timing.
```

**2. URL Deduplication (Bloom Filter):**
```
Bloom Filter: Space-efficient probabilistic data structure
  "Is this URL in the set?"
  → "Definitely not" or "Probably yes"

For 1B URLs:
  Bloom filter with 1% false positive rate = ~1.2 GB
  vs. storing all URLs as strings = ~50 GB

False positive = we skip a URL we haven't crawled (acceptable)
False negative = impossible (we never re-crawl a seen URL)
```

**3. Content Dedup (Fingerprinting):**
```
Many pages have same content at different URLs:
  example.com/page and example.com/page?ref=twitter

Compute content fingerprint (SimHash or MinHash):
  If fingerprint matches existing page → skip (duplicate content)
  Saves storage and avoids indexing duplicates
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| URL dedup | Bloom filter | Space-efficient for billions of URLs. False positives acceptable. |
| Politeness | Per-domain rate limiter + robots.txt | Required by internet etiquette and to avoid getting blocked |
| Storage | S3/HDFS | Petabyte-scale, cheap, batch-processing friendly |
| Link extraction | BFS with priority | BFS ensures breadth of coverage. Priority ensures important pages first. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the URL frontier? | A priority queue with politeness constraints. Schedules which URLs to crawl next while respecting per-domain rate limits. |
| 2 | Why use a Bloom filter for URL dedup? | Space-efficient (1.2 GB for 1B URLs vs 50 GB for full strings). False positives are acceptable (skip a URL = OK). |
| 3 | What is robots.txt? | A file on each website that specifies which paths crawlers are allowed/disallowed from accessing. Politeness requires respecting it. |
| 4 | How do you handle duplicate content at different URLs? | Content fingerprinting (SimHash). Compare fingerprint with existing pages. If match → skip as duplicate. |
| 5 | Why BFS over DFS for web crawling? | BFS ensures broad coverage (don't go infinitely deep into one site). With priority, important pages are crawled first. |

---

[← Previous: Notification System](../phase-6-classic-designs/chapter-32-notification-system.md) · [Next: Ticket Booking →](./chapter-34-ticket-booking.md)
