# Chapter 24: Availability Patterns & SLAs

> **Phase 5 — Distributed Systems Concepts** · Estimated Time: 3 hours · Difficulty: 🟡 Medium

[← Previous: Distributed Transactions](./chapter-23-distributed-transactions.md) · [Next: URL Shortener →](../phase-6-classic-designs/chapter-25-url-shortener.md)

---

## Learning Objectives

- ✅ Calculate SLA uptime and understand the "nines" of availability
- ✅ Compare active-passive and active-active failover strategies
- ✅ Design for fault tolerance with redundancy, retries, circuit breakers
- ✅ Understand graceful degradation and feature flagging
- ✅ Discuss multi-region deployment strategies

---

## Introduction — Why This Matters

**Interview relevance:** When you say "99.99% availability," the interviewer expects you to know what that means (52 minutes of downtime per year) and how to achieve it (redundancy, failover, multi-AZ, health checks). Availability is a first-class concern in every system design.

---

## Core Concepts

### 1. The Nines of Availability

| Availability | Downtime/Year | Downtime/Month | Downtime/Day |
|-------------|--------------|----------------|-------------|
| 99% (two nines) | 3.65 days | 7.3 hours | 14.4 minutes |
| 99.9% (three nines) | 8.77 hours | 43.8 minutes | 1.44 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.38 minutes | 8.6 seconds |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds | 0.86 seconds |

```
SLA of composed systems (serial dependencies):

System availability = A1 × A2 × A3

Web Server (99.99%) × Database (99.99%) × Cache (99.99%)
= 0.9999 × 0.9999 × 0.9999 = 99.97%

Each dependency REDUCES overall availability.
This is why you eliminate SPOFs — each one is a multiplier.

With redundancy (parallel components):
Availability = 1 - (1 - A)^n

Two load balancers (each 99.9%):
= 1 - (1 - 0.999)^2 = 1 - 0.000001 = 99.9999%
```

### 2. Failover Strategies

```
ACTIVE-PASSIVE (Hot Standby):
┌──────────┐     ┌──────────┐
│ Active    │     │ Passive   │
│ (handles  │     │ (idle,    │
│  traffic) │     │  standby) │
└─────┬────┘     └─────┬────┘
      │ replication     │
      └────────────────▶│
      
If Active fails → Passive promoted to Active
Downtime: seconds to minutes (detection + promotion)


ACTIVE-ACTIVE (Both handle traffic):
┌──────────┐     ┌──────────┐
│ Active 1  │     │ Active 2  │
│ (handles  │     │ (handles  │
│  traffic) │     │  traffic) │
└──────────┘     └──────────┘

Both handle traffic. If one fails → other handles all.
Better utilization but more complex (data sync, conflicts).
```

| Strategy | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Active-Passive** | Simple, no conflict | Wasted standby resource, brief downtime | Database failover |
| **Active-Active** | No wasted resources, instant failover | Conflict resolution needed, complex | Multi-region web servers |

### 3. Fault Tolerance Patterns

| Pattern | What It Does | Example |
|---------|-------------|---------|
| **Retries with backoff** | Retry failed requests with increasing delay | 1s → 2s → 4s → 8s (exponential backoff) |
| **Circuit breaker** | Stop calling a failing service, fail fast | If >50% of calls fail → open circuit for 30s |
| **Bulkhead** | Isolate failures so they don't cascade | Separate thread pools per downstream service |
| **Timeout** | Don't wait forever for a response | Set 3-second timeout for all downstream calls |
| **Graceful degradation** | Serve reduced functionality when a component fails | Netflix: if recommendations fail, show popular titles |
| **Feature flags** | Toggle features on/off without deployment | Disable new feature if it causes issues |

```
CIRCUIT BREAKER states:

CLOSED ────(failures > threshold)────▶ OPEN
(normal,                                (reject all calls,
 calls pass                             fail fast,
 through)                               don't call service)
                                          │
  ▲                                       │ (timeout expires)
  │                                       ▼
  └────(success)──── HALF-OPEN ◀──────────
                     (allow one test call,
                      check if service recovered)
```

### 4. Multi-Region Deployment

```
┌─────────────────────┐          ┌─────────────────────┐
│    US-EAST Region    │          │   EU-WEST Region     │
│                      │          │                      │
│  ┌────┐ ┌────┐      │          │  ┌────┐ ┌────┐      │
│  │App │ │App │      │          │  │App │ │App │      │
│  │ 1  │ │ 2  │      │          │  │ 1  │ │ 2  │      │
│  └──┬─┘ └──┬─┘      │          │  └──┬─┘ └──┬─┘      │
│     │      │        │          │     │      │        │
│  ┌──▼──────▼──┐     │ Cross-   │  ┌──▼──────▼──┐     │
│  │ Primary DB  │◀────┼─region──▶│  │ Replica DB  │     │
│  │ (writes)    │     │ repl.   │  │ (reads +    │     │
│  └────────────┘     │          │  │  local write)│     │
│                      │          │  └────────────┘     │
└─────────────────────┘          └─────────────────────┘

Strategies:
  1. Active-Passive: One region handles writes. Other is standby.
     Simple but standby region is wasted + higher write latency for remote users.
  
  2. Active-Active: Both regions handle reads AND writes.
     Low latency everywhere, but need conflict resolution (multi-leader replication).
  
  3. Read-Local, Write-Remote: Reads from local region. Writes routed to primary.
     Good balance — reads are fast, writes have cross-region latency.
```

---

## Common Interview Questions

### Q1: "How would you achieve 99.99% availability?"

**Model answer:**
"99.99% allows 52 minutes of downtime per year. To achieve this: first, eliminate all single points of failure — redundant load balancers, multiple app server instances, database primary with hot standby replicas, Redis cluster with failover. Second, deploy across multiple availability zones so a single AZ failure doesn't take us down. Third, implement health checks with automatic failover — unhealthy instances are removed from the load balancer within seconds. Fourth, use circuit breakers so a failing downstream service doesn't cascade. Fifth, have automated rollback for bad deployments. For five nines (99.999%), I'd add multi-region active-active deployment."

### Q2: "What is graceful degradation?"

**Model answer:**
"Graceful degradation means continuing to serve users with reduced functionality when a component fails, rather than going completely down. For example, if our recommendation engine fails, we show trending or popular items instead of personalized recommendations. If the search service is degraded, we might serve cached results from 5 minutes ago. The key is identifying which features are critical (must work) vs nice-to-have (can degrade). We'd implement this with circuit breakers and fallback logic."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 24 CHEAT SHEET                        │
│                  Availability Patterns & SLAs                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  NINES: 99.9% = 8.77 hrs/yr, 99.99% = 52 min/yr              │
│  SERIAL: A_total = A1 × A2 × A3 (each dependency reduces)     │
│  PARALLEL: A = 1 - (1-A)^n (redundancy increases)             │
│                                                                  │
│  FAILOVER: Active-passive (simple) vs active-active (no waste) │
│  MULTI-AZ: Always. MULTI-REGION: For 99.99%+                  │
│                                                                  │
│  PATTERNS: Retries (exponential backoff), circuit breaker,     │
│  timeouts, bulkhead, graceful degradation, feature flags       │
│                                                                  │
│  INTERVIEW: "Redundancy at every layer, multi-AZ, health       │
│  checks, circuit breakers, automated failover."                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What does 99.99% availability mean in downtime? | 52.6 minutes per year, 4.38 minutes per month. |
| 2 | How do you calculate availability for serial dependencies? | Multiply: A_total = A1 × A2 × A3. Each component reduces total availability. |
| 3 | Active-passive vs active-active failover? | Active-passive: standby takes over on failure (simple, some downtime). Active-active: both handle traffic (no downtime, but need conflict resolution). |
| 4 | What is a circuit breaker? | A pattern that stops calling a failing service after a threshold of failures, failing fast instead. After a timeout, allows test calls to check recovery. |
| 5 | What is graceful degradation? | Serving reduced functionality when a component fails instead of going fully down. Example: showing popular items when recommendation engine is down. |
| 6 | How does multi-AZ deployment help availability? | If one availability zone fails (hardware, power, network), traffic is served from other AZs with no downtime. |

---

[← Previous: Distributed Transactions](./chapter-23-distributed-transactions.md) · [Next: URL Shortener →](../phase-6-classic-designs/chapter-25-url-shortener.md)
