# 🌐 Phase 0 — Networking & Communication Fundamentals

> *You can't design systems if you don't understand how machines talk to each other. This phase gives you the language to discuss system design fluently.*

**⏱ Estimated Time: ~7.5 hours**

---

## Chapters in This Phase

| # | Chapter | Time | Difficulty |
|---|---------|------|------------|
| 0 | [How the Internet Works — DNS, TCP, HTTP](./chapter-00-internet-dns-tcp-http.md) | 2.5 hrs | 🟢 Beginner |
| 1 | [APIs — REST, GraphQL, gRPC](./chapter-01-apis-rest-graphql-grpc.md) | 2.5 hrs | 🟢 Beginner |
| 2 | [Communication Patterns](./chapter-02-communication-patterns.md) | 2.5 hrs | 🟡 Medium |

---

## What You'll Learn

After completing this phase, you'll be able to:

- ✅ Explain how a URL typed in a browser reaches a server and returns a response
- ✅ Choose the right API style (REST/GraphQL/gRPC) for a given use case
- ✅ Discuss real-time communication trade-offs in an interview
- ✅ Draw a complete request lifecycle diagram on a whiteboard
- ✅ Compare WebSockets, SSE, long-polling, and know when to use each

---

## Why This Phase Matters

Every system design answer starts with "the client sends a request to the server." If you don't understand *how* that request travels — DNS resolution, TCP connections, HTTP semantics, API design — your entire design rests on a shaky foundation.

Interviewers won't ask you to explain TCP in detail, but they **will** notice if you:
- Don't know the difference between HTTP and WebSocket (when designing a chat system)
- Can't explain why gRPC is faster than REST (when designing microservices)
- Don't mention DNS when discussing how users connect to your system

---

## Checkpoint Exercise

Before moving to Phase 1, make sure you can:

1. **Explain the full lifecycle of an HTTP request** — from typing a URL to rendering the page
2. **Compare REST vs gRPC** for a microservices architecture — when would you use each?
3. **Design the communication layer** for a real-time dashboard — which pattern (WebSocket, SSE, polling) and why?

---

[← Course Home](../README.md) · [Phase 1 — Scalability →](../phase-1-scalability/README.md)
