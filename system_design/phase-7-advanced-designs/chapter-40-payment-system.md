# Chapter 40: Design a Payment System

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3.5 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Metrics Pipeline](./chapter-39-metrics-logging-pipeline.md) · [Next: Ad Click Aggregation →](./chapter-41-ad-click-aggregation.md)

---

## Why This Problem Matters

Payments are the hardest consistency problem. Money cannot be lost, duplicated, or miscounted. This tests idempotency, exactly-once processing, reconciliation, and handling failures in distributed systems where correctness is non-negotiable.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Transactions/day** | 10M |
| **Correctness** | ZERO financial discrepancies |
| **Idempotency** | No double charges |
| **Availability** | 99.99% |
| **Compliance** | PCI DSS, audit trail |

---

## Architecture

```
┌──────────┐    ┌───────────────────────────────────────────────┐
│ Checkout  │───▶│           PAYMENT SERVICE                     │
│ Service   │    │                                               │
└──────────┘    │  1. Validate request + idempotency check     │
                │  2. Create payment record (PENDING)            │
                │  3. Call Payment Gateway (Stripe/Adyen)        │
                │  4. Handle response (SUCCESS/FAILURE)          │
                │  5. Update payment record (COMPLETED/FAILED)  │
                │  6. Notify order service                       │
                │                                               │
                │  ┌──────────────────────┐                     │
                │  │ Payment DB (MySQL)    │                     │
                │  │ • ACID transactions   │                     │
                │  │ • Idempotency keys    │                     │
                │  │ • Full audit log      │                     │
                │  └──────────────────────┘                     │
                │                                               │
                │  ┌──────────────────────┐                     │
                │  │ Payment Gateway       │ (Stripe, Adyen)    │
                │  │ (external 3rd party)  │                     │
                │  └──────────────────────┘                     │
                └───────────────────────────────────────────────┘
```

### Key Design Decisions

**1. Idempotency (The Most Critical Feature):**
```
PROBLEM: Network timeout during payment.
  Client → Payment Service → Stripe: "Charge $50"
  Stripe charges $50, but response is lost (network timeout)
  Client retries → Payment Service → Stripe: "Charge $50" AGAIN
  → $100 charged instead of $50!

SOLUTION: Idempotency Key
  Each payment request has a unique key: {order_id}_{attempt}
  
  Payment Service:
    1. Check DB: does idempotency_key exist?
       → Yes: return stored result (don't reprocess)
       → No: proceed with payment
    2. INSERT INTO payments (idempotency_key, status='PENDING')
    3. Call Stripe with idempotency_key
    4. Update payment status based on Stripe response

  Stripe also supports idempotency keys internally.
  Double protection: our key + Stripe's key.
```

**2. Payment State Machine:**
```
┌─────────┐     ┌─────────┐     ┌───────────┐
│ CREATED  │────▶│ PENDING  │────▶│ COMPLETED  │
└─────────┘     └────┬────┘     └───────────┘
                      │
                      ├────────▶ ┌─────────┐
                      │          │ FAILED   │
                      │          └─────────┘
                      │
                      └────────▶ ┌─────────┐
                                 │ TIMED_OUT│──▶ Reconcile
                                 └─────────┘

Every state transition is logged for audit.
State can only move forward (no going back from COMPLETED).
```

**3. Reconciliation:**
```
PROBLEM: Payment service says PENDING but Stripe says COMPLETED.
  (response was lost)

SOLUTION: Daily reconciliation job
  1. Pull all transactions from Stripe for the day
  2. Compare with internal payment DB
  3. Flag discrepancies:
     - We say PENDING, Stripe says COMPLETED → update to COMPLETED
     - We say COMPLETED, Stripe has no record → ALERT!
     - Amount mismatch → ALERT!
  
  Run every hour for recent transactions, daily for full reconciliation.
```

**4. Double-Entry Bookkeeping:**
```
Every transaction creates TWO entries:

  Debit:  Customer Account  -$50 (money leaves customer)
  Credit: Merchant Account   +$50 (money enters merchant)

  SUM of all debits = SUM of all credits (always balanced)

  If they don't balance → something is wrong → ALERT!
  This is how banks detect errors and fraud.
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Database | MySQL with ACID | Financial data requires strong consistency. No eventual consistency. |
| Idempotency | Idempotency key in DB + Stripe | Network failures are inevitable. Idempotency prevents double charges. |
| Gateway | Third-party (Stripe/Adyen) | Don't handle credit card numbers directly. PCI DSS compliance. |
| Consistency | Strong (synchronous confirmation) | Payment must be confirmed before telling user "order placed." |
| Reconciliation | Hourly + daily batch | Catches discrepancies between internal state and gateway state. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you prevent double charges? | Idempotency key per payment. Check DB before processing. If key exists, return stored result instead of reprocessing. |
| 2 | What happens when a payment gateway call times out? | Payment stays PENDING. Reconciliation job later checks gateway status and updates. Never retry blindly — use idempotency key. |
| 3 | What is reconciliation? | Comparing internal payment records with the payment gateway's records to find and resolve discrepancies. Runs hourly/daily. |
| 4 | Why use a third-party payment gateway? | Handling credit card numbers requires PCI DSS compliance (extremely expensive). Stripe/Adyen handle card data, we handle business logic. |
| 5 | What is double-entry bookkeeping? | Every transaction has a debit and credit entry. Total debits = total credits. Imbalance indicates an error. |

---

[← Previous: Metrics Pipeline](./chapter-39-metrics-logging-pipeline.md) · [Next: Ad Click Aggregation →](./chapter-41-ad-click-aggregation.md)
