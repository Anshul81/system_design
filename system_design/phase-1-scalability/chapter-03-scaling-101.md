# Chapter 3: Scaling 101 — Vertical vs Horizontal

> **Phase 1 — Scalability Fundamentals** · Estimated Time: 2.5 hours · Difficulty: 🟢 Beginner

[← Previous: Communication Patterns](../phase-0-networking/chapter-02-communication-patterns.md) · [Next: Back-of-the-Envelope Estimation →](./chapter-04-estimation.md)

---

## Learning Objectives

After this chapter, you will be able to:

- ✅ Explain vertical vs horizontal scaling and choose the right approach
- ✅ Design stateless services that scale horizontally
- ✅ Identify and eliminate single points of failure (SPOF)
- ✅ Understand shared-nothing architecture and why it matters
- ✅ Discuss auto-scaling strategies and their trade-offs
- ✅ Draw a scalable architecture on a whiteboard

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Phase 0 (Networking & Communication) | ✅ Yes |
| Basic understanding of servers | Helpful |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Architecture exercises | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** Imagine you run a restaurant.
- **Vertical scaling (scale up):** You hire a super-chef who can cook 10x faster. Eventually, even the best chef has limits.
- **Horizontal scaling (scale out):** You open more kitchens with more chefs. Each kitchen handles its share of orders.

**Interview relevance:** "How would you scale this?" is asked in every single system design interview. The answer is almost always "horizontal scaling" — but you need to explain WHY, what the trade-offs are, and what changes you need to make (stateless services, shared databases, load balancers). Candidates who can't articulate the scaling path get rejected.

---

## Visual Diagram — Scaling Approaches

```
VERTICAL SCALING (Scale Up):

Before:                          After:
┌─────────────┐                 ┌─────────────────────┐
│   Server     │                 │      Server          │
│   4 CPU      │                 │      32 CPU          │
│   16 GB RAM  │    ═══════▶    │      256 GB RAM      │
│   500 GB SSD │                 │      4 TB SSD        │
│              │                 │                      │
│   Handles    │                 │      Handles         │
│   1K QPS     │                 │      8K QPS          │
└─────────────┘                 └─────────────────────┘

Problem: There's a ceiling. You can't buy a 10,000-CPU server.
         Also: single point of failure. If it dies, everything dies.


HORIZONTAL SCALING (Scale Out):

Before:                          After:
┌─────────────┐                       ┌──────────────┐
│   Server     │                       │ Load Balancer │
│   4 CPU      │                       └──────┬───────┘
│   16 GB RAM  │    ═══════▶           ┌──────┼───────┐
│   1K QPS     │                 ┌─────┴─┐ ┌──┴────┐ ┌┴──────┐
└─────────────┘                 │Server 1│ │Server 2│ │Server 3│
                                 │ 4 CPU  │ │ 4 CPU  │ │ 4 CPU  │
                                 │ 1K QPS │ │ 1K QPS │ │ 1K QPS │
                                 └────────┘ └────────┘ └────────┘
                                         Total: 3K QPS

Advantage: Add more servers → more capacity. No ceiling.
Challenge: Need load balancer, stateless design, shared storage.
```

---

## Core Concepts

### 1. Vertical Scaling (Scale Up)

Add more power to a single machine: more CPU, RAM, faster disks, better network.

| Pros | Cons |
|------|------|
| Simple — no code changes needed | Hardware ceiling (biggest AWS instance: 448 vCPUs, 24 TB RAM) |
| No distributed system complexity | Single point of failure |
| Lower latency (no network hops) | Expensive (cost grows super-linearly) |
| Easier to manage | Downtime for upgrades |
| ACID transactions are simpler | Can't scale infinitely |

**When vertical scaling makes sense:**
- Early stage startups (simple, fast, cheap enough)
- Database servers where horizontal scaling (sharding) is complex
- In-memory caches where you need more RAM
- Low-traffic applications

**The cost curve problem:**
```
Cost vs. Capacity (Vertical):

Cost ($)
  │
  │                              ╱
  │                            ╱
  │                          ╱
  │                       ╱
  │                    ╱     ← Cost grows SUPER-LINEARLY
  │                 ╱
  │             ╱
  │          ╱
  │      ╱
  │   ╱
  │╱
  └────────────────────────── Capacity

A server with 2x the power costs 3-4x more.
A server with 10x the power costs 20-50x more.
```

---

### 2. Horizontal Scaling (Scale Out)

Add more machines to distribute the load. This is the default strategy for most system designs.

| Pros | Cons |
|------|------|
| No ceiling — add as many servers as needed | More complex architecture |
| Built-in redundancy (one server dies, others continue) | Need load balancer |
| Cost-effective (commodity hardware) | Data consistency challenges |
| Can scale to millions of QPS | Stateless design required |
| Zero-downtime rolling deployments | More operational overhead |

**What you need for horizontal scaling:**
```
                  ┌─────────────────┐
                  │  Load Balancer   │  ← Distributes traffic
                  └────────┬────────┘
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ App       │ │ App       │ │ App       │  ← Stateless
        │ Server 1  │ │ Server 2  │ │ Server 3  │     (no local state)
        └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
              │             │             │
              └──────────┬──┴─────────────┘
                         ▼
                  ┌─────────────────┐
                  │  Shared State    │  ← Database, cache, etc.
                  │  (Database,      │     (centralized state)
                  │   Redis, etc.)   │
                  └─────────────────┘
```

---

### 3. Stateless vs Stateful Services

This is the **single most important concept** for horizontal scaling. If a service stores state locally, you can't just add more instances.

#### Stateful Service (Hard to Scale)

```
User A's session is on Server 1:

┌──────────┐     ┌──────────────┐
│  User A   │────▶│  Server 1     │  session: {userId: "A", cart: [...]}
└──────────┘     └──────────────┘

If Server 1 dies or User A gets routed to Server 2:
  → Session is LOST. User has to log in again, cart is empty.

If you add Server 3:
  → User A's session is still only on Server 1.
  → Load balancer must use "sticky sessions" (fragile).
```

#### Stateless Service (Easy to Scale) ✅

```
Session is stored externally (Redis, database):

┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  User A   │────▶│  ANY Server   │────▶│  Redis       │
└──────────┘     │  (no local    │     │  session:    │
                  │   state)      │     │  {userId:"A",│
                  └──────────────┘     │   cart:[...]}│
                                        └─────────────┘

Any request from User A can go to ANY server.
The server reads session from Redis, processes the request, and responds.

If Server 1 dies → User A goes to Server 2 → same session from Redis.
If you add Server 4 → it can handle User A's requests immediately.
```

**The stateless rule:**
> *Application servers should store ZERO state locally. All state goes to a shared store (database, cache, object storage). The server is just a compute node that reads state, processes logic, and writes state back.*

**What moves to shared storage:**
| Local State (Bad) | Shared State (Good) |
|-------------------|---------------------|
| In-memory sessions | Redis/Memcached |
| Local file uploads | S3 / Object storage |
| Local caches | Redis / CDN |
| In-memory job queues | Kafka / SQS / RabbitMQ |
| Local logs | ELK stack / CloudWatch |

---

### 4. Single Point of Failure (SPOF)

A SPOF is any component whose failure takes down the entire system. Eliminating SPOFs is fundamental to building reliable, scalable systems.

```
BEFORE (Multiple SPOFs):

┌────────┐     ┌────────────┐     ┌────────────┐
│ Client  │────▶│ Web Server  │────▶│ Database    │
└────────┘     │ (SPOF!)     │     │ (SPOF!)     │
                └────────────┘     └────────────┘

If web server dies → entire system is down.
If database dies → entire system is down.


AFTER (No SPOFs):

┌────────┐     ┌──────────────┐     ┌──────────────────────┐
│ Client  │────▶│ Load Balancer │     │    Web Servers        │
└────────┘     │ (redundant    │────▶│ ┌─────┐ ┌─────┐     │
                │  pair: active │     │ │ WS1 │ │ WS2 │ ... │
                │  + standby)   │     │ └─────┘ └─────┘     │
                └──────────────┘     └──────────┬───────────┘
                                                │
                                     ┌──────────▼───────────┐
                                     │   Database Cluster    │
                                     │ ┌────────┐ ┌───────┐ │
                                     │ │Primary  │ │Replica │ │
                                     │ │(writes) │ │(reads) │ │
                                     │ └────────┘ └───────┘ │
                                     └──────────────────────┘

Every component has redundancy:
• Load balancer: active-passive pair
• Web servers: multiple instances behind LB
• Database: primary with one or more replicas
```

**Common SPOFs to address in interviews:**

| Component | SPOF Solution |
|-----------|---------------|
| Web server | Multiple instances + load balancer |
| Database | Replication (primary + replicas) |
| Load balancer | Active-passive pair, or DNS failover |
| Cache (Redis) | Redis Sentinel or Redis Cluster |
| DNS | Multiple DNS providers |
| Datacenter | Multi-region / multi-AZ deployment |

---

### 5. Shared-Nothing Architecture

In a shared-nothing architecture, each node is independent and self-sufficient. No node shares memory, disk, or state with another. This is the gold standard for horizontal scaling.

```
SHARED-EVERYTHING (traditional):
┌──────┐  ┌──────┐  ┌──────┐
│Node 1│  │Node 2│  │Node 3│
└──┬───┘  └──┬───┘  └──┬───┘
   │         │         │
   └─────────┼─────────┘
             │
      ┌──────▼──────┐
      │ Shared Disk  │  ← Bottleneck & SPOF
      │ (SAN/NAS)    │
      └─────────────┘

SHARED-NOTHING (scalable):
┌───────────┐  ┌───────────┐  ┌───────────┐
│  Node 1    │  │  Node 2    │  │  Node 3    │
│  CPU       │  │  CPU       │  │  CPU       │
│  Memory    │  │  Memory    │  │  Memory    │
│  Disk      │  │  Disk      │  │  Disk      │
│  Data: A-F │  │  Data: G-M │  │  Data: N-Z │
└───────────┘  └───────────┘  └───────────┘

Each node owns its data. No shared bottleneck.
Scale by adding more nodes.
```

**Interview tip:** When explaining database sharding, mention shared-nothing architecture. "Each shard is a shared-nothing node — it owns its data, has its own CPU and memory, and can be scaled independently."

---

### 6. Auto-Scaling

Auto-scaling automatically adjusts the number of server instances based on demand.

```
Auto-Scaling Behavior:

Load (QPS)
  │
  │         ╱╲         Peak traffic
  │        ╱  ╲        (scale up)
  │       ╱    ╲
  │      ╱      ╲
  │     ╱        ╲
  │    ╱          ╲     Off-peak
  │   ╱            ╲    (scale down)
  │  ╱              ╲
  │ ╱                ╲
  └────────────────────── Time

Servers
  │
  │     ┌────┐          8 servers at peak
  │     │    │
  │  ┌──┘    └──┐       5 servers normal
  │  │          │
  │──┘          └──     3 servers minimum
  └────────────────────── Time
```

**Auto-scaling strategies:**

| Strategy | Trigger | Use Case |
|----------|---------|----------|
| **CPU-based** | Scale when CPU > 70% | General web servers |
| **Memory-based** | Scale when RAM > 80% | In-memory processing |
| **QPS-based** | Scale when QPS > threshold | API servers |
| **Queue-based** | Scale when queue depth > N | Workers processing background jobs |
| **Schedule-based** | Scale at specific times | Known traffic patterns (morning rush, sales events) |

**Key auto-scaling concepts:**
- **Minimum instances:** Always keep N servers running (don't scale to zero)
- **Cooldown period:** Wait N seconds after scaling before scaling again (prevent thrashing)
- **Health checks:** Don't send traffic to unhealthy instances
- **Graceful shutdown:** Drain connections before terminating an instance

---

## Comparison Table — Vertical vs Horizontal Scaling

| Factor | Vertical (Scale Up) | Horizontal (Scale Out) |
|--------|--------------------|-----------------------|
| **How** | Bigger machine | More machines |
| **Ceiling** | Hardware limits | No practical limit |
| **Complexity** | Low (no code changes) | High (distributed systems) |
| **Cost** | Super-linear (expensive) | Linear (cost-effective) |
| **Availability** | SPOF (single machine) | Redundant (multiple machines) |
| **Downtime** | Needed for upgrades | Zero-downtime rolling updates |
| **Data consistency** | Easy (single node) | Hard (distributed state) |
| **When to use** | Early stage, databases, caches | Web servers, application layer |

---

## Real-World Example — How Instagram Scaled

```
Instagram's Scaling Journey (2010 → 2012):

2010 (Launch):
  • 1 server (EC2 instance)
  • Single PostgreSQL database
  • Handles 25K signups on day 1
  • Classic vertical scaling: bigger instance

2011 (Growth to 10M users):
  • Multiple app servers + load balancer (horizontal)
  • PostgreSQL read replicas (vertical + horizontal)
  • Redis for caching (reduce DB load)
  • S3 for photo storage (offload to cloud)
  • CDN for serving photos (reduce server load)

2012 (30M users, acquired by Facebook):
  • Sharded PostgreSQL (horizontal DB scaling)
  • Multiple Redis clusters
  • Celery + RabbitMQ for async tasks (background processing)
  • Multiple datacenters

Key insight:
  Instagram started with vertical scaling (bigger servers).
  When they hit limits, they went horizontal piece by piece:
  1. First: stateless web servers behind a load balancer
  2. Second: read replicas for the database
  3. Third: caching layer (Redis)
  4. Fourth: sharding the database
  5. Fifth: async processing via message queues

This is the typical scaling evolution. Don't design for
a billion users on day one — scale incrementally.
```

---

## Common Interview Questions

### Q1: "How would you scale a web application from 100 to 10 million users?"

**Model answer:**
"I'd scale incrementally. At 100 users, a single server with a database is fine. At 10K users, I'd add a load balancer with 2-3 stateless app servers and make sure sessions are stored in Redis. At 100K, I'd add database read replicas and a caching layer (Redis) to reduce DB load. At 1M, I'd introduce a CDN for static assets, add more app servers, and start thinking about database sharding. At 10M, I'd have sharded databases, multiple cache clusters, message queues for async processing, and potentially a multi-region setup. The key principle is: keep app servers stateless, scale them horizontally, and defer database scaling as long as possible because it's the hardest part."

### Q2: "What's the difference between stateful and stateless services?"

**Model answer:**
"A stateless service stores no data locally between requests. Any instance can handle any request because all shared state — sessions, carts, user data — lives in external stores like Redis or a database. This makes horizontal scaling trivial: just add more instances behind a load balancer. A stateful service stores data locally — like in-memory sessions or local file caches. This creates problems: you need sticky sessions, you can't freely route requests, and if a server dies, the local state is lost. The rule in system design: always make your application tier stateless."

### Q3: "What is a single point of failure and how do you eliminate it?"

**Model answer:**
"A single point of failure is any component whose failure takes down the entire system. In a basic setup, your web server and database are both SPOFs. To eliminate them: run multiple app server instances behind a load balancer, set up database replication with a primary and one or more replicas that can be promoted if the primary fails, use a redundant load balancer pair (active-passive), and deploy across multiple availability zones. The goal is that no single hardware failure can take down your service."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "We'll horizontally scale the database from day one" | Start with a single database. Vertical scaling and read replicas first. Sharding is complex — only shard when you must. |
| Designing for 1 billion users immediately | Scale incrementally. Start simple, then scale the bottleneck. The architecture at 10K users should differ from 10M users. |
| "Stateless means no state at all" | Stateless means no LOCAL state. The app is still working with state — it's just stored externally in Redis, a database, or S3. |
| Ignoring SPOFs in the design | Every component in your architecture should have redundancy. If you draw a single database box, the interviewer will ask "what if it dies?" |
| Not mentioning auto-scaling | Modern systems auto-scale. Mention "we'd configure auto-scaling based on CPU utilization or QPS" to show awareness. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 3 CHEAT SHEET                         │
│                  Scaling 101                                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  VERTICAL (Scale Up): Bigger machine. Simple but limited.       │
│  HORIZONTAL (Scale Out): More machines. Complex but unlimited.  │
│                                                                  │
│  STATELESS SERVICE (Required for horizontal scaling):           │
│  • No local sessions, files, or caches                          │
│  • All state in Redis / DB / S3                                 │
│  • Any instance handles any request                             │
│                                                                  │
│  SPOF ELIMINATION:                                               │
│  • Web servers → multiple instances + load balancer             │
│  • Database → primary + replica(s)                               │
│  • Load balancer → active-passive pair                          │
│  • Datacenter → multi-AZ / multi-region                         │
│                                                                  │
│  SCALING EVOLUTION:                                              │
│  Single server → LB + app servers → Read replicas →             │
│  Caching → CDN → Sharding → Multi-region                       │
│                                                                  │
│  AUTO-SCALING TRIGGERS:                                          │
│  CPU > 70%, Memory > 80%, QPS > threshold, Queue depth         │
│                                                                  │
│  GOLDEN RULE: Scale the application tier first (easy,           │
│  stateless). Scale the database tier last (hard, stateful).     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the main limitation of vertical scaling? | Hardware ceiling — you can't endlessly add CPU/RAM. Also creates a single point of failure. |
| 2 | What does "stateless" mean for a web server? | The server stores no data locally between requests. All state is in external stores (Redis, DB, S3). Any server can handle any request. |
| 3 | Why are stateless services easier to scale? | You can add/remove instances freely behind a load balancer. No sticky sessions needed, no local data to migrate. |
| 4 | What are the 4 main things needed for horizontal scaling? | Load balancer, stateless application servers, shared state store (Redis/DB), and shared file storage (S3). |
| 5 | What is shared-nothing architecture? | Each node is independent — has its own CPU, memory, and disk. No shared resources between nodes. Enables true horizontal scaling. |
| 6 | What is a single point of failure (SPOF)? | Any component whose failure takes down the entire system. Eliminated through redundancy (replicas, multiple instances, failover). |
| 7 | What's the typical scaling evolution for a web app? | Single server → LB + multiple app servers → DB read replicas → Caching (Redis) → CDN → DB sharding → Multi-region |
| 8 | When should you vertically scale vs horizontally scale? | Vertically scale first (simpler, no code changes). Switch to horizontal when you hit the ceiling or need high availability. Application servers: horizontal from the start. Databases: vertical first, then replicas, then sharding. |
| 9 | What is a cooldown period in auto-scaling? | A waiting period after a scaling event before another can occur. Prevents rapid oscillation (thrashing) between scale up and scale down. |
| 10 | Why is database scaling harder than app server scaling? | Databases are stateful — they hold data. Scaling horizontally (sharding) means splitting data across nodes, handling cross-shard queries, maintaining consistency, and dealing with rebalancing. App servers are stateless — just add more. |

---

[← Previous: Communication Patterns](../phase-0-networking/chapter-02-communication-patterns.md) · [Next: Back-of-the-Envelope Estimation →](./chapter-04-estimation.md)
