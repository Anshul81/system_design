# 🏗️ System Design Interview Prep — Complete Course

> **Crack system design interviews at junior-to-senior level (covers 90%+ of interview questions)**
>
> 📚 48 Chapters · 8 Phases · ~132 Hours · Interview-First

---

## 🎯 Course Philosophy

- **Interview-first** — every topic is taught through the lens of "what will the interviewer ask and what do they want to hear?"
- **Build mental models, not memorize answers** — understand trade-offs deeply so you can reason about systems you've never seen
- **Progressive depth** — start with fundamentals (networking, storage, compute) before jumping into full system designs
- **Real systems as case studies** — learn from how Twitter, Netflix, Uber, WhatsApp, and Google actually work
- **No ivory tower** — we skip pure academic distributed systems theory (Paxos proofs, FLP impossibility). We focus on what gets asked and what senior engineers actually use
- **Estimation-ready** — back-of-the-envelope calculations are practiced throughout, not just in one chapter
- **Both breadth AND depth** — cover all major system design topics, then go deep on the 19 most-asked design questions

---

## 🗺️ Visual Course Map

```
Week 1–2            Week 3–4            Week 5–6            Week 7–8            Week 9–10
──────────          ──────────          ──────────          ──────────          ──────────

🏗️ FOUNDATIONS       📦 BUILDING BLOCKS   🔧 INFRA & DATA      🎨 FULL DESIGNS      🎨 MORE DESIGNS +
                                                                                  STRATEGY

Phase 0              Phase 2             Phase 4              Phase 6             Phase 7
Networking &         Databases &         Message Queues,      Design URL          Design Web Crawler,
Communication        Storage             Microservices,       Shortener,          Ticket Booking,
                                         Search, Monitoring   Design Twitter,     E-Commerce,
Phase 1              Phase 3                                  Design Chat,        Google Docs,
Scalability          Load Balancing,     Phase 5              Design YouTube,     Payment System,
Fundamentals         Caching, Proxies    Distributed          Design Instagram,   etc.
                                         Systems              Design Uber,
                                         Concepts             Design Dropbox      Phase 8
                                                                                  Interview
                                                                                  Strategy
```

```
Dependency Flow:

Phase 0 ──→ Phase 1 ──→ Phase 2 ──→ Phase 3
                           │            │
                           ▼            ▼
                        Phase 4 ──→ Phase 5
                                       │
                                       ▼
                               Phase 6 ──→ Phase 7 ──→ Phase 8
```

---

## 📖 Table of Contents

### 🌐 [Phase 0 — Networking & Communication](./phase-0-networking/README.md) (~7.5 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 0 | [How the Internet Works — DNS, TCP, HTTP](./phase-0-networking/chapter-00-internet-dns-tcp-http.md) | 2.5 hrs | 🟢 Beginner |
| 1 | [APIs — REST, GraphQL, gRPC](./phase-0-networking/chapter-01-apis-rest-graphql-grpc.md) | 2.5 hrs | 🟢 Beginner |
| 2 | [Communication Patterns](./phase-0-networking/chapter-02-communication-patterns.md) | 2.5 hrs | 🟡 Medium |

### 📈 [Phase 1 — Scalability Fundamentals](./phase-1-scalability/README.md) (~7.5 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 3 | [Scaling 101 — Vertical vs Horizontal](./phase-1-scalability/chapter-03-scaling-101.md) | 2.5 hrs | 🟢 Beginner |
| 4 | [Back-of-the-Envelope Estimation](./phase-1-scalability/chapter-04-estimation.md) | 3 hrs | 🟡 Medium |
| 5 | [The System Design Interview Framework](./phase-1-scalability/chapter-05-interview-framework.md) | 2 hrs | 🟢 Beginner |

### 🗄️ [Phase 2 — Databases & Storage](./phase-2-databases/README.md) (~13 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 6 | [SQL Databases — Relational Modeling](./phase-2-databases/chapter-06-sql-databases.md) | 3 hrs | 🟡 Medium |
| 7 | [NoSQL Databases — Types & Trade-offs](./phase-2-databases/chapter-07-nosql-databases.md) | 3 hrs | 🟡 Medium |
| 8 | [Database Scaling — Replication & Sharding](./phase-2-databases/chapter-08-replication-sharding.md) | 3.5 hrs | 🟡 Medium |
| 9 | [SQL vs NoSQL — The Decision Framework](./phase-2-databases/chapter-09-sql-vs-nosql.md) | 2 hrs | 🟡 Medium |
| 10 | [Blob Storage & Data Lakes](./phase-2-databases/chapter-10-blob-storage.md) | 1.5 hrs | 🟢 Beginner |

### ⚖️ [Phase 3 — Load Balancing, Caching & Proxies](./phase-3-load-balancing-caching/README.md) (~10 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 11 | [Load Balancing](./phase-3-load-balancing-caching/chapter-11-load-balancing.md) | 2.5 hrs | 🟡 Medium |
| 12 | [Caching — Strategies & Patterns](./phase-3-load-balancing-caching/chapter-12-caching.md) | 3.5 hrs | 🟡 Medium |
| 13 | [CDN & Edge Computing](./phase-3-load-balancing-caching/chapter-13-cdn-edge.md) | 2 hrs | 🟡 Medium |
| 14 | [Reverse Proxies & API Gateways](./phase-3-load-balancing-caching/chapter-14-reverse-proxies-api-gateways.md) | 2 hrs | 🟡 Medium |

### 🔧 [Phase 4 — Infrastructure & Middleware](./phase-4-infrastructure/README.md) (~10.5 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 15 | [Message Queues & Event Streaming](./phase-4-infrastructure/chapter-15-message-queues.md) | 3.5 hrs | 🟡 Medium |
| 16 | [Microservices vs Monolith](./phase-4-infrastructure/chapter-16-microservices-vs-monolith.md) | 2.5 hrs | 🟡 Medium |
| 17 | [Search Systems](./phase-4-infrastructure/chapter-17-search-systems.md) | 2.5 hrs | 🟡 Medium |
| 18 | [Monitoring, Logging & Alerting](./phase-4-infrastructure/chapter-18-monitoring-logging.md) | 2 hrs | 🟡 Medium |

### 🌍 [Phase 5 — Distributed Systems Concepts](./phase-5-distributed-systems/README.md) (~16 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 19 | [Rate Limiting & Throttling](./phase-5-distributed-systems/chapter-19-rate-limiting.md) | 2.5 hrs | 🟡 Medium |
| 20 | [Unique ID Generation](./phase-5-distributed-systems/chapter-20-unique-id-generation.md) | 2 hrs | 🟡 Medium |
| 21 | [Consensus & Leader Election](./phase-5-distributed-systems/chapter-21-consensus-leader-election.md) | 2.5 hrs | 🔴 Hard |
| 22 | [Consistent Hashing](./phase-5-distributed-systems/chapter-22-consistent-hashing.md) | 2.5 hrs | 🟡 Medium |
| 23 | [Distributed Transactions](./phase-5-distributed-systems/chapter-23-distributed-transactions.md) | 3 hrs | 🔴 Hard |
| 24 | [Availability Patterns & SLAs](./phase-5-distributed-systems/chapter-24-availability-patterns.md) | 3 hrs | 🟡 Medium |

### 🎨 [Phase 6 — Classic System Design Problems](./phase-6-classic-designs/README.md) (~28 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 25 | [Design URL Shortener (TinyURL)](./phase-6-classic-designs/chapter-25-url-shortener.md) | 3 hrs | 🟡 Medium |
| 26 | [Design Twitter / News Feed](./phase-6-classic-designs/chapter-26-twitter-news-feed.md) | 4 hrs | 🔴 Hard |
| 27 | [Design Chat System (WhatsApp / Slack)](./phase-6-classic-designs/chapter-27-chat-system.md) | 4 hrs | 🔴 Hard |
| 28 | [Design YouTube / Video Streaming](./phase-6-classic-designs/chapter-28-youtube-video-streaming.md) | 3.5 hrs | 🔴 Hard |
| 29 | [Design Instagram / Photo Sharing](./phase-6-classic-designs/chapter-29-instagram-photo-sharing.md) | 3 hrs | 🟡 Medium |
| 30 | [Design Uber / Ride Sharing](./phase-6-classic-designs/chapter-30-uber-ride-sharing.md) | 4 hrs | 🔴 Hard |
| 31 | [Design Dropbox / Google Drive](./phase-6-classic-designs/chapter-31-dropbox-google-drive.md) | 4 hrs | 🔴 Hard |
| 32 | [Design Notification System](./phase-6-classic-designs/chapter-32-notification-system.md) | 3 hrs | 🟡 Medium |

### 🚀 [Phase 7 — Advanced System Design Problems](./phase-7-advanced-designs/README.md) (~35 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 33 | [Design Web Crawler](./phase-7-advanced-designs/chapter-33-web-crawler.md) | 3 hrs | 🔴 Hard |
| 34 | [Design Ticket Booking (Ticketmaster)](./phase-7-advanced-designs/chapter-34-ticket-booking.md) | 3.5 hrs | 🔴 Hard |
| 35 | [Design Google Maps / Proximity Service](./phase-7-advanced-designs/chapter-35-proximity-service.md) | 3 hrs | 🔴 Hard |
| 36 | [Design E-Commerce (Amazon)](./phase-7-advanced-designs/chapter-36-ecommerce.md) | 3.5 hrs | 🔴 Hard |
| 37 | [Design Distributed Cache](./phase-7-advanced-designs/chapter-37-distributed-cache.md) | 3 hrs | 🔴 Hard |
| 38 | [Design Google Docs / Collaborative Editing](./phase-7-advanced-designs/chapter-38-collaborative-editing.md) | 3.5 hrs | 🔴 Hard |
| 39 | [Design Metrics & Logging Pipeline](./phase-7-advanced-designs/chapter-39-metrics-logging-pipeline.md) | 3 hrs | 🟡 Medium |
| 40 | [Design Payment System](./phase-7-advanced-designs/chapter-40-payment-system.md) | 3.5 hrs | 🔴 Hard |
| 41 | [Design Ad Click Aggregation](./phase-7-advanced-designs/chapter-41-ad-click-aggregation.md) | 3 hrs | 🟡 Medium |
| 42 | [Design Key-Value Store](./phase-7-advanced-designs/chapter-42-key-value-store.md) | 3 hrs | 🔴 Hard |
| 43 | [Design Leaderboard / Ranking System](./phase-7-advanced-designs/chapter-43-leaderboard.md) | 2.5 hrs | 🟡 Medium |

### 🎤 [Phase 8 — Interview Strategy](./phase-8-final-boss/README.md) (~9 hrs)

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 44 | [The System Design Interview Framework (Detailed)](./phase-8-final-boss/chapter-44-interview-framework-detailed.md) | 2.5 hrs | 🟢 Beginner |
| 45 | [Functional vs Non-Functional Requirements](./phase-8-final-boss/chapter-45-requirements-deep-dive.md) | 2 hrs | 🟢 Beginner |
| 46 | [Trade-off Analysis](./phase-8-final-boss/chapter-46-tradeoff-analysis.md) | 2.5 hrs | 🟡 Medium |
| 47 | [Common Mistakes & Anti-Patterns](./phase-8-final-boss/chapter-47-common-mistakes.md) | 2 hrs | 🟡 Medium |

---

## 📊 Time Summary

| Phase | Topic | Chapters | Hours |
|-------|-------|----------|-------|
| Phase 0 | Networking & Communication | 3 | 7.5 |
| Phase 1 | Scalability Fundamentals | 3 | 7.5 |
| Phase 2 | Databases & Storage | 5 | 13 |
| Phase 3 | Load Balancing, Caching & Proxies | 4 | 10 |
| Phase 4 | Infrastructure & Middleware | 4 | 10.5 |
| Phase 5 | Distributed Systems Concepts | 6 | 16 |
| Phase 6 | Classic Design Problems | 8 | 28 |
| Phase 7 | Advanced Design Problems | 11 | 35 |
| Phase 8 | Interview Strategy | 4 | 9 |
| | **TOTAL** | **48** | **~136** |

---

## 📅 Study Schedules

### 🏃 10-Week Plan (~14 hrs/week) — Recommended

| Week | Phases | Focus |
|------|--------|-------|
| 1 | Phase 0 + Phase 1 | Networking, APIs, scaling, estimation, framework |
| 2 | Phase 2 | Databases — SQL, NoSQL, replication, sharding |
| 3 | Phase 3 + Phase 4 (Ch 15-16) | Load balancers, caching, CDN, queues, microservices |
| 4 | Phase 4 (Ch 17-18) + Phase 5 (Ch 19-21) | Search, monitoring, rate limiting, IDs, consensus |
| 5 | Phase 5 (Ch 22-24) + Phase 6 (Ch 25) | Consistent hashing, transactions, availability, URL shortener |
| 6 | Phase 6 (Ch 26-29) | Twitter, chat system, YouTube, Instagram |
| 7 | Phase 6 (Ch 30-32) + Phase 7 (Ch 33) | Uber, Dropbox, notifications, web crawler |
| 8 | Phase 7 (Ch 34-37) | Ticketmaster, Google Maps, Amazon, distributed cache |
| 9 | Phase 7 (Ch 38-43) | Google Docs, metrics, payments, ad clicks, KV store, leaderboard |
| 10 | Phase 8 + Review | Interview strategy, mock interviews, weak area revision |

### 🐢 14-Week Plan (~10 hrs/week)

| Week | Phases | Focus |
|------|--------|-------|
| 1 | Phase 0 | Networking, APIs, communication patterns |
| 2 | Phase 1 | Scaling, estimation, interview framework |
| 3 | Phase 2 (Ch 6-8) | SQL, NoSQL, replication & sharding |
| 4 | Phase 2 (Ch 9-10) + Phase 3 (Ch 11) | SQL vs NoSQL, blob storage, load balancing |
| 5 | Phase 3 (Ch 12-14) | Caching, CDN, API gateways |
| 6 | Phase 4 | Message queues, microservices, search, monitoring |
| 7 | Phase 5 (Ch 19-22) | Rate limiting, IDs, consensus, consistent hashing |
| 8 | Phase 5 (Ch 23-24) + Phase 6 (Ch 25-26) | Transactions, availability, URL shortener, Twitter |
| 9 | Phase 6 (Ch 27-30) | Chat, YouTube, Instagram, Uber |
| 10 | Phase 6 (Ch 31-32) + Phase 7 (Ch 33-34) | Dropbox, notifications, web crawler, Ticketmaster |
| 11 | Phase 7 (Ch 35-38) | Google Maps, Amazon, distributed cache, Google Docs |
| 12 | Phase 7 (Ch 39-41) | Metrics pipeline, payments, ad clicks |
| 13 | Phase 7 (Ch 42-43) + Phase 8 (Ch 44-45) | KV store, leaderboard, interview framework, requirements |
| 14 | Phase 8 (Ch 46-47) + Review | Trade-offs, anti-patterns, mock interviews |

---

## 📚 Key References

| Resource | Use |
|----------|-----|
| [Latency Numbers Reference](./LATENCY_REFERENCE.md) | Numbers used throughout the course |
| *System Design Interview* — Alex Xu (Vol 1 & 2) | Gold standard for structured walkthroughs |
| *Designing Data-Intensive Applications* — Martin Kleppmann | Deep DB & distributed systems understanding |
| [ByteByteGo](https://bytebytego.com) | Visual system design explanations |

---

## 🧭 How to Use This Course

1. **Follow the phases in order** — each builds on the previous one
2. **Do the checkpoint exercises** at the end of each phase
3. **Use the flashcards** for spaced repetition review
4. **Practice estimation** on every design problem — show your math
5. **Whiteboard designs** from memory after studying each design chapter
6. **Time yourself** — aim for 35-40 minutes per design in practice
7. **Review the cheat sheets** the night before an interview
