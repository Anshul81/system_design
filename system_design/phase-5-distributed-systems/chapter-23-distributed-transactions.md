# Chapter 23: Distributed Transactions

> **Phase 5 — Distributed Systems Concepts** · Estimated Time: 3 hours · Difficulty: 🔴 Hard

[← Previous: Consistent Hashing](./chapter-22-consistent-hashing.md) · [Next: Availability Patterns →](./chapter-24-availability-patterns.md)

---

## Learning Objectives

- ✅ Explain the Two-Phase Commit (2PC) protocol and its limitations
- ✅ Compare 2PC with the Saga pattern for distributed transactions
- ✅ Understand eventual consistency and how to design systems around it
- ✅ Apply the Outbox pattern for reliable event publishing
- ✅ Choose between strong and eventual consistency for different use cases

---

## Introduction — Why This Matters

**Interview relevance:** In microservices, you can't use a single database transaction across services. When placing an order involves the Order Service, Payment Service, and Inventory Service — each with its own database — how do you ensure all succeed or all fail? The Saga pattern and eventual consistency are the interview-expected answers.

---

## Core Concepts

### 1. The Problem

```
Monolith (single DB transaction):
  BEGIN TRANSACTION
    INSERT INTO orders (...)
    UPDATE inventory SET stock = stock - 1
    INSERT INTO payments (...)
  COMMIT   ← All succeed or all fail. ACID guarantees this.

Microservices (separate DBs — NO cross-service transaction!):
  Order Service (MySQL 1): INSERT INTO orders ✅
  Payment Service (MySQL 2): INSERT INTO payments ✅
  Inventory Service (MySQL 3): UPDATE stock ❌ (FAILS!)

  → Order and Payment succeeded, but Inventory failed.
  → INCONSISTENT STATE! How do we roll back?
```

### 2. Two-Phase Commit (2PC)

```
Coordinator asks all participants to prepare, then commit:

Phase 1 — PREPARE:
  Coordinator → "Can you commit?" → Participant A → "Yes (PREPARED)"
  Coordinator → "Can you commit?" → Participant B → "Yes (PREPARED)"
  Coordinator → "Can you commit?" → Participant C → "Yes (PREPARED)"

Phase 2 — COMMIT:
  Coordinator → "COMMIT" → Participant A → ✅ Done
  Coordinator → "COMMIT" → Participant B → ✅ Done
  Coordinator → "COMMIT" → Participant C → ✅ Done

If any participant says "No" in Phase 1:
  Coordinator → "ABORT" → All participants → Rollback
```

| Pros | Cons |
|------|------|
| Strong consistency | **Blocking:** If coordinator crashes after Phase 1, participants are stuck holding locks |
| Simple mental model | **Slow:** 2 round trips, all participants must be available |
| | **SPOF:** Coordinator is a single point of failure |
| | **Not scalable:** Locks held across services for the duration |

**Verdict:** 2PC is rarely used in modern microservices. It's too slow and fragile. But know it for interviews — it's the "textbook answer" you then explain why you'd avoid.

### 3. Saga Pattern (The Modern Answer)

```
CHOREOGRAPHY-BASED SAGA (event-driven):

  ┌───────────┐  OrderCreated  ┌───────────┐  PaymentDone  ┌───────────┐
  │Order Svc   │──────────────▶│Payment Svc │──────────────▶│Inventory  │
  │Create order│               │Charge card │               │Reserve    │
  └───────────┘               └───────────┘               └───────────┘

  If Inventory fails:
  ┌───────────┐  RefundPayment ┌───────────┐ InventoryFailed┌───────────┐
  │Order Svc   │◀──────────────│Payment Svc │◀──────────────│Inventory  │
  │Cancel order│               │Refund card │               │Failed     │
  └───────────┘               └───────────┘               └───────────┘

  Each step has a COMPENSATING ACTION for rollback.

ORCHESTRATION-BASED SAGA (central coordinator):

  ┌────────────────┐
  │ Saga            │
  │ Orchestrator    │
  │                │
  │ 1. Create order │──▶ Order Service
  │ 2. Charge card  │──▶ Payment Service
  │ 3. Reserve item │──▶ Inventory Service
  │                │
  │ If step 3 fails:│
  │ 3. Compensate 2 │──▶ Payment Service (refund)
  │ 4. Compensate 1 │──▶ Order Service (cancel)
  └────────────────┘
```

| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| **Coordination** | Decentralized (events) | Central orchestrator |
| **Coupling** | Loose | Tighter (orchestrator knows all steps) |
| **Visibility** | Hard to see full flow | Easy to see in orchestrator |
| **Complexity** | Can become tangled (event chains) | Orchestrator can become complex |
| **Best for** | Simple sagas (2-3 steps) | Complex sagas (4+ steps) |

### 4. The Outbox Pattern

Problem: How do you atomically update a database AND publish an event?

```
WRONG (dual writes — can fail between steps):
  1. UPDATE orders SET status='CONFIRMED'  ← succeeds
  2. Publish event to Kafka                ← FAILS (network issue)
  → DB updated but event never sent!

OUTBOX PATTERN (correct):
  1. In a SINGLE DB transaction:
     - UPDATE orders SET status='CONFIRMED'
     - INSERT INTO outbox (event_type='OrderConfirmed', payload='...')
  2. A separate process reads the outbox table and publishes to Kafka
  3. After publishing, mark the outbox entry as sent

  ┌──────────────────────┐        ┌─────────┐
  │ Order Service DB      │        │  Kafka   │
  │                       │        │          │
  │ orders table          │        │          │
  │ outbox table ─────────┼──CDC──▶│  events  │
  │  (event_type, payload,│        │          │
  │   sent_at)            │        │          │
  └──────────────────────┘        └─────────┘

  Atomicity guaranteed by the single DB transaction.
  CDC (Change Data Capture) or a poller reads the outbox.
```

### 5. Eventual Consistency Guidelines

```
INTERVIEW PATTERN:

"For this operation, do we need strong or eventual consistency?"

Strong consistency (pay the performance cost):
  • Payment processing
  • Account balance checks
  • Inventory for limited items (flash sales)

Eventual consistency (accept the speed benefit):
  • Like/follower counts
  • News feed updates
  • Search index updates
  • Analytics and metrics
  • Notification delivery

"The system is eventually consistent — a tweet might take
1-2 seconds to appear in all followers' feeds, which is
acceptable for a social media platform."
```

---

## Common Interview Questions

### Q1: "How do you handle distributed transactions in microservices?"

**Model answer:**
"I'd use the Saga pattern instead of 2PC. Each service performs its local transaction and publishes an event. Downstream services react to events. If any step fails, compensating transactions undo the previous steps. For example, if payment succeeds but inventory reservation fails, a compensating transaction refunds the payment. For reliable event publishing, I'd use the Outbox pattern — write the event to an outbox table in the same transaction as the business data, then a separate process publishes it to Kafka. This avoids the dual-write problem."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 23 CHEAT SHEET                        │
│                  Distributed Transactions                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  2PC: Prepare → Commit. Strong consistency but blocking, slow.  │
│  SAGA: Chain of local transactions + compensating actions.      │
│    Choreography: event-driven (loose coupling)                  │
│    Orchestration: central coordinator (clear flow)              │
│                                                                  │
│  OUTBOX PATTERN: Write event to outbox table in same DB         │
│  transaction. Separate process publishes to Kafka. Atomic.      │
│                                                                  │
│  STRONG CONSISTENCY: Payments, account balances, inventory      │
│  EVENTUAL CONSISTENCY: Feeds, counts, search, notifications    │
│                                                                  │
│  INTERVIEW: "Saga pattern with outbox for reliable events"      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | Why can't you use database transactions across microservices? | Each service has its own database. DB transactions are local to one database — they can't span multiple databases across services. |
| 2 | What is 2PC and why is it avoided? | Two-Phase Commit: coordinator asks all to prepare, then commit. Avoided because it's blocking, slow, and the coordinator is a SPOF. |
| 3 | What is the Saga pattern? | A sequence of local transactions across services. Each step has a compensating action for rollback. Either choreography (events) or orchestration (coordinator). |
| 4 | What is the Outbox pattern? | Write business data AND event to the same DB in one transaction. A separate process reads the outbox and publishes to the message broker. Solves the dual-write problem. |
| 5 | When is eventual consistency acceptable? | For non-critical derived data: social media feeds, like counts, search indexes, analytics. Not for payments or inventory for limited items. |

---

[← Previous: Consistent Hashing](./chapter-22-consistent-hashing.md) · [Next: Availability Patterns →](./chapter-24-availability-patterns.md)
