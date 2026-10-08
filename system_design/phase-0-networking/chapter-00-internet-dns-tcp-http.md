# Chapter 0: How the Internet Works — DNS, TCP, HTTP

> **Phase 0 — Networking & Communication** · Estimated Time: 2.5 hours · Difficulty: 🟢 Beginner

[← Phase 0 Overview](./README.md) · [Next: APIs — REST, GraphQL, gRPC →](./chapter-01-apis-rest-graphql-grpc.md)

---

## Learning Objectives

After this chapter, you will be able to:

- ✅ Trace the complete lifecycle of an HTTP request from browser to server and back
- ✅ Explain DNS resolution including recursive and iterative queries
- ✅ Compare TCP vs UDP and know when to use each in system design
- ✅ Describe HTTP/1.1 vs HTTP/2 vs HTTP/3 and their performance implications
- ✅ Explain the TLS handshake and why HTTPS matters for system design
- ✅ Draw a request lifecycle diagram on a whiteboard in an interview

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Basic programming knowledge | Yes |
| Understanding of client-server model | Helpful |
| Prior networking experience | Not required |

| Component | Time |
|-----------|------|
| Reading & concepts | 1.5 hrs |
| Diagrams & practice | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** Imagine you want to send a letter to a friend. You need to know their address (DNS), you need a reliable postal service that confirms delivery (TCP), and you need to write the letter in a language they understand (HTTP). The internet works exactly the same way.

**Interview relevance:** Every single system design question starts with a client talking to a server. When you design a URL shortener, the first thing that happens is a user types `short.url/abc123` in their browser. Understanding what happens between that keystroke and the redirect is foundational. Interviewers won't grill you on TCP details, but they **will** expect you to casually mention DNS, load balancers, and HTTPS when walking through your high-level design.

---

## Visual Diagram — The Complete Request Lifecycle

```
What happens when you type "https://www.example.com/page" in your browser?

┌──────────┐
│  Browser  │
└─────┬─────┘
      │
      │ 1. Parse URL → protocol (https), domain (www.example.com), path (/page)
      │
      │ 2. DNS Resolution (find the IP address)
      ▼
┌──────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│  Browser  │────▶│ DNS Resolver  │────▶│ Root DNS (.com)│────▶│ Authoritative │
│  Cache    │     │ (ISP/Google)  │     │  TLD Server    │     │ DNS Server    │
└──────────┘     └──────────────┘     └───────────────┘     └──────┬───────┘
                                                                    │
                                                          IP: 93.184.216.34
                                                                    │
      ┌─────────────────────────────────────────────────────────────┘
      │
      │ 3. TCP Handshake (establish reliable connection)
      ▼
┌──────────┐                                          ┌──────────┐
│  Client   │ ──── SYN ─────────────────────────────▶ │  Server   │
│           │ ◀─── SYN-ACK ─────────────────────────  │  :443     │
│           │ ──── ACK ─────────────────────────────▶ │           │
└──────────┘                                          └──────────┘

      │ 4. TLS Handshake (if HTTPS — establish encrypted channel)
      ▼
┌──────────┐                                          ┌──────────┐
│  Client   │ ──── ClientHello (supported ciphers) ──▶│  Server   │
│           │ ◀─── ServerHello + Certificate ─────── │           │
│           │ ──── Key Exchange ──────────────────▶   │           │
│           │ ◀─── Finished ─────────────────────── │           │
└──────────┘                                          └──────────┘

      │ 5. HTTP Request
      ▼
┌──────────┐         GET /page HTTP/1.1               ┌──────────┐
│  Client   │ ──── Host: www.example.com ───────────▶ │  Server   │
│           │       Accept: text/html                  │           │
│           │                                          │           │
│           │ ◀─── HTTP/1.1 200 OK ────────────────── │           │
│           │       Content-Type: text/html            │           │
│           │       <html>...</html>                    │           │
└──────────┘                                          └──────────┘

      │ 6. Browser renders the HTML, loads CSS/JS/images (more requests)
      ▼
┌──────────┐
│  Rendered │
│  Page     │
└──────────┘
```

---

## Core Concepts

### 1. DNS — The Internet's Phone Book

DNS (Domain Name System) translates human-readable domain names (`www.google.com`) into IP addresses (`142.250.80.46`) that computers use to find each other.

#### How DNS Resolution Works

```
Step-by-step DNS resolution for "www.example.com":

┌──────────┐    ①    ┌──────────────┐    ②    ┌─────────────┐
│  Browser  │───────▶│  Recursive    │───────▶│  Root DNS    │
│           │        │  Resolver     │        │  Server (.)  │
│           │        │  (ISP/8.8.8.8)│◀───────│  "Try .com"  │
│           │        │               │   ③    └─────────────┘
│           │        │               │
│           │        │               │───────▶┌─────────────┐
│           │        │               │   ④    │  TLD DNS     │
│           │        │               │◀───────│  Server(.com)│
│           │        │               │   ⑤    │  "Try NS for │
│           │        │               │        │   example.com"│
│           │        │               │        └─────────────┘
│           │        │               │
│           │        │               │───────▶┌──────────────┐
│           │        │               │   ⑥    │ Authoritative │
│           │◀───────│               │◀───────│ DNS Server    │
│           │   ⑧    │               │   ⑦    │ 93.184.216.34 │
└──────────┘        └──────────────┘        └──────────────┘
```

**DNS Caching** happens at multiple layers:
1. **Browser cache** — Chrome caches DNS for ~1 minute
2. **OS cache** — your operating system maintains a DNS cache
3. **Router cache** — your home/office router caches DNS
4. **ISP DNS resolver cache** — the recursive resolver caches based on TTL

**TTL (Time to Live)** is how long a DNS record can be cached. Low TTL (e.g., 60 seconds) means faster propagation when you change IPs, but more DNS queries. High TTL (e.g., 24 hours) reduces DNS load but means slower failover.

#### DNS Record Types You Should Know

| Record Type | Purpose | Example |
|------------|---------|---------|
| **A** | Maps domain to IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | Maps domain to IPv6 address | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias to another domain | `www.example.com → example.com` |
| **NS** | Specifies authoritative name servers | `example.com NS ns1.example.com` |
| **MX** | Mail server for the domain | `example.com MX mail.example.com` |
| **TXT** | Arbitrary text (used for verification) | SPF, DKIM, domain ownership |

**Interview tip:** DNS comes up in system design when discussing:
- **Load balancing** — DNS-based load balancing (round-robin DNS, GeoDNS)
- **CDN** — CDN providers use DNS to route users to the nearest edge server
- **Failover** — Changing DNS records to point to a backup server during outages

---

### 2. TCP vs UDP — Reliable vs Fast

TCP and UDP are the two main transport protocols. Every system design decision about networking comes down to: "Do I need reliability or speed?"

#### TCP (Transmission Control Protocol)

TCP guarantees reliable, ordered delivery of data. It's like a phone call — you establish a connection, talk, and hang up.

**The TCP 3-Way Handshake:**

```
Client                    Server
  │                          │
  │──── SYN (seq=x) ───────▶│   "Hey, I want to connect"
  │                          │
  │◀─── SYN-ACK (seq=y, ──── │   "OK, I'm ready too"
  │      ack=x+1)            │
  │                          │
  │──── ACK (ack=y+1) ──────▶│   "Great, let's talk"
  │                          │
  │   Connection Established  │
```

**TCP Key Features:**
- **Reliable delivery** — retransmits lost packets
- **Ordered delivery** — packets arrive in sequence
- **Flow control** — sender slows down if receiver is overwhelmed
- **Congestion control** — backs off when network is congested
- **Connection-oriented** — must establish connection before sending data

#### UDP (User Datagram Protocol)

UDP is "fire and forget." No connection, no guarantees. It's like dropping a postcard in the mailbox — fast, but no confirmation of delivery.

**UDP Key Features:**
- **No connection setup** — no handshake overhead
- **No delivery guarantee** — packets may be lost
- **No ordering** — packets may arrive out of order
- **Low latency** — no retransmission delays
- **Low overhead** — smaller header (8 bytes vs TCP's 20+ bytes)

#### When to Use Each

| Criteria | TCP | UDP |
|----------|-----|-----|
| Reliability needed? | ✅ Yes | ❌ No |
| Order matters? | ✅ Yes | ❌ No |
| Latency critical? | ❌ Higher | ✅ Lower |
| **Use cases** | HTTP, email, file transfer, database connections | Video streaming, gaming, DNS queries, VoIP |
| **System design use** | Almost everything | Real-time features, live video, DNS |

**Interview tip:** When designing a video streaming system (YouTube/Netflix), mention that the actual video stream often uses UDP-based protocols (QUIC, WebRTC) because dropping a video frame is better than waiting for retransmission. But the API calls (search, recommendations, auth) use TCP/HTTP.

---

### 3. HTTP — The Language of the Web

HTTP (HyperText Transfer Protocol) is the application-layer protocol that defines how clients and servers communicate. It runs on top of TCP (or QUIC for HTTP/3).

#### HTTP Request Structure

```
┌─────────────────────────────────────────────┐
│  Request Line                                │
│  GET /api/users/123 HTTP/1.1                 │
├─────────────────────────────────────────────┤
│  Headers                                     │
│  Host: api.example.com                       │
│  Authorization: Bearer eyJhbGci...           │
│  Accept: application/json                    │
│  Content-Type: application/json              │
├─────────────────────────────────────────────┤
│  Body (optional, for POST/PUT/PATCH)         │
│  { "name": "John", "email": "john@x.com" }  │
└─────────────────────────────────────────────┘
```

#### HTTP Response Structure

```
┌─────────────────────────────────────────────┐
│  Status Line                                 │
│  HTTP/1.1 200 OK                             │
├─────────────────────────────────────────────┤
│  Headers                                     │
│  Content-Type: application/json              │
│  Cache-Control: max-age=3600                 │
│  X-Request-Id: abc-123                       │
├─────────────────────────────────────────────┤
│  Body                                        │
│  { "id": 123, "name": "John" }               │
└─────────────────────────────────────────────┘
```

#### HTTP Methods You Must Know

| Method | Purpose | Idempotent? | Safe? |
|--------|---------|-------------|-------|
| **GET** | Retrieve data | ✅ Yes | ✅ Yes |
| **POST** | Create resource | ❌ No | ❌ No |
| **PUT** | Replace resource entirely | ✅ Yes | ❌ No |
| **PATCH** | Partially update resource | ❌ No | ❌ No |
| **DELETE** | Remove resource | ✅ Yes | ❌ No |

**Idempotent** means calling it multiple times has the same effect as calling it once. This matters for retry logic — you can safely retry a GET or PUT, but retrying a POST might create duplicate records.

#### HTTP Status Codes — The Essential Ones

```
2xx — Success
  200 OK              — Standard success
  201 Created         — Resource created (after POST)
  204 No Content      — Success but no body (after DELETE)

3xx — Redirection
  301 Moved Permanently — URL shortener redirect
  302 Found            — Temporary redirect
  304 Not Modified     — Use cached version

4xx — Client Error
  400 Bad Request      — Malformed request
  401 Unauthorized     — Not authenticated
  403 Forbidden        — Authenticated but not authorized
  404 Not Found        — Resource doesn't exist
  409 Conflict         — Conflicting update
  429 Too Many Requests — Rate limited

5xx — Server Error
  500 Internal Server Error — Server crashed
  502 Bad Gateway      — Upstream server error
  503 Service Unavailable — Server overloaded/maintenance
  504 Gateway Timeout  — Upstream server timeout
```

**Interview tip:** When you design an API in a system design interview, mention specific status codes. "The create endpoint returns 201 with the created resource, and 409 if there's a collision" shows attention to detail.

---

### 4. HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP Evolution:

HTTP/1.0 (1996)     HTTP/1.1 (1997)     HTTP/2 (2015)      HTTP/3 (2022)
──────────────      ──────────────      ─────────────      ─────────────
One request per     Keep-alive          Multiplexing       QUIC (UDP-based)
connection          (reuse conn)        (many requests     No head-of-line
                                        on one conn)       blocking
No caching          Caching headers     Header             0-RTT connection
                    Chunked transfer    compression        setup
                    Host header         Server push        Built-in TLS
                    (virtual hosting)   Binary protocol    Connection
                                                           migration
```

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | ❌ (head-of-line blocking) | ✅ | ✅ |
| Header compression | ❌ | ✅ (HPACK) | ✅ (QPACK) |
| Server push | ❌ | ✅ | ✅ |
| Connection setup | TCP + TLS (2-3 RTTs) | TCP + TLS (2-3 RTTs) | 1 RTT (0-RTT for repeat) |
| Head-of-line blocking | TCP level | TCP level (reduced) | ❌ None |

**Interview tip:** You rarely need to discuss HTTP versions in depth, but mentioning "HTTP/2 multiplexing means we can send multiple requests on a single connection" when designing a client-heavy system (like a web dashboard) shows depth. For mobile apps or real-time systems, mentioning HTTP/3's reduced latency is a plus.

---

### 5. HTTPS & the TLS Handshake

HTTPS = HTTP + TLS (Transport Layer Security). TLS encrypts the communication so no one between client and server can read or tamper with the data.

#### Why HTTPS Matters for System Design

- **Data integrity** — prevents man-in-the-middle attacks
- **Authentication** — the certificate proves the server is who it claims to be
- **Required for HTTP/2 and HTTP/3** — browsers enforce this
- **SEO ranking factor** — Google ranks HTTPS sites higher
- **User trust** — the padlock icon

#### The TLS Handshake (Simplified — TLS 1.3)

```
Client                                   Server
  │                                         │
  │ ─── ClientHello ──────────────────────▶ │
  │     (supported cipher suites,           │
  │      client random, key share)          │
  │                                         │
  │ ◀── ServerHello + Certificate ──────── │
  │     (chosen cipher suite,               │
  │      server random, key share,          │
  │      server certificate)                │
  │                                         │
  │  [Client verifies certificate            │
  │   against trusted CAs]                   │
  │                                         │
  │  [Both compute shared secret             │
  │   from key exchange]                     │
  │                                         │
  │ ─── Finished ─────────────────────────▶ │
  │                                         │
  │ ◀── Finished ──────────────────────── │
  │                                         │
  │  ═══ Encrypted communication ═══════  │
```

**TLS 1.3** reduced the handshake from 2 round trips (TLS 1.2) to 1 round trip, and supports 0-RTT for repeat connections.

**Interview tip:** When designing any system, always assume HTTPS. If the interviewer asks about security, mention "all communication is over TLS" and move on. Don't spend 5 minutes on the TLS handshake unless specifically asked.

---

## Comparison Table — TCP vs UDP vs HTTP

| Aspect | TCP | UDP | HTTP |
|--------|-----|-----|------|
| Layer | Transport (L4) | Transport (L4) | Application (L7) |
| Runs on | IP | IP | TCP (or QUIC/UDP for HTTP/3) |
| Connection | Connection-oriented | Connectionless | Request-response |
| Reliability | Guaranteed delivery | Best effort | Depends on TCP underneath |
| Use in system design | Database connections, API calls | Video streaming, gaming, DNS | REST APIs, web pages |

---

## Real-World Example — How Google Search Works (Networking Perspective)

When you type "system design interview" into Google:

```
1. BROWSER → DNS Resolver
   - Check browser cache, OS cache
   - Query ISP DNS resolver
   - Resolver queries root → .com TLD → google.com authoritative
   - Returns IP: e.g., 142.250.80.46
   - Total time: ~20-50ms (if not cached)

2. BROWSER → TCP + TLS to Google's edge server
   - Google uses Anycast IP routing — the same IP routes to the
     nearest datacenter based on BGP routing
   - TCP handshake: ~10-30ms (intra-continent)
   - TLS handshake: ~10-30ms (TLS 1.3 = 1 RTT)

3. BROWSER → HTTP/2 GET /search?q=system+design+interview
   - Request sent over encrypted HTTP/2 connection
   - Google's load balancer routes to a search frontend server

4. GOOGLE INTERNAL
   - Frontend server → Index servers (find matching documents)
   - Index servers → Document servers (retrieve snippets)
   - Ranking service scores and orders results
   - Ad server inserts relevant ads
   - All internal communication via gRPC over HTTP/2
   - Total processing: ~200-400ms

5. SERVER → BROWSER HTTP/2 200 OK
   - Response: ~100-500 KB of HTML
   - Browser renders the search results page
   - Additional requests for images, CSS, JS (multiplexed over HTTP/2)

Total time: ~300-800ms from keystroke to rendered results
```

**Key takeaway:** Google uses DNS (with Anycast), TCP/TLS, HTTP/2, and internal gRPC — every concept from this chapter in one system.

---

## Common Interview Questions

### Q1: "Walk me through what happens when you type a URL in the browser."

**Model answer:**
"The browser first checks its DNS cache for the domain's IP address. If not cached, it queries the DNS resolver, which recursively queries root, TLD, and authoritative name servers. Once we have the IP, the browser establishes a TCP connection via the 3-way handshake — SYN, SYN-ACK, ACK. For HTTPS, there's a TLS handshake on top where the client and server agree on encryption and exchange certificates. Then the browser sends an HTTP GET request, the server processes it and returns an HTTP response with the HTML. The browser parses the HTML and makes additional requests for CSS, JavaScript, and images. With HTTP/2, these can be multiplexed on a single connection."

### Q2: "Why do we use TCP for most web traffic instead of UDP?"

**Model answer:**
"TCP provides reliable, ordered delivery which is essential for web pages, APIs, and file transfers where you need every byte to arrive correctly. UDP is faster because it skips connection setup and doesn't retransmit lost packets, which makes it better for real-time applications like video calls or gaming where low latency matters more than perfect delivery. In system design, I'd use TCP for almost everything — APIs, database connections, message queue clients — and only consider UDP for real-time features like video streaming or live location tracking."

### Q3: "How does DNS affect system design?"

**Model answer:**
"DNS is critical for several system design components. First, DNS-based load balancing lets us distribute traffic across multiple server IPs using round-robin DNS or GeoDNS for routing users to the nearest datacenter. Second, CDNs use DNS to redirect users to the closest edge server — when you query a CDN domain, the DNS response points to a nearby POP. Third, DNS TTL affects how quickly we can failover — a 5-minute TTL means during an outage, some users will still hit the old IP for up to 5 minutes. For critical systems, we keep TTLs low (30-60 seconds) at the cost of more DNS queries."

### Q4: "What's the difference between HTTP/1.1 and HTTP/2, and why does it matter?"

**Model answer:**
"The biggest difference is multiplexing. HTTP/1.1 suffers from head-of-line blocking — you send one request and wait for the response before sending the next on that connection. Browsers work around this by opening 6-8 parallel connections, but that wastes resources. HTTP/2 multiplexes many requests and responses on a single TCP connection, streams them in parallel, and uses header compression to reduce overhead. For system design, HTTP/2 matters when designing APIs with many parallel requests, like a dashboard loading 20 widgets simultaneously, or a mobile app that needs to be efficient with connections."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| "DNS just translates domains to IPs" | "DNS is a distributed, hierarchical system with caching at multiple levels. It's also used for load balancing (round-robin DNS, GeoDNS) and CDN routing." |
| "TCP is slow, we should use UDP" | "TCP has connection overhead but provides reliability. For system design, use TCP unless you specifically need low-latency best-effort delivery (video, gaming)." |
| "HTTPS is for security, it doesn't affect design" | "TLS adds latency (handshake RTTs), requires certificate management, and is mandatory for HTTP/2. Design should account for TLS termination (at the load balancer or CDN)." |
| Spending 5 minutes explaining TCP internals | Keep networking explanations to 2-3 sentences. Interviewers want to see you USE the concepts, not lecture on them. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                  CHAPTER 0 CHEAT SHEET                           │
│              Internet, DNS, TCP, HTTP                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DNS RESOLUTION:                                                 │
│  Browser cache → OS cache → Resolver → Root → TLD → Auth DNS    │
│  TTL controls caching duration. Low TTL = faster failover.       │
│                                                                  │
│  TCP 3-WAY HANDSHAKE: SYN → SYN-ACK → ACK                      │
│  TCP = reliable, ordered. UDP = fast, unreliable.                │
│                                                                  │
│  TLS HANDSHAKE (TLS 1.3): 1 RTT. Adds ~10-30ms.                │
│  Terminate TLS at the load balancer or CDN.                      │
│                                                                  │
│  HTTP METHODS:                                                   │
│  GET (read), POST (create), PUT (replace), PATCH (update),       │
│  DELETE (remove)                                                 │
│  Idempotent: GET, PUT, DELETE. Non-idempotent: POST, PATCH.     │
│                                                                  │
│  STATUS CODES:                                                   │
│  2xx success, 3xx redirect, 4xx client error, 5xx server error   │
│  Key: 200, 201, 301, 400, 401, 403, 404, 429, 500, 503         │
│                                                                  │
│  HTTP/2: Multiplexing, header compression, binary protocol       │
│  HTTP/3: UDP-based (QUIC), 0-RTT, no head-of-line blocking     │
│                                                                  │
│  SYSTEM DESIGN RELEVANCE:                                        │
│  • DNS → load balancing, CDN routing, failover                   │
│  • TCP → API calls, DB connections, reliability                  │
│  • UDP → video streaming, real-time features                     │
│  • HTTPS → always assume TLS, terminate at LB/CDN              │
│  • HTTP/2 → internal microservice communication                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 4 steps of DNS resolution? | Browser cache → Recursive resolver → Root/TLD name servers → Authoritative name server |
| 2 | What does TTL mean in DNS? | Time To Live — how long a DNS record can be cached before needing re-resolution |
| 3 | What are the 3 steps of the TCP handshake? | SYN (client → server), SYN-ACK (server → client), ACK (client → server) |
| 4 | When would you choose UDP over TCP? | When low latency matters more than reliable delivery: video streaming, gaming, VoIP, DNS queries |
| 5 | What does "idempotent" mean for HTTP methods? | Calling the method multiple times has the same effect as calling it once. GET, PUT, DELETE are idempotent; POST is not. |
| 6 | What HTTP status code indicates rate limiting? | 429 Too Many Requests |
| 7 | What's the key advantage of HTTP/2 over HTTP/1.1? | Multiplexing — multiple requests/responses can be in-flight simultaneously on a single TCP connection |
| 8 | How does HTTP/3 differ from HTTP/2? | HTTP/3 uses QUIC (UDP-based) instead of TCP, eliminating TCP-level head-of-line blocking and enabling 0-RTT connections |
| 9 | Where should TLS be terminated in a system design? | At the load balancer or CDN edge server, so internal traffic doesn't need encryption overhead |
| 10 | How is DNS used for load balancing? | Round-robin DNS returns different IPs for the same domain, distributing traffic. GeoDNS returns different IPs based on the client's geographic location. |

---

[← Phase 0 Overview](./README.md) · [Next: APIs — REST, GraphQL, gRPC →](./chapter-01-apis-rest-graphql-grpc.md)
