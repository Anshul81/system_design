# Chapter 13: CDN & Edge Computing

> **Phase 3 — Load Balancing, Caching & Proxies** · Estimated Time: 2 hours · Difficulty: 🟡 Medium

[← Previous: Caching](./chapter-12-caching.md) · [Next: Reverse Proxies & API Gateways →](./chapter-14-reverse-proxies-api-gateways.md)

---

## Learning Objectives

- ✅ Explain how CDNs work and why they reduce latency
- ✅ Compare push vs pull CDN strategies
- ✅ Design a CDN architecture for static and dynamic content
- ✅ Understand cache hierarchy and CDN invalidation
- ✅ Know when to use a CDN in system design

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 0 (DNS) | ✅ Yes |
| Chapter 12 (Caching) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** A CDN is like a franchise restaurant chain. Instead of one central kitchen serving the whole world, you open local branches (edge servers) in every city. The food (content) is the same, but customers get it from their nearest branch — faster, fresher, less travel.

**Interview relevance:** When designing any system with static assets (images, videos, CSS/JS), or when users are globally distributed, you MUST mention a CDN. The interviewer expects it. For video streaming (YouTube/Netflix) and photo sharing (Instagram), the CDN IS the design — it's where 90%+ of content is served from.

---

## Visual Diagram — CDN Architecture

```
WITHOUT CDN:
┌────────────┐                                    ┌──────────────┐
│ User in     │──── 150ms round trip ────────────▶│ Origin Server │
│ Tokyo       │◀── (content travels across ocean)──│ (US East)     │
└────────────┘                                    └──────────────┘

WITH CDN:
┌────────────┐         ┌──────────────┐          ┌──────────────┐
│ User in     │── 10ms─▶│ CDN Edge     │          │ Origin Server │
│ Tokyo       │◀─ 10ms──│ (Tokyo POP)  │          │ (US East)     │
└────────────┘         └──────┬───────┘          └──────┬───────┘
                              │                         │
                              │  Cache miss? Fetch      │
                              │  from origin ───────────┘
                              │  Then cache locally
                              │
Latency reduction: 150ms → 10ms (15x faster!)
```

```
CDN GLOBAL ARCHITECTURE:

                    ┌──────────────┐
                    │ Origin Server │  (your actual servers)
                    │ (US East)     │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼─────┐ ┌───▼─────┐
        │ CDN Edge   │ │ CDN Edge │ │ CDN Edge │
        │ Tokyo      │ │ London   │ │ São Paulo│
        │ (POP)      │ │ (POP)    │ │ (POP)    │
        └─────┬─────┘ └───┬─────┘ └───┬─────┘
              │            │            │
        ┌─────▼─────┐ ┌───▼─────┐ ┌───▼─────┐
        │ Users in   │ │ Users in │ │ Users in │
        │ Asia       │ │ Europe   │ │ S.America│
        └───────────┘ └─────────┘ └─────────┘

POP = Point of Presence (edge server location)
Major CDNs have 200-400+ POPs worldwide
```

---

## Core Concepts

### 1. Push vs Pull CDN

| Approach | How It Works | Best For |
|----------|-------------|----------|
| **Pull (lazy)** | CDN fetches from origin on first request. Cached for subsequent requests. | Dynamic content, long-tail content |
| **Push (eager)** | You proactively upload content to CDN edge servers. | Static assets you know will be popular |

```
PULL CDN:
1. User requests image.jpg from CDN
2. CDN edge doesn't have it (cache miss)
3. CDN fetches image.jpg from origin server
4. CDN caches it and serves to user
5. Next user gets it from CDN cache (cache hit!)

PUSH CDN:
1. You upload image.jpg to CDN
2. CDN distributes it to all edge servers
3. User requests image.jpg — served immediately from nearest edge
```

**Interview default:** Pull CDN for most cases. Push for known popular content (video files after transcoding, app bundles).

### 2. CDN Cache Headers

Your origin server tells the CDN how long to cache content using HTTP headers:

```
Cache-Control: public, max-age=31536000
  → CDN can cache this for 1 year (static asset with hash in filename)

Cache-Control: public, max-age=300
  → CDN caches for 5 minutes (semi-dynamic content)

Cache-Control: no-store
  → CDN must NOT cache this (personalized content, private data)

Cache-Control: public, max-age=3600, stale-while-revalidate=60
  → Cache for 1 hour. After expiry, serve stale for 60s while refreshing.
```

### 3. CDN Invalidation

When content changes, you need to clear the CDN cache:

| Strategy | How | Pros/Cons |
|----------|-----|-----------|
| **Cache busting** | Change filename: `style.v2.css` or `style.abc123.css` | Instant, reliable. Recommended. |
| **Purge API** | Call CDN API to delete specific URLs | Takes seconds to propagate globally |
| **Short TTL** | Set max-age=60 | Content refreshes every minute. More origin traffic. |

**Interview tip:** "For static assets like images and JS bundles, I'd use cache-busting by including a content hash in the filename. The CDN caches indefinitely, and when content changes, the new filename is a completely different URL — no invalidation needed."

### 4. What to Cache on a CDN

| Content Type | CDN? | TTL |
|-------------|------|-----|
| Images, videos | ✅ Yes | Long (hours/days/forever with cache busting) |
| CSS, JavaScript | ✅ Yes | Forever (with hash in filename) |
| HTML pages | Maybe | Short (5-60 seconds) for semi-static pages |
| API responses | Rarely | Very short or not at all (personalized data) |
| User-specific content | ❌ No | Never (private data) |

---

## Real-World Example — Netflix CDN (Open Connect)

```
Netflix built their OWN CDN called Open Connect:

1. Video is uploaded and transcoded into 100+ formats/qualities
2. Popular titles are PUSHED to ISP-hosted appliances
3. Less popular titles are PULLED on demand

┌─────────────────────────────────────────────────────────┐
│                 Netflix Open Connect                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────────┐                                     │
│  │ Central Storage │  (All content, all formats)        │
│  │ (AWS S3)        │                                     │
│  └───────┬────────┘                                     │
│          │ push popular titles                           │
│     ┌────┼────────────────────┐                         │
│     ▼    ▼                    ▼                         │
│  ┌────────┐  ┌────────┐  ┌────────┐                    │
│  │ISP Box  │  │ISP Box  │  │ISP Box  │                   │
│  │Comcast  │  │Verizon  │  │BT (UK)  │   ~18,000 servers │
│  │         │  │         │  │         │   in ~6,000 ISPs  │
│  └────────┘  └────────┘  └────────┘   worldwide         │
│                                                         │
│  When you stream: content comes from a box INSIDE       │
│  your ISP's network — minimal hops, maximum bandwidth   │
│                                                         │
│  Result: Netflix delivers 90%+ of traffic from          │
│  ISP-local servers, not from AWS.                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Common Interview Questions

### Q1: "When and why would you use a CDN?"

**Model answer:**
"I'd use a CDN whenever the system serves static content (images, videos, CSS, JS) to a geographically distributed user base. A CDN caches content at edge servers close to users, reducing latency from 100-200ms cross-continent to 10-30ms from a local POP. It also offloads bandwidth from the origin server and provides DDoS protection. For a system like Instagram, the CDN handles 90%+ of image serving. I'd use a pull CDN model by default, with long cache durations and cache-busting filenames for versioning."

### Q2: "How does a CDN work with DNS?"

**Model answer:**
"When a user requests content from `images.myapp.com`, DNS resolves to the CDN's network — typically using a CNAME record pointing to the CDN provider (e.g., `images.myapp.com CNAME d111.cloudfront.net`). The CDN's DNS uses anycast or GeoDNS to return the IP of the nearest edge server. The user's browser connects to that edge server. If the content is cached, it's served immediately. If not, the edge server fetches from the origin, caches it, and then serves it."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 13 CHEAT SHEET                        │
│                  CDN & Edge Computing                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CDN = Geographically distributed cache for static content      │
│  Pull CDN: fetch on first request (default)                     │
│  Push CDN: proactively upload popular content                   │
│                                                                  │
│  CACHE BUSTING: Include content hash in filename (best)         │
│  SHORT TTL: 5-60 seconds for semi-dynamic content               │
│  PURGE API: Force CDN to re-fetch (takes seconds)               │
│                                                                  │
│  WHAT TO CDN: Images ✅ Videos ✅ CSS/JS ✅ APIs ❌            │
│                                                                  │
│  PROVIDERS: CloudFront (AWS), Cloudflare, Akamai, Fastly       │
│  Netflix Open Connect: custom CDN inside ISPs                   │
│                                                                  │
│  INTERVIEW: "Serve static assets through CDN, origin for API"  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is a CDN? | A globally distributed network of servers that caches content at edge locations close to users, reducing latency. |
| 2 | Pull vs Push CDN? | Pull: fetches from origin on first request. Push: you proactively upload content to edge servers. |
| 3 | What is cache busting? | Including a content hash in filenames (`style.abc123.css`) so new content gets a new URL, bypassing old cache. |
| 4 | What is a POP? | Point of Presence — a physical location where CDN edge servers are deployed. Major CDNs have 200-400+ POPs. |
| 5 | What is Open Connect? | Netflix's custom CDN that places servers inside ISP networks to serve video from within the user's ISP. |
| 6 | How does DNS route users to the nearest CDN edge? | Using GeoDNS or Anycast — DNS returns the IP of the geographically closest or lowest-latency edge server. |

---

[← Previous: Caching](./chapter-12-caching.md) · [Next: Reverse Proxies & API Gateways →](./chapter-14-reverse-proxies-api-gateways.md)
