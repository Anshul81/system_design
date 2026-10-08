# Chapter 16: Microservices vs Monolith

> **Phase 4 — Infrastructure & Middleware** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium

[← Previous: Message Queues](./chapter-15-message-queues.md) · [Next: Search Systems →](./chapter-17-search-systems.md)

---

## Learning Objectives

- ✅ Compare monolithic and microservices architectures with clear trade-offs
- ✅ Know when each architecture is appropriate and defend the choice
- ✅ Understand service boundaries and decomposition strategies
- ✅ Discuss challenges of microservices: data consistency, distributed transactions, service discovery

---

## Introduction — Why This Matters

**Interview relevance:** Most system design answers should be microservices at scale — but you need to articulate WHY and know the trade-offs. "Microservices because it's modern" is a weak answer. "Microservices because the user service team can deploy independently from the payment team, and we can scale the search service differently from the auth service" is strong.

---

## Core Concepts

### 1. Monolith vs Microservices

```
MONOLITH:
┌──────────────────────────────────────────┐
│              Single Application           │
│                                          │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌─────┐ │
│  │ Auth  │  │ User │  │ Order│  │ Pay │ │
│  │Module │  │Module│  │Module│  │ment │ │
│  └──────┘  └──────┘  └──────┘  └─────┘ │
│                                          │
│  Single codebase, single deployment      │
│  Single database                         │
└──────────────────────────────────────────┘

MICROSERVICES:
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Auth     │  │ User    │  │ Order   │  │ Payment │
│ Service  │  │ Service │  │ Service │  │ Service │
│          │  │         │  │         │  │         │
│ Own DB   │  │ Own DB  │  │ Own DB  │  │ Own DB  │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
  Each service: own codebase, own deployment, own database
  Communicate via APIs (REST/gRPC) or message queues
```

### 2. Comparison

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Deploy everything together | Deploy services independently |
| **Scaling** | Scale the whole app | Scale individual services |
| **Development** | Simple, one codebase | Complex, many codebases |
| **Team structure** | One team (or tightly coupled) | Independent teams per service |
| **Data consistency** | Easy (single DB, transactions) | Hard (distributed transactions) |
| **Debugging** | Easy (single process) | Hard (distributed tracing needed) |
| **Latency** | Function calls (ns) | Network calls (ms) |
| **Operational cost** | Low | High (many services to manage) |
| **Best for** | Early stage, small teams, simple products | Large orgs, complex products, independent scaling needs |

### 3. When to Choose Each

```
CHOOSE MONOLITH WHEN:
✅ Small team (< 10 engineers)
✅ Early startup (move fast, iterate quickly)
✅ Simple domain (CRUD app, MVP)
✅ Tight consistency requirements (single DB transactions)
✅ Don't want operational overhead of many services

CHOOSE MICROSERVICES WHEN:
✅ Large team (50+ engineers, multiple teams)
✅ Different services need different scaling (search vs auth)
✅ Independent deployments needed (daily releases per team)
✅ Different tech stacks per service (Python ML, Go APIs)
✅ Clear domain boundaries exist

THE PRAGMATIC PATH:
Start monolith → Identify bottleneck/team boundaries →
Extract services gradually (the "strangler fig" pattern)
```

### 4. Microservices Challenges

| Challenge | Problem | Solution |
|-----------|---------|----------|
| **Data consistency** | No cross-service transactions | Saga pattern, eventual consistency |
| **Service discovery** | How do services find each other? | Service registry (Consul, Eureka), DNS, Kubernetes |
| **Distributed tracing** | Following a request across services | Jaeger, Zipkin, OpenTelemetry |
| **API versioning** | Changing one service's API affects others | Contract testing, backward compatibility |
| **Network failures** | Services can't reach each other | Circuit breakers, retries, timeouts |
| **Data duplication** | Each service owns its data | Accept duplication, event-driven sync |

### 5. The Saga Pattern (Distributed Transactions)

```
Order Saga (choreography-based):

1. Order Service: Create order (PENDING)
   → publish: OrderCreated

2. Payment Service: Charge customer
   → success: publish PaymentCompleted
   → failure: publish PaymentFailed

3. Inventory Service: Reserve items
   → success: publish InventoryReserved
   → failure: publish InventoryFailed → compensate payment

4. Shipping Service: Schedule delivery
   → success: publish ShipmentScheduled
   → failure: compensate inventory + payment

5. Order Service: Mark order CONFIRMED

If any step fails → compensating transactions undo previous steps
```

---

## Common Interview Questions

### Q1: "Would you design this as a monolith or microservices?"

**Model answer:**
"For a system design interview at scale — let's say we're designing for Twitter's 250M DAU — I'd use microservices. The tweet service, timeline service, user service, and search service all have different scaling needs. The tweet service is write-heavy, the timeline service is read-heavy, and the search service needs its own specialized infrastructure (Elasticsearch). Independent teams can own each service and deploy independently. I'd use gRPC for synchronous inter-service communication and Kafka for async event processing."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 16 CHEAT SHEET                        │
│                  Microservices vs Monolith                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MONOLITH: Simple, fast development, single DB transactions     │
│  MICROSERVICES: Independent scaling/deployment, complex ops     │
│                                                                  │
│  PATH: Monolith → Extract services as needed                    │
│  INTERVIEW: Use microservices for scale, explain WHY            │
│                                                                  │
│  CHALLENGES: Data consistency (Saga), service discovery,        │
│  distributed tracing, network failures, API versioning          │
│                                                                  │
│  SAGA PATTERN: Chain of local transactions + compensating       │
│  actions for rollback. Replaces distributed transactions.       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is the main advantage of microservices over monoliths? | Independent deployment and scaling of services. Different teams can work on different services without blocking each other. |
| 2 | When should you start with a monolith? | Small team, early startup, simple domain, need for strong consistency, want to move fast without operational overhead. |
| 3 | What is the Saga pattern? | A sequence of local transactions across services, each with a compensating action for rollback if a later step fails. Replaces distributed transactions. |
| 4 | What is the strangler fig pattern? | Gradually replacing a monolith by extracting services one at a time, routing traffic to new services while the monolith shrinks. |
| 5 | How do microservices communicate? | Synchronous: REST/gRPC for request-response. Asynchronous: message queues (Kafka, SQS) for event-driven communication. |
| 6 | What is service discovery? | The mechanism by which services find each other's network addresses. Implemented via service registry (Consul, Eureka) or DNS. |

---

[← Previous: Message Queues](./chapter-15-message-queues.md) · [Next: Search Systems →](./chapter-17-search-systems.md)
