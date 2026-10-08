# Chapter 34: Design Ticket Booking (Ticketmaster)

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3.5 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Web Crawler](./chapter-33-web-crawler.md) · [Next: Proximity Service →](./chapter-35-proximity-service.md)

---

## Why This Problem Matters

Ticket booking is THE concurrency challenge. When a hot concert goes on sale, millions of users race for thousands of seats simultaneously. The core problems: preventing overselling, handling extreme traffic spikes, and implementing fair queuing.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Events** | 100K active events |
| **Peak traffic** | 10M users in queue for hot events |
| **Seats per event** | 1K - 100K |
| **Booking SLA** | No overselling (CRITICAL) |
| **Fairness** | First-come-first-served |

---

## Architecture

```
┌────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────────┐
│ Users   │───▶│ CDN +    │───▶│ Virtual Queue │───▶│ Booking Svc  │
│ (10M)   │    │ Static   │    │ Service       │    │              │
└────────┘    │ Pages   │    │ (rate-limit  │    │ Reserve seat │
              └─────────┘    │  entry)       │    │ Process pay  │
                              └──────────────┘    │ Confirm      │
                                                   └──────┬───────┘
                                                          │
                                                   ┌──────▼───────┐
                                                   │ Seat Inventory│
                                                   │ (MySQL + Redis│
                                                   │  distributed  │
                                                   │  lock)        │
                                                   └──────────────┘
```

### The Core Problem: Preventing Overselling

```
WRONG — Race Condition:
  Thread 1: SELECT available FROM seats WHERE id=A1 → available=true
  Thread 2: SELECT available FROM seats WHERE id=A1 → available=true
  Thread 1: UPDATE seats SET available=false WHERE id=A1 ✅
  Thread 2: UPDATE seats SET available=false WHERE id=A1 ✅
  → SEAT SOLD TWICE!

SOLUTION 1 — Optimistic Locking:
  SELECT available, version FROM seats WHERE id=A1
  → available=true, version=1

  UPDATE seats SET available=false, version=2
  WHERE id=A1 AND version=1
  → rows_affected=1 → SUCCESS

  Second thread:
  UPDATE seats SET available=false, version=2
  WHERE id=A1 AND version=1
  → rows_affected=0 → CONFLICT, seat already taken!

SOLUTION 2 — Redis Distributed Lock:
  SETNX lock:seat:A1 {user_id} EX 300  (5 min hold)
  → If SET: user has lock, proceed to payment
  → If NOT SET: seat locked by another user

  After payment: DELETE lock:seat:A1, mark seat as SOLD in DB
  If user doesn't pay in 5 min: lock expires, seat released
```

### Virtual Queue (Traffic Management)

```
10M users can't all hit the booking API simultaneously.

VIRTUAL QUEUE:
  1. Users join queue → assigned position number
  2. System admits N users/sec to booking flow
  3. Users wait on a static page (served from CDN)
  4. When admitted: "It's your turn!" → redirect to booking

┌──────────────────────────────────────────────────┐
│ 10M users → Queue → Admit 1000/sec → Booking API │
│                                                    │
│ Position 1-1000: NOW BOOKING                      │
│ Position 1001-5000: Estimated wait: 4 min          │
│ Position 5001-10000: Estimated wait: 10 min        │
│ Position 10000+: Estimated wait: 15+ min           │
└──────────────────────────────────────────────────┘

Queue page is STATIC (CDN) with a WebSocket for position updates.
Backend only handles 1000 concurrent booking sessions.
```

### Seat Reservation Flow

```
1. User selects seats → POST /reserve
2. Server acquires lock (Redis SETNX, 5 min TTL)
   → Lock acquired: seats reserved, start payment timer
   → Lock failed: "Seats no longer available"
3. User has 5 minutes to complete payment
4. Payment successful → mark seats SOLD in DB, release lock
5. Payment timeout → lock expires, seats auto-released
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Concurrency control | Optimistic locking (DB) + Redis locks | DB optimistic lock for correctness. Redis for fast seat holds with auto-expiry. |
| Traffic spike | Virtual queue with CDN | 10M users can't hit booking API. Queue + admit N/sec protects backend. |
| Seat hold | 5-min TTL on Redis lock | Prevents indefinite seat blocking. Auto-releases if user abandons. |
| Payment | Async with timeout | User has a time window. If exceeded, seat released for others. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you prevent overselling? | Optimistic locking (version column) or Redis distributed locks (SETNX with TTL). Both prevent two users from buying the same seat. |
| 2 | How do you handle 10M users trying to book at once? | Virtual queue: users wait on a CDN-served page. System admits a controlled rate (1000/sec) to the actual booking flow. |
| 3 | What happens if a user selects a seat but doesn't pay? | Seat lock has a 5-min TTL in Redis. If payment not completed, lock auto-expires and seat becomes available for others. |
| 4 | Optimistic vs pessimistic locking for seat booking? | Optimistic: check version on update (works for moderate contention). Pessimistic (SELECT FOR UPDATE): holds DB lock (works for high contention). |

---

[← Previous: Web Crawler](./chapter-33-web-crawler.md) · [Next: Proximity Service →](./chapter-35-proximity-service.md)
