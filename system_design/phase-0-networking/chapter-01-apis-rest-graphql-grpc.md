# Chapter 1: APIs — REST, GraphQL, gRPC

> **Phase 0 — Networking & Communication** · Estimated Time: 2.5 hours · Difficulty: 🟢 Beginner

[← Previous: How the Internet Works](./chapter-00-internet-dns-tcp-http.md) · [Next: Communication Patterns →](./chapter-02-communication-patterns.md)

---

## Learning Objectives

After this chapter, you will be able to:

- ✅ Design RESTful APIs with proper resource naming, methods, and status codes
- ✅ Explain when to choose REST vs GraphQL vs gRPC
- ✅ Design pagination for large datasets (cursor-based vs offset-based)
- ✅ Understand Protocol Buffers and why gRPC is faster than REST
- ✅ Discuss API versioning strategies and their trade-offs
- ✅ Design API endpoints during a system design interview

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapter 0 (Internet, DNS, TCP, HTTP) | ✅ Yes |
| Basic HTTP understanding | ✅ Yes |
| Experience calling APIs | Helpful |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| API design practice | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** APIs are like restaurant menus. The menu (API specification) tells you what's available, how to order (request format), and what you'll get (response format). REST is like a traditional menu with fixed dishes. GraphQL is like a build-your-own-bowl where you pick exactly what you want. gRPC is like a private kitchen line where chefs communicate in shorthand — fast but you need to know the language.

**Interview relevance:** Every system design question requires you to design at least 3-5 API endpoints. Interviewers evaluate whether you can define clean, RESTful APIs with proper methods, status codes, and pagination. Choosing between REST and gRPC for internal microservice communication is a common follow-up question. Getting API design right shows you can think at the interface level, not just the infrastructure level.

---

## Visual Diagram — API Styles Compared

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API STYLES OVERVIEW                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  REST                    GraphQL                 gRPC               │
│  ────                    ───────                 ────               │
│                                                                     │
│  Client ──GET /users──▶  Client ──POST /graphql─▶ Client ──Proto──▶│
│         ◀──JSON────────         ◀──JSON (exact)──        ◀──Proto──│
│                                                                     │
│  Multiple endpoints      Single endpoint          Service methods   │
│  /users, /posts, etc.    query { user { name } }  GetUser(req)      │
│                                                                     │
│  JSON over HTTP          JSON over HTTP           Protobuf over     │
│                                                   HTTP/2            │
│                                                                     │
│  ┌─────────────┐        ┌─────────────┐         ┌─────────────┐   │
│  │ GET /users/1 │        │ query {      │         │ service User │   │
│  │ GET /posts   │        │   user(id:1) │         │ { rpc Get    │   │
│  │ POST /users  │        │   { name     │         │   (GetReq)   │   │
│  │ PUT /users/1 │        │     email    │         │   returns    │   │
│  │ DELETE /u/1  │        │     posts {  │         │   (GetResp)  │   │
│  │              │        │       title  │         │ }            │   │
│  └─────────────┘        │     }        │         └─────────────┘   │
│                          │   }          │                           │
│                          │ }            │                           │
│                          └─────────────┘                           │
│                                                                     │
│  Best for:               Best for:               Best for:          │
│  Public APIs,            Mobile apps,             Internal           │
│  CRUD operations,        complex queries,         microservices,     │
│  simple interfaces       avoiding over-fetch      low latency        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### 1. REST (REpresentational State Transfer)

REST is an architectural style (not a protocol) for designing APIs around **resources**. A resource is any named entity: a user, a tweet, a photo, an order.

#### REST Principles

1. **Resources as URLs:** Every entity gets a unique URL
   - `/users/123` — User with ID 123
   - `/users/123/posts` — Posts by user 123
   - `/posts/456/comments` — Comments on post 456

2. **HTTP methods as actions:** Use the right method for the right operation
   ```
   GET    /users        → List users
   GET    /users/123    → Get user 123
   POST   /users        → Create a new user
   PUT    /users/123    → Replace user 123 entirely
   PATCH  /users/123    → Update specific fields of user 123
   DELETE /users/123    → Delete user 123
   ```

3. **Stateless:** Each request contains all the information needed to process it. The server doesn't remember previous requests. Authentication is sent with every request (via headers).

4. **Standard status codes:** Use HTTP status codes to communicate outcomes (see Chapter 0).

#### REST API Design — Best Practices for Interviews

```
✅ GOOD REST Design:
  GET    /api/v1/tweets                    → List tweets
  GET    /api/v1/tweets/123                → Get tweet 123
  POST   /api/v1/tweets                    → Create tweet
  DELETE /api/v1/tweets/123                → Delete tweet
  GET    /api/v1/users/456/tweets          → Get user 456's tweets
  GET    /api/v1/tweets?hashtag=sde&limit=20  → Filter tweets

❌ BAD REST Design:
  GET    /getTweets                        → Verb in URL
  POST   /createTweet                      → Verb in URL
  GET    /api/tweet/delete/123             → Using GET for delete
  POST   /api/tweets/123                   → POST for a specific ID
```

**Key rules:**
- Use **nouns** for URLs, not verbs (the HTTP method IS the verb)
- Use **plural nouns** (`/users` not `/user`)
- Use **kebab-case** for multi-word resources (`/user-profiles`)
- Nest resources logically (`/users/123/orders`)
- Use **query parameters** for filtering, sorting, and pagination

#### Pagination — Two Approaches

When an API returns thousands of results, you paginate. Two main strategies:

**Offset-based pagination:**
```
GET /api/tweets?offset=0&limit=20    → Items 1-20
GET /api/tweets?offset=20&limit=20   → Items 21-40
GET /api/tweets?offset=40&limit=20   → Items 41-60
```

| Pros | Cons |
|------|------|
| Simple to implement | Breaks if items are inserted/deleted between pages |
| Can jump to page N directly | Expensive for large offsets (DB must scan & skip) |
| Easy to understand | Inconsistent results if data changes |

**Cursor-based pagination (recommended for most system designs):**
```
GET /api/tweets?limit=20
  Response: { tweets: [...], next_cursor: "tweet_id_20" }

GET /api/tweets?limit=20&cursor=tweet_id_20
  Response: { tweets: [...], next_cursor: "tweet_id_40" }
```

| Pros | Cons |
|------|------|
| Consistent even if data changes | Can't jump to page N |
| Efficient for large datasets (uses index) | More complex to implement |
| Works well with append-only data | Cursor must be opaque to client |

**Interview tip:** Always use cursor-based pagination for feeds and timelines. When designing Twitter's timeline or a news feed, say "I'd use cursor-based pagination because the feed is constantly changing and offset pagination would give inconsistent results."

---

### 2. GraphQL

GraphQL is a query language for APIs, developed by Facebook in 2012 and open-sourced in 2015. Instead of multiple endpoints, you have a **single endpoint** where the client specifies exactly what data it needs.

#### How GraphQL Works

```
┌──────────────┐                          ┌──────────────┐
│   Client      │    POST /graphql        │   Server      │
│               │                          │               │
│  query {      │─────────────────────────▶│  Resolver     │
│    user(id:1) │                          │  functions    │
│    {          │                          │  fetch data   │
│      name     │                          │  from DBs,    │
│      email    │                          │  services     │
│      posts {  │                          │               │
│        title  │◀─────────────────────────│  Returns      │
│      }        │                          │  exactly      │
│    }          │   { "data": {            │  what was     │
│  }            │     "user": {            │  requested    │
│               │       "name": "John",    │               │
│               │       "email": "j@x.com",│               │
│               │       "posts": [         │               │
│               │         {"title":"Hi"}   │               │
│               │       ]                  │               │
│               │     }                    │               │
│               │   }}                     │               │
└──────────────┘                          └──────────────┘
```

#### GraphQL Operations

```
# QUERY — Read data (like GET)
query {
  user(id: "123") {
    name
    email
    followers {
      name
    }
  }
}

# MUTATION — Modify data (like POST/PUT/DELETE)
mutation {
  createTweet(content: "Hello world") {
    id
    content
    createdAt
  }
}

# SUBSCRIPTION — Real-time updates (like WebSocket)
subscription {
  newMessage(chatId: "456") {
    content
    sender {
      name
    }
  }
}
```

#### GraphQL Advantages

| Advantage | Explanation |
|-----------|-------------|
| **No over-fetching** | Client gets exactly the fields it needs, not the entire resource |
| **No under-fetching** | Get related data in one request (user + their posts) instead of multiple REST calls |
| **Strong typing** | Schema defines types, preventing runtime errors |
| **Self-documenting** | The schema IS the documentation |
| **Single endpoint** | Simplifies API gateway routing |

#### GraphQL Disadvantages

| Disadvantage | Explanation |
|-------------|-------------|
| **Caching is harder** | Can't use HTTP caching easily (all requests are POST to the same URL) |
| **N+1 query problem** | Naive resolvers can cause many DB queries (mitigated by DataLoader) |
| **Complexity** | More complex server-side implementation than REST |
| **Rate limiting is harder** | Can't rate-limit by endpoint — a single query can be cheap or expensive |
| **File uploads** | Not natively supported (need multipart workarounds) |

---

### 3. gRPC (Google Remote Procedure Call)

gRPC is a high-performance RPC framework that uses **Protocol Buffers** (protobuf) for serialization and **HTTP/2** for transport. It's the go-to choice for internal microservice communication.

#### How gRPC Works

```
┌──────────────┐                          ┌──────────────┐
│  Client       │   HTTP/2 + Protobuf     │  Server       │
│  (generated   │                          │  (implements  │
│   stub)       │─── Binary protobuf ────▶│   service)    │
│               │                          │               │
│  user = stub  │◀── Binary protobuf ─────│               │
│  .GetUser(req)│                          │               │
└──────────────┘                          └──────────────┘

Proto Definition (user.proto):
┌─────────────────────────────────────────┐
│ syntax = "proto3";                       │
│                                          │
│ service UserService {                    │
│   rpc GetUser (GetUserRequest)           │
│     returns (GetUserResponse);           │
│   rpc CreateUser (CreateUserRequest)     │
│     returns (CreateUserResponse);        │
│   rpc ListUsers (ListUsersRequest)       │
│     returns (stream User);              │
│ }                                        │
│                                          │
│ message GetUserRequest {                 │
│   string user_id = 1;                    │
│ }                                        │
│                                          │
│ message GetUserResponse {                │
│   string user_id = 1;                    │
│   string name = 2;                       │
│   string email = 3;                      │
│   int64 created_at = 4;                  │
│ }                                        │
└─────────────────────────────────────────┘
```

#### Why gRPC Is Faster Than REST

```
REST (JSON over HTTP/1.1):
┌────────────────────────────────────────────────────┐
│ {"user_id":"123","name":"John","email":"john@x.com │
│ ","created_at":1634567890,"avatar_url":"https://..."│
│ ,"bio":"Software engineer","followers_count":1234}  │
│                                                    │
│ Size: ~200 bytes (text, with field names repeated) │
│ Parse time: JSON parsing is slow                   │
│ Transport: HTTP/1.1 (one request at a time)        │
└────────────────────────────────────────────────────┘

gRPC (Protobuf over HTTP/2):
┌────────────────────────────────────────────────────┐
│ 0a 03 31 32 33 12 04 4a 6f 68 6e 1a 0c 6a 6f 68  │
│ 6e 40 78 2e 63 6f 6d 20 d2 85 ad 94 06 ...        │
│                                                    │
│ Size: ~60 bytes (binary, no field names)           │
│ Parse time: Protobuf parsing is 10-100x faster     │
│ Transport: HTTP/2 (multiplexed, compressed)        │
└────────────────────────────────────────────────────┘
```

| Feature | gRPC Advantage |
|---------|----------------|
| **Serialization** | Protobuf is 3-10x smaller than JSON |
| **Parsing speed** | 10-100x faster than JSON parsing |
| **Transport** | HTTP/2 multiplexing, header compression |
| **Streaming** | Native support for server/client/bidirectional streaming |
| **Code generation** | Auto-generates client/server stubs from .proto files |
| **Strong typing** | Compile-time type checking |

#### gRPC Streaming Modes

```
1. UNARY (request-response, like REST):
   Client ──── Request ────▶ Server
   Client ◀─── Response ─── Server

2. SERVER STREAMING (server sends multiple responses):
   Client ──── Request ────▶ Server
   Client ◀─── Response 1 ── Server
   Client ◀─── Response 2 ── Server
   Client ◀─── Response N ── Server
   Use case: Real-time stock prices, log streaming

3. CLIENT STREAMING (client sends multiple requests):
   Client ──── Request 1 ──▶ Server
   Client ──── Request 2 ──▶ Server
   Client ──── Request N ──▶ Server
   Client ◀─── Response ─── Server
   Use case: File upload, batch processing

4. BIDIRECTIONAL STREAMING:
   Client ◀──▶ Server (both send messages independently)
   Use case: Chat, real-time collaboration
```

#### When NOT to Use gRPC

- **Public-facing APIs** — browsers can't natively call gRPC (need gRPC-Web proxy)
- **Simple CRUD** — REST is simpler for basic operations
- **When human readability matters** — protobuf is binary, not inspectable in logs
- **Loose coupling** — gRPC requires shared .proto definitions between services

---

### 4. API Versioning

APIs evolve. You need a strategy to make changes without breaking existing clients.

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL versioning** | `/api/v1/users` | Clear, easy to understand | Pollutes URLs, hard to deprecate |
| **Header versioning** | `Accept: application/vnd.api.v2+json` | Clean URLs | Hidden, harder to test |
| **Query param** | `/api/users?version=2` | Flexible | Easy to miss |

**Interview tip:** URL versioning (`/api/v1/`) is the most common and easiest to explain. Use it in your system designs.

---

## Comparison Table — REST vs GraphQL vs gRPC

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP (POST) | HTTP/2 |
| **Data format** | JSON (text) | JSON (text) | Protobuf (binary) |
| **Endpoints** | Multiple (`/users`, `/posts`) | Single (`/graphql`) | Service methods |
| **Over-fetching** | Common | ❌ No | ❌ No |
| **Under-fetching** | Common (need multiple calls) | ❌ No | Possible |
| **Caching** | Easy (HTTP caching) | Hard | Hard |
| **Learning curve** | Low | Medium | Medium-High |
| **Browser support** | ✅ Native | ✅ Via POST | ❌ Need proxy |
| **Streaming** | ❌ (need WebSocket) | ✅ Subscriptions | ✅ Native |
| **Performance** | Good | Good | Excellent |
| **Best for** | Public APIs, CRUD, simple apps | Mobile apps, complex queries | Internal microservices |

### Decision Framework

```
Choosing an API style:

Is it a PUBLIC-facing API?
├── Yes → REST (universal, well-understood, cacheable)
│         GraphQL if clients need flexibility (mobile apps)
└── No (internal microservice-to-microservice) →
    ├── Need low latency & high throughput? → gRPC
    ├── Need streaming? → gRPC
    ├── Simple CRUD between services? → REST is fine
    └── Complex data requirements? → GraphQL
```

---

## Real-World Example — How Netflix Uses APIs

```
Netflix API Architecture:

┌────────────┐     REST/GraphQL      ┌─────────────┐      gRPC        ┌────────────┐
│  Mobile     │─────────────────────▶│  API Gateway  │────────────────▶│ User        │
│  App        │◀─────────────────────│  (Zuul)       │◀────────────────│ Service     │
└────────────┘                       │               │                 └────────────┘
                                     │               │      gRPC        ┌────────────┐
┌────────────┐     REST/GraphQL      │               │────────────────▶│ Content     │
│  Web App    │─────────────────────▶│               │◀────────────────│ Service     │
│  (Browser)  │◀─────────────────────│               │                 └────────────┘
└────────────┘                       │               │      gRPC        ┌────────────┐
                                     │               │────────────────▶│ Recommend.  │
┌────────────┐     REST              │               │◀────────────────│ Service     │
│  Smart TV   │─────────────────────▶│               │                 └────────────┘
│  App        │◀─────────────────────│               │
└────────────┘                       └─────────────┘

Key decisions:
• External APIs: REST for TVs/browsers, GraphQL for mobile (reduce data transfer)
• Internal APIs: gRPC between microservices (low latency, type safety)
• API Gateway: Handles auth, rate limiting, routing, response aggregation
```

Netflix originally used REST everywhere, but migrated to a **federated GraphQL** layer for external clients to solve the over-fetching problem (mobile devices on slow networks were downloading too much data), and **gRPC** internally for performance between their ~1000+ microservices.

---

## Common Interview Questions

### Q1: "When would you use REST vs GraphQL vs gRPC?"

**Model answer:**
"I use REST for public-facing APIs because it's the most universal — any HTTP client can consume it, caching is straightforward with CDNs, and the learning curve is low. I'd choose GraphQL when I have mobile clients that need to minimize data transfer, or when the frontend needs to fetch complex, nested data in a single request — like a social media profile with posts, followers, and notifications. For internal microservice communication, I'd use gRPC because it uses protobuf which is 3-10x smaller and 10-100x faster to parse than JSON, plus it runs on HTTP/2 with multiplexing and native streaming support."

### Q2: "How would you design pagination for a Twitter-like timeline?"

**Model answer:**
"I'd use cursor-based pagination. The first request is `GET /api/v1/timeline?limit=20`, which returns the 20 most recent tweets and a `next_cursor` pointing to the last tweet's ID or timestamp. The next request includes that cursor: `GET /api/v1/timeline?limit=20&cursor=tweet_1234`. This approach is better than offset pagination because the timeline is constantly changing — new tweets are being posted — so offset pagination would give duplicate or missing tweets. The cursor-based approach always gives a consistent window of tweets."

### Q3: "What is idempotency and why does it matter?"

**Model answer:**
"Idempotency means performing an operation multiple times has the same effect as performing it once. GET, PUT, and DELETE are idempotent — if you DELETE /users/123 twice, the result is the same (user is deleted). POST is not idempotent — POSTing twice might create two resources. This matters in system design because networks are unreliable — if a client sends a request and doesn't get a response, it will retry. For non-idempotent operations like payment processing, I'd implement an idempotency key: the client generates a unique key per operation, and the server checks if that key was already processed before executing."

### Q4: "How would you handle API versioning?"

**Model answer:**
"I'd use URL-based versioning — `/api/v1/users` and `/api/v2/users`. It's the most explicit and easiest for API consumers to understand. When we need to make a breaking change, we deploy v2 alongside v1, give clients a migration window (say 6 months), and then deprecate v1. For non-breaking changes like adding a new field, I'd just add it to the existing version — well-designed clients should ignore unknown fields."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "We'll use GraphQL for everything" | Choose the right tool: REST for public APIs, gRPC for internal services, GraphQL when client flexibility is critical |
| Using verbs in REST URLs: `POST /createUser` | REST URLs are nouns: `POST /users`. The HTTP method is the verb. |
| Returning 200 for everything | Use proper status codes: 201 for creation, 404 for not found, 429 for rate limiting |
| "gRPC is always better than REST" | gRPC requires protobuf schemas shared between services. It doesn't work natively in browsers. REST is simpler for many use cases. |
| Using offset pagination for feeds | Feeds change constantly. Use cursor-based pagination for any append-heavy, time-ordered data. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 1 CHEAT SHEET                         │
│                  APIs: REST, GraphQL, gRPC                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  REST:                                                           │
│  • Resources as URLs, HTTP methods as actions                    │
│  • GET/PUT/DELETE = idempotent, POST = not                       │
│  • Pagination: cursor-based for feeds, offset for tables         │
│  • Version: /api/v1/resource                                     │
│  • Best for: public APIs, simple CRUD                            │
│                                                                  │
│  GraphQL:                                                        │
│  • Single endpoint, client specifies exact data needed           │
│  • Query (read), Mutation (write), Subscription (realtime)       │
│  • Solves over-fetching and under-fetching                       │
│  • Harder to cache, rate-limit, and secure                       │
│  • Best for: mobile apps, complex nested data                    │
│                                                                  │
│  gRPC:                                                           │
│  • Protobuf (binary) over HTTP/2                                 │
│  • 3-10x smaller, 10-100x faster parsing than JSON              │
│  • 4 streaming modes: unary, server, client, bidirectional      │
│  • Auto-generated client stubs from .proto files                │
│  • Best for: internal microservices, low latency                │
│                                                                  │
│  INTERVIEW DEFAULTS:                                             │
│  • External → REST                                               │
│  • Internal → gRPC                                               │
│  • Mobile with complex queries → GraphQL                         │
│  • Always mention pagination strategy                            │
│  • Always version your API (/api/v1/)                           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 5 main HTTP methods used in REST? | GET (read), POST (create), PUT (replace), PATCH (update), DELETE (remove) |
| 2 | What's the difference between cursor-based and offset-based pagination? | Offset uses skip+limit (breaks with changing data), cursor uses a pointer to the last seen item (consistent for feeds) |
| 3 | What problem does GraphQL solve that REST doesn't? | Over-fetching (getting fields you don't need) and under-fetching (needing multiple REST calls to assemble data) |
| 4 | Why is gRPC faster than REST? | Protobuf binary serialization (3-10x smaller than JSON, 10-100x faster parsing) + HTTP/2 multiplexing |
| 5 | What are the 4 gRPC streaming modes? | Unary (request-response), server streaming, client streaming, bidirectional streaming |
| 6 | When should you NOT use gRPC? | Public/browser-facing APIs (no native browser support), when human readability matters, simple CRUD operations |
| 7 | What is an idempotency key? | A unique key sent by the client to ensure a non-idempotent operation (like payment) is only processed once, even if the request is retried |
| 8 | What's the best API versioning strategy for interviews? | URL versioning: `/api/v1/resource` — explicit, easy to understand, easy to route |
| 9 | How does Netflix use different API styles? | GraphQL for mobile clients (reduce data transfer), gRPC for internal microservices (performance), REST for simple integrations |
| 10 | What makes a REST API "RESTful"? | Resources as URLs, HTTP methods for actions, stateless, standard status codes, uniform interface |

---

[← Previous: How the Internet Works](./chapter-00-internet-dns-tcp-http.md) · [Next: Communication Patterns →](./chapter-02-communication-patterns.md)
