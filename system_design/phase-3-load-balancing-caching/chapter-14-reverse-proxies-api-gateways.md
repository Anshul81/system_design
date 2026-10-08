# Chapter 14: Reverse Proxies & API Gateways

> **Phase 3 — Load Balancing, Caching & Proxies** · Estimated Time: 2 hours · Difficulty: 🟡 Medium

[← Previous: CDN & Edge](./chapter-13-cdn-edge.md) · [Next: Message Queues →](../phase-4-infrastructure/chapter-15-message-queues.md)

---

## Learning Objectives

- ✅ Distinguish forward proxy from reverse proxy
- ✅ Explain API gateway responsibilities and why they matter for microservices
- ✅ Understand service mesh basics (Envoy, Istio concept)
- ✅ Know when to use a reverse proxy vs a load balancer vs an API gateway

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 11 (Load Balancing) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Interview relevance:** In microservices architectures, an API gateway sits between clients and your services. It handles cross-cutting concerns: authentication, rate limiting, routing, request transformation. If you don't mention it in your design, the interviewer will ask "where does auth happen?" and "how do you rate-limit?"

---

## Core Concepts

### 1. Forward Proxy vs Reverse Proxy

```
FORWARD PROXY (client-side):
┌────────┐     ┌──────────┐     ┌──────────┐
│ Client  │────▶│ Forward   │────▶│ Internet  │
│         │     │ Proxy     │     │ Server    │
└────────┘     └──────────┘     └──────────┘
  Client knows about proxy. Server doesn't know about client.
  Use: VPN, content filtering, bypass restrictions.

REVERSE PROXY (server-side):
┌────────┐     ┌──────────┐     ┌──────────┐
│ Client  │────▶│ Reverse   │────▶│ Backend   │
│         │     │ Proxy     │     │ Server    │
└────────┘     └──────────┘     └──────────┘
  Client doesn't know about backend. Server is hidden.
  Use: Load balancing, SSL termination, caching, security.
```

### 2. API Gateway

An API gateway is a reverse proxy with additional features for microservices:

```
┌────────┐          ┌─────────────────────────────────┐
│ Mobile  │─────┐   │         API GATEWAY              │
│ App     │     │   │                                  │
└────────┘     │   │  1. Authentication (JWT verify)  │
               │   │  2. Rate Limiting                 │
┌────────┐     ├──▶│  3. Request Routing               │───▶ User Service
│ Web App │─────┤   │  4. Request/Response Transform   │───▶ Order Service
└────────┘     │   │  5. Response Aggregation          │───▶ Product Service
               │   │  6. Caching                       │───▶ Payment Service
┌────────┐     │   │  7. Logging & Monitoring          │
│ 3rd     │─────┘   │  8. Circuit Breaking             │
│ Party   │         │  9. CORS, Compression            │
└────────┘          └─────────────────────────────────┘
```

**API Gateway responsibilities:**

| Responsibility | What It Does |
|---------------|-------------|
| **Authentication** | Verify JWT/API key before requests reach services |
| **Rate Limiting** | Enforce per-client request limits (429 Too Many Requests) |
| **Routing** | `/api/users/*` → User Service, `/api/orders/*` → Order Service |
| **Response Aggregation** | Combine responses from multiple services into one |
| **SSL Termination** | Handle HTTPS, forward plain HTTP internally |
| **Circuit Breaking** | If a downstream service is failing, fail fast instead of waiting |
| **Logging** | Centralized request logging, distributed tracing |

**Popular API Gateways:**

| Gateway | Used By |
|---------|---------|
| **Nginx** | Most common, lightweight, fast |
| **Kong** | Built on Nginx, plugin-based |
| **AWS API Gateway** | Managed, serverless-friendly |
| **Zuul** | Netflix, Java-based |
| **Envoy** | Cloud-native, sidecar proxy |

### 3. Service Mesh (Brief)

A service mesh handles service-to-service communication in microservices. Instead of each service implementing retry logic, circuit breaking, and mTLS, a sidecar proxy handles it.

```
WITHOUT Service Mesh:
┌─────────────┐                    ┌─────────────┐
│ Service A    │──── HTTP ────────▶│ Service B    │
│ (implements  │                   │ (implements  │
│  retries,    │                   │  retries,    │
│  circuit     │                   │  TLS, etc.)  │
│  breaking,   │                   │              │
│  TLS, etc.)  │                   │              │
└─────────────┘                    └─────────────┘

WITH Service Mesh (Istio/Envoy):
┌─────────────┐     ┌─────────┐     ┌─────────┐     ┌─────────────┐
│ Service A    │────▶│ Envoy   │────▶│ Envoy   │────▶│ Service B    │
│ (just biz    │     │ Sidecar │     │ Sidecar │     │ (just biz    │
│  logic)      │     │ (handles│     │ (handles│     │  logic)      │
└─────────────┘     │ retries,│     │ mTLS,   │     └─────────────┘
                     │ mTLS,   │     │ metrics)│
                     │ metrics)│     └─────────┘
                     └─────────┘

Services focus on business logic. Proxies handle networking.
```

**When to mention in interviews:** Only for complex microservices architectures with many services. Don't over-engineer — most interview designs don't need a service mesh.

---

## Comparison Table

| Component | What It Is | When to Use |
|-----------|-----------|-------------|
| **Load Balancer** | Distributes traffic across servers | Always (any multi-server architecture) |
| **Reverse Proxy** | Hides backend, SSL termination, caching | Always (security, flexibility) |
| **API Gateway** | Reverse proxy + auth, routing, rate limiting | Microservices architectures |
| **Service Mesh** | Sidecar proxies for service-to-service communication | Large microservices (50+ services) |

---

## Common Interview Questions

### Q1: "Where does authentication happen in your design?"

**Model answer:**
"At the API gateway. When a request comes in, the gateway verifies the JWT token or API key before routing to any backend service. This centralizes authentication — individual services don't need to implement it. The gateway also attaches user identity information to the request headers so downstream services know who the caller is without re-validating the token."

### Q2: "What's the difference between a load balancer and an API gateway?"

**Model answer:**
"A load balancer distributes traffic across multiple instances of the same service. An API gateway routes different requests to different services and adds cross-cutting concerns like auth, rate limiting, and logging. In practice, the API gateway often includes load balancing. I'd use them together: API gateway at the edge for routing and auth, with load balancers in front of each service's instances."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 14 CHEAT SHEET                        │
│                  Reverse Proxies & API Gateways                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  REVERSE PROXY: Hides backend, SSL termination, caching         │
│  API GATEWAY: Reverse proxy + auth, rate limiting, routing      │
│                                                                  │
│  API GATEWAY RESPONSIBILITIES:                                   │
│  Auth, rate limiting, routing, SSL, logging, circuit breaking   │
│                                                                  │
│  POPULAR: Nginx, Kong, AWS API Gateway, Envoy                  │
│  SERVICE MESH: Istio/Envoy — sidecar proxies between services  │
│                                                                  │
│  INTERVIEW: "Auth and rate limiting at the API gateway layer"   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is a reverse proxy? | A server-side proxy that hides backend servers from clients. Handles SSL termination, caching, load balancing. |
| 2 | Forward proxy vs reverse proxy? | Forward: client-side (hides client from server). Reverse: server-side (hides server from client). |
| 3 | What are the key responsibilities of an API gateway? | Authentication, rate limiting, request routing, SSL termination, logging, circuit breaking, response aggregation |
| 4 | What is a service mesh? | A layer of sidecar proxies (like Envoy) that handle service-to-service communication: retries, mTLS, observability. Services focus on business logic. |
| 5 | When should you mention an API gateway in an interview? | Any microservices design. It centralizes auth, rate limiting, and routing — the interviewer will ask where these happen. |
| 6 | What is circuit breaking? | If a downstream service is failing, the gateway/proxy stops sending requests to it temporarily, failing fast instead of waiting for timeouts. |

---

[← Previous: CDN & Edge](./chapter-13-cdn-edge.md) · [Next: Message Queues →](../phase-4-infrastructure/chapter-15-message-queues.md)
