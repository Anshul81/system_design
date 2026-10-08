# Chapter 11: Load Balancing

> **Phase 3 — Load Balancing, Caching & Proxies** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium

[← Previous: Blob Storage](../phase-2-databases/chapter-10-blob-storage.md) · [Next: Caching →](./chapter-12-caching.md)

---

## Learning Objectives

- ✅ Explain L4 vs L7 load balancing and when to use each
- ✅ Compare load balancing algorithms (round-robin, least connections, consistent hashing)
- ✅ Design health checking and failover strategies
- ✅ Understand sticky sessions and when they're needed (and why to avoid them)
- ✅ Explain Global Server Load Balancing (GSLB) for multi-region architectures

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 0 (Networking) | ✅ Yes |
| Chapter 3 (Scaling) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Architecture exercises | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** A load balancer is like a host at a restaurant who decides which table (server) each guest (request) is seated at. They monitor which tables are full, which are available, and distribute guests evenly so no single waiter is overwhelmed.

**Interview relevance:** A load balancer appears in every system design diagram. You must know where to place it, which algorithm to use, and why. When the interviewer says "what happens if one of your servers goes down?", the load balancer is your first answer.

---

## Visual Diagram — Load Balancer Placement

```
TYPICAL ARCHITECTURE WITH LOAD BALANCERS:

                    ┌──────────────┐
   Internet ───────▶│  DNS / GSLB   │  Global load balancing
                    └──────┬───────┘  (route to nearest DC)
                           │
              ┌────────────┼────────────┐
              ▼                         ▼
     ┌─────────────┐           ┌─────────────┐
     │  LB (L7)     │           │  LB (L7)     │  Public LB
     │  Datacenter 1 │           │  Datacenter 2 │  (external)
     └──────┬──────┘           └──────┬──────┘
            │                         │
     ┌──────┼──────┐          ┌──────┼──────┐
     ▼      ▼      ▼          ▼      ▼      ▼
   ┌───┐  ┌───┐  ┌───┐     ┌───┐  ┌───┐  ┌───┐
   │WS1│  │WS2│  │WS3│     │WS1│  │WS2│  │WS3│  Web Servers
   └─┬─┘  └─┬─┘  └─┬─┘     └───┘  └───┘  └───┘
     │      │      │
     └──────┼──────┘
            │
     ┌──────┴──────┐
     │  LB (L4)     │  Internal LB (between tiers)
     └──────┬──────┘
     ┌──────┼──────┐
     ▼      ▼      ▼
   ┌───┐  ┌───┐  ┌───┐
   │AS1│  │AS2│  │AS3│    App Servers
   └───┘  └───┘  └───┘
```

---

## Core Concepts

### 1. L4 vs L7 Load Balancing

```
OSI MODEL (relevant layers):

Layer 7 (Application):  HTTP, HTTPS, WebSocket
Layer 4 (Transport):    TCP, UDP
Layer 3 (Network):      IP

L4 Load Balancer:
  • Operates at TCP/UDP level
  • Routes based on: IP address + port number
  • Cannot inspect HTTP content (URL, headers, cookies)
  • Faster — less processing per request
  • Example: AWS NLB, HAProxy (TCP mode)

L7 Load Balancer:
  • Operates at HTTP level
  • Routes based on: URL path, headers, cookies, request body
  • Can do: URL-based routing, SSL termination, header manipulation
  • More flexible — content-aware routing
  • Example: AWS ALB, Nginx, HAProxy (HTTP mode), Envoy
```

| Feature | L4 | L7 |
|---------|----|----|
| **Routes by** | IP + Port | URL, Headers, Cookies |
| **SSL termination** | ❌ (passthrough) | ✅ |
| **URL routing** | ❌ | ✅ `/api` → service A, `/web` → service B |
| **WebSocket support** | Basic | Full (connection upgrade aware) |
| **Performance** | Faster (less inspection) | Slightly slower |
| **Use case** | Internal services, TCP traffic | Public APIs, microservice routing |

**Interview tip:** "I'd use an L7 load balancer (like AWS ALB or Nginx) at the edge for HTTP routing and SSL termination, and L4 load balancers (like AWS NLB) for internal service-to-service communication where we don't need content inspection."

### 2. Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| **Round Robin** | Distribute requests 1-2-3-1-2-3... | Identical servers, equal request cost |
| **Weighted Round Robin** | Server A gets 3x requests, Server B gets 1x | Mixed hardware (powerful + weak servers) |
| **Least Connections** | Send to the server with fewest active connections | Varying request durations (long + short) |
| **Least Response Time** | Send to the server with fastest recent response | Latency-sensitive applications |
| **IP Hash** | hash(client_IP) % N → server N | Sticky sessions without cookies |
| **Consistent Hashing** | Hash ring, minimizes remapping when servers change | Cache servers, stateful connections |
| **Random** | Pick a random server | Simple, surprisingly effective at scale |

**Interview default:** "I'd use round-robin for stateless web servers and least connections for services with variable request processing times."

### 3. Health Checks

```
Load Balancer Health Check:

LB ──── GET /health ──────▶ Server 1 ──── 200 OK ────▶ ✅ HEALTHY
LB ──── GET /health ──────▶ Server 2 ──── 200 OK ────▶ ✅ HEALTHY
LB ──── GET /health ──────▶ Server 3 ──── timeout ───▶ ❌ UNHEALTHY

After 3 consecutive failures:
  → Server 3 removed from pool
  → Traffic redistributed to Server 1 and 2
  → Alert sent to ops team

After Server 3 recovers:
  → Health check passes again
  → Server 3 re-added to pool (gradually)
```

**Health check types:**
- **TCP check:** Can we connect on the port? (basic)
- **HTTP check:** Does `/health` return 200? (standard)
- **Deep check:** Does the server connect to DB and cache successfully? (thorough)

### 4. Sticky Sessions (Session Affinity)

Sticky sessions route all requests from the same client to the same server.

```
WITHOUT sticky sessions:
  Request 1 from User A → Server 1 (creates session)
  Request 2 from User A → Server 3 (session not found!)

WITH sticky sessions:
  Request 1 from User A → Server 1 (creates session)
  Request 2 from User A → Server 1 (session found ✅)
  Request 3 from User A → Server 1 (always same server)
```

**Why to AVOID sticky sessions:**
- Server failure = all sticky users lose their sessions
- Uneven load distribution (popular users stuck on one server)
- Breaks auto-scaling (new servers don't get sticky users)

**Better alternative:** Store sessions in Redis (shared state). Any server can handle any request.

### 5. Global Server Load Balancing (GSLB)

```
User in Tokyo                        User in London
     │                                      │
     ▼                                      ▼
┌─────────┐                          ┌─────────┐
│  DNS     │ GeoDNS resolves to      │  DNS     │
│  Query   │ nearest datacenter      │  Query   │
└────┬────┘                          └────┬────┘
     │                                      │
     ▼                                      ▼
┌──────────────┐                  ┌──────────────┐
│ Tokyo DC      │                  │ London DC     │
│ LB → Servers  │                  │ LB → Servers  │
└──────────────┘                  └──────────────┘

GSLB Methods:
  • GeoDNS: Return IP of nearest DC based on client location
  • Anycast: Same IP announced from multiple DCs, BGP routes to nearest
  • Latency-based: Route to DC with lowest measured latency
```

---

## Real-World Example — Netflix Load Balancing

```
Netflix uses a multi-layer load balancing approach:

1. DNS (Route 53): GeoDNS routes users to nearest AWS region
2. AWS ALB: L7 load balancing for HTTP routing
3. Zuul (API Gateway): Custom L7 routing, canary deployments
4. Eureka: Client-side load balancing for internal services
   (each service discovers and load-balances to other services)

Key insight: Netflix uses CLIENT-SIDE load balancing internally.
Instead of a central LB, each service has a list of available
instances and picks one (using round-robin or weighted random).
This eliminates the LB as a bottleneck and SPOF.
```

---

## Common Interview Questions

### Q1: "Where would you place load balancers in your architecture?"

**Model answer:**
"I'd place an L7 load balancer at the edge, between clients and web servers, for HTTP routing and SSL termination. If I have separate API and web-serving tiers, I'd place an internal L4 load balancer between them. For a microservices architecture, I might use client-side load balancing or a service mesh instead of centralized load balancers, to avoid bottlenecks. For multi-region, I'd use GeoDNS or Anycast to route users to the nearest datacenter."

### Q2: "What happens if the load balancer itself goes down?"

**Model answer:**
"The load balancer is deployed as a redundant pair — active and standby. They share a virtual IP using a protocol like VRRP. If the active LB fails, the standby detects it via heartbeat and takes over the virtual IP within seconds. For cloud deployments, managed load balancers like AWS ALB are already highly available across multiple availability zones."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| Using sticky sessions for stateful apps | Make apps stateless with Redis for sessions. Sticky sessions break scaling. |
| Single load balancer (SPOF) | Always deploy LB as redundant pair or use a managed service |
| Using L7 everywhere | Use L4 for internal TCP traffic (faster, less overhead). L7 for HTTP routing. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 11 CHEAT SHEET                        │
│                  Load Balancing                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  L4: TCP-level, fast, IP+port routing (NLB, HAProxy TCP)       │
│  L7: HTTP-level, content-aware, URL/header routing (ALB, Nginx)│
│                                                                  │
│  ALGORITHMS:                                                     │
│  Round Robin (default), Least Connections (variable load),      │
│  Consistent Hashing (cache/stateful), IP Hash (simple sticky)  │
│                                                                  │
│  HEALTH CHECKS: TCP, HTTP /health, deep (DB+cache check)       │
│  STICKY SESSIONS: Avoid — use shared session store (Redis)      │
│  GSLB: GeoDNS, Anycast for multi-region routing                │
│  REDUNDANCY: Active-passive pair or managed cloud LB            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the difference between L4 and L7 load balancing? | L4 routes by IP+port (TCP level). L7 routes by URL path, headers, cookies (HTTP level). |
| 2 | Which LB algorithm is best for servers with different processing speeds? | Least Connections — sends requests to the server with the fewest active connections |
| 3 | Why should you avoid sticky sessions? | Server failure loses all sticky sessions, creates uneven load, and breaks auto-scaling |
| 4 | What is GSLB? | Global Server Load Balancing — routes users to the nearest datacenter using GeoDNS, Anycast, or latency-based routing |
| 5 | What is an active-passive LB pair? | Two load balancers sharing a virtual IP. Active handles traffic; passive monitors and takes over if active fails. |
| 6 | When would you use consistent hashing for load balancing? | For cache servers or stateful connections where you want requests for the same key to consistently go to the same server |
| 7 | What should a health check endpoint verify? | At minimum: HTTP 200 from the server. Ideally: server can connect to its database, cache, and other dependencies. |
| 8 | What is client-side load balancing? | Each client/service maintains a list of available servers and selects one directly, without a central load balancer. Used by Netflix (Eureka). |

---

[← Previous: Blob Storage](../phase-2-databases/chapter-10-blob-storage.md) · [Next: Caching →](./chapter-12-caching.md)
