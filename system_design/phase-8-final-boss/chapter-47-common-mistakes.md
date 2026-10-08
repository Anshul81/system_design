# Chapter 47: Common Mistakes & Anti-Patterns

> **Phase 8 — Interview Strategy** · Estimated Time: 2 hours · Difficulty: 🟡 Medium · Type: 📋 STRATEGY

[← Previous: Trade-off Analysis](./chapter-46-tradeoff-analysis.md)

---

## Why This Chapter Matters

After coaching hundreds of system design interviews, the same mistakes appear again and again. This chapter is a checklist of anti-patterns to avoid. Read it the night before your interview.

---

## Mistake #1: Jumping Into Design Without Requirements

```
THE MISTAKE:
  Interviewer: "Design a URL shortener."
  Candidate: "OK, so I'll have a MySQL database with a URLs table..."

WHY IT'S BAD:
  • You don't know the scale → might under-engineer or over-engineer
  • You don't know the priority → might focus on the wrong thing
  • You miss a chance to show product thinking
  • The interviewer can't tell if you're building the right thing

THE FIX:
  Spend 5 minutes on requirements:
  "Before I design, let me clarify the scope. Are we focusing on
   URL creation and redirection? What's the expected scale — 
   millions or billions of URLs? Do we need analytics/click tracking?"
  
  Then summarize:
  "Great, so I'll focus on: create short URL, redirect, and analytics.
   100M new URLs/day, 100:1 read:write ratio, < 100ms redirect latency."
```

---

## Mistake #2: Over-Engineering

```
THE MISTAKE:
  "For our URL shortener, I'll use Kafka for event streaming, 
   Flink for real-time processing, Cassandra for storage, Redis 
   for caching, Elasticsearch for search, and Kubernetes with 
   a service mesh for orchestration."

WHY IT'S BAD:
  • A URL shortener at 1,200 writes/sec doesn't need Kafka or Flink
  • Complexity without justification signals that you don't understand 
    what problems each technology solves
  • Interviewer will ask "why?" and you won't have a good answer

THE FIX:
  Start simple. Add complexity only when requirements demand it.
  "MySQL handles 1,200 writes/sec easily. I don't need Cassandra unless
   we scale beyond what MySQL with sharding can handle."
  
  RULE OF THUMB:
    If you can't explain WHY you need a technology, remove it.
```

---

## Mistake #3: Under-Engineering

```
THE MISTAKE:
  "I'll just have one server with a database."
  (for a system that needs to serve 500K requests/sec)

WHY IT'S BAD:
  • Ignores the scale requirements you just gathered
  • Shows lack of understanding of distributed systems
  • No fault tolerance, no scalability story

THE FIX:
  Let the estimation drive your architecture.
  "At 500K reads/sec, a single MySQL handles ~10K. I need a cache layer
   (Redis) to absorb 90%+ of reads, and I'll add read replicas for
   the remaining cache misses."
```

---

## Mistake #4: Not Discussing Trade-offs

```
THE MISTAKE:
  "I'll use Cassandra." (no explanation of why)
  "I'll use WebSocket." (no comparison with alternatives)

WHY IT'S BAD:
  Trade-off discussion is literally 20% of the interview score.
  Without it, you look like you're memorizing answers instead 
  of understanding systems.

THE FIX:
  For every major decision, mention the alternative:
  "I'll use Cassandra over MySQL because we have 50B writes/day
   and need to scale horizontally. The trade-off is we lose JOIN
   support, but our access pattern is simple (partition key lookup),
   so that's acceptable."
```

---

## Mistake #5: Ignoring the Data Model

```
THE MISTAKE:
  Beautiful architecture diagram with boxes and arrows,
  but no mention of what data is stored or how.

WHY IT'S BAD:
  The data model IS the design. It determines:
    • Which queries are fast or slow
    • How you scale (shard key)
    • What consistency model you need

THE FIX:
  Show at least the key tables/collections:
  "The core table is 'urls' with columns: short_code (PK), 
   original_url, created_at, click_count. I'll shard by 
   hash(short_code) for even distribution."
```

---

## Mistake #6: Single Point of Failure

```
THE MISTAKE:
  Architecture has ONE database, ONE cache, ONE API server
  with no replication, failover, or redundancy.

WHY IT'S BAD:
  If any single component fails → entire system goes down.
  For a 99.99% availability target, this is a non-starter.

THE FIX:
  Everything critical should have redundancy:
    • Database: primary + replicas (automatic failover)
    • Cache: Redis Cluster or Sentinel (replicated)
    • App servers: multiple behind load balancer (stateless)
    • Load balancer: active-passive pair
    • Message queue: Kafka cluster (3+ brokers)

  "I have MySQL with a primary and 2 read replicas with automatic 
   failover. If the primary goes down, a replica is promoted 
   within seconds."
```

---

## Mistake #7: Not Knowing Your Numbers

```
THE MISTAKE:
  "I'll use a database."
  → How many QPS?
  "Um... a lot?"

WHY IT'S BAD:
  Without numbers, you can't justify technology choices.
  Without numbers, you can't identify bottlenecks.

THE FIX:
  Memorize these baselines:
    Single MySQL:       ~10K simple query QPS
    Single Redis:       ~100K ops/sec
    Single app server:  ~10K concurrent connections
    1 KB × 1M/sec:     ~1 GB/sec bandwidth
    
  Do the math:
    "120K read QPS, Redis handles 100K → 1 Redis instance covers 
     80% of reads. Remaining 20% (24K) → DB with read replicas."
```

---

## Mistake #8: Designing in Silence

```
THE MISTAKE:
  *Candidate draws silently for 3 minutes*
  *Interviewer has no idea what's happening*

WHY IT'S BAD:
  The interviewer can't evaluate thinking they can't see.
  System design is a COLLABORATIVE exercise.

THE FIX:
  Narrate everything:
  "I'm thinking about whether to use push or pull for the timeline.
   Push means pre-computing timelines which is fast for reads but 
   expensive for celebrities. Pull means computing on the fly which
   is slow but avoids the celebrity problem. Let me go with a hybrid..."
```

---

## Mistake #9: Not Handling Edge Cases

```
EDGE CASES TO ALWAYS CONSIDER:

1. CELEBRITY/HOT KEY PROBLEM:
   One entity gets millions of requests → overwhelms one shard/server.
   Fix: Split hot key across multiple shards, or cache separately.

2. THUNDERING HERD:
   Cache expires → thousands of requests hit DB simultaneously.
   Fix: Cache stampede protection (lock on miss, request coalescing).

3. DATA RACE:
   Two users book the same seat/buy the last item simultaneously.
   Fix: Optimistic/pessimistic locking, idempotency.

4. NETWORK PARTITION:
   Communication between data centers fails.
   Fix: Choose AP or CP (CAP theorem). Document the behavior.

5. CASCADE FAILURE:
   One service goes down → requests queue up → dependent services 
   also go down.
   Fix: Circuit breaker pattern, timeouts, bulkheading.
```

---

## Mistake #10: Not Drawing a Clear Diagram

```
THE MISTAKE:
  Boxes floating with no labels, no arrows, no flow direction.

WHY IT'S BAD:
  The diagram IS your design. If it's messy, your design is messy.

THE FIX:
  Follow this layout:
  
  LEFT: Clients / Users
  CENTER: Services (with labels)
  RIGHT: Data stores (with specific technology names)
  ARROWS: Labeled with what data flows (not just lines)
  
  "Client → API Gateway → Tweet Service → MySQL (tweets)"
  Not: "Box → Box → Box"
```

---

## Pre-Interview Checklist

```
THE NIGHT BEFORE:
  □ Review the 4-step framework (Ch 44)
  □ Review the estimation reference numbers
  □ Read through your weakest 2 design problems
  □ Review this anti-patterns checklist

15 MINUTES BEFORE:
  □ Take a deep breath
  □ Have pen/paper or whiteboard ready
  □ Remember: the interviewer WANTS you to succeed
  □ Remember: narrate your thinking

DURING THE INTERVIEW:
  □ Step 1: Requirements (5 min) ← DON'T SKIP
  □ Step 2: Estimation (3-5 min) ← state the implications
  □ Step 3: High-level (10-15 min) ← draw, explain, walk through flows
  □ Step 4: Deep dive (15-20 min) ← trade-offs on every decision
  □ End: Summarize trade-offs + what you'd improve with more time
```

---

## The "What Would You Improve?" Question

```
MANY INTERVIEWS END WITH: "If you had more time, what would you add?"

HAVE 3 ANSWERS READY:

1. MONITORING & OBSERVABILITY:
   "I'd add comprehensive monitoring: request latency histograms,
    error rates, DB query performance, cache hit ratios. With
    alerting for anomalies."

2. SECURITY:
   "I'd add rate limiting, input validation, authentication via 
    OAuth 2.0, and encryption at rest + in transit. For a payment
    system, PCI DSS compliance."

3. COST OPTIMIZATION:
   "I'd implement tiered storage (hot/cold), auto-scaling, 
    and CDN optimization. Move old data to cheaper storage."
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the #1 interview mistake? | Jumping into design without gathering requirements. Spend 5 minutes on functional + non-functional requirements first. |
| 2 | How do you avoid over-engineering? | Start simple. Add complexity only when requirements demand it. If you can't explain WHY you need a technology, remove it. |
| 3 | What should you do for every major decision? | Discuss the trade-off: state your choice, state the alternative, explain why your choice fits the requirements better. |
| 4 | What is the thundering herd problem? | Cache expires → thousands of requests simultaneously hit the database. Fix: lock on cache miss, only one thread refills cache, others wait. |
| 5 | What should you always end with? | A summary of key trade-offs and 2-3 improvements you'd make with more time (monitoring, security, cost optimization). |
| 6 | What are the 3 numbers to memorize? | MySQL: ~10K QPS. Redis: ~100K ops/sec. App server: ~10K concurrent connections. These help you reason about scaling. |

---

[← Previous: Trade-off Analysis](./chapter-46-tradeoff-analysis.md)
