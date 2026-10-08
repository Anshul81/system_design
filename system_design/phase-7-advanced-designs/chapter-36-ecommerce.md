# Chapter 36: Design E-Commerce (Amazon)

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3.5 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Proximity Service](./chapter-35-proximity-service.md) · [Next: Distributed Cache →](./chapter-37-distributed-cache.md)

---

## Why This Problem Matters

E-commerce combines product catalog (search + browse), shopping cart, checkout with payment, inventory management, and order processing. It's the ultimate "microservices" interview question because each concern maps to a distinct service.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Products** | 500M product listings |
| **DAU** | 200M |
| **Orders/day** | 10M |
| **Peak** | Flash sales (100x normal traffic) |
| **Inventory** | No overselling |

---

## Architecture

```
┌────────┐    ┌──────────┐    ┌──────────────────────────────────────┐
│ Client  │───▶│ API       │    │              SERVICES                │
│         │    │ Gateway   │───▶│                                      │
└────────┘    │ (auth,    │    │  ┌──────────┐ ┌──────────┐          │
              │  rate     │    │  │ Product   │ │ Search   │          │
              │  limit)   │    │  │ Catalog   │ │ Service  │          │
              └──────────┘    │  │ Service   │ │ (ES)     │          │
                               │  └──────────┘ └──────────┘          │
                               │                                      │
                               │  ┌──────────┐ ┌──────────┐          │
                               │  │ Cart      │ │ Order    │          │
                               │  │ Service   │ │ Service  │          │
                               │  │ (Redis)   │ │          │          │
                               │  └──────────┘ └──────────┘          │
                               │                                      │
                               │  ┌──────────┐ ┌──────────┐          │
                               │  │ Inventory │ │ Payment  │          │
                               │  │ Service   │ │ Service  │          │
                               │  └──────────┘ └──────────┘          │
                               └──────────────────────────────────────┘
```

### Key Design Decisions

**1. Product Catalog:**
```
Products have varying attributes:
  Electronics: brand, specs, warranty
  Clothing: size, color, material
  Books: author, ISBN, pages

→ Use PostgreSQL for core product data + MongoDB for flexible attributes
→ Or: PostgreSQL with JSONB column for variable attributes

Search: Elasticsearch with CDC pipeline from product DB
  Faceted search: category, price range, brand, rating, etc.
```

**2. Shopping Cart:**
```
Redis for cart (fast, ephemeral for guests):
  Key: cart:{user_id} or cart:{session_id}
  Value: Hash of {product_id: quantity}
  TTL: 7 days for guests, no TTL for logged-in users

When user logs in: merge guest cart with user cart
When user checks out: cart → order (atomic)
```

**3. Inventory & Checkout (Critical Path):**
```
CHECKOUT FLOW (Saga pattern):

1. Cart Service: Validate cart items still available
2. Inventory Service: RESERVE items (decrement available_count)
   → Uses SELECT ... FOR UPDATE (pessimistic lock)
   → If any item unavailable → fail entire checkout
3. Payment Service: Charge user
   → If payment fails → Inventory compensates (un-reserve)
4. Order Service: Create order record (CONFIRMED)
5. Notification: Send confirmation email

INVENTORY TABLE:
  product_id | total_stock | reserved | available
  P123       | 100         | 5        | 95

  available = total_stock - reserved
  Reserve: UPDATE inventory SET reserved = reserved + 1
           WHERE product_id = 'P123' AND available > 0
```

**4. Flash Sales (Traffic Spike):**
```
Normal: 10M orders/day → 115 orders/sec
Flash sale: 100x → 11,500 orders/sec for specific products

Strategies:
  • Virtual queue (like Ticketmaster) for hot products
  • Pre-warm CDN with product pages
  • Rate limiting per user
  • Separate "hot product" inventory service (Redis-based)
    with pre-loaded inventory counts
  • Auto-scale app servers based on traffic prediction
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Product catalog | PostgreSQL + Elasticsearch | PG for ACID, ES for search. CDC keeps them in sync. |
| Cart storage | Redis | Fast, supports TTL for guest carts, handles high read/write |
| Inventory control | Pessimistic locking | Flash sales = extreme contention on same rows. Pessimistic lock prevents overselling. |
| Checkout | Saga pattern | Cross-service transaction: inventory → payment → order. Compensating actions on failure. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you prevent overselling during flash sales? | Pessimistic locking on inventory rows (SELECT FOR UPDATE) + Redis pre-loaded counts for hot products. Virtual queue for traffic management. |
| 2 | Where do you store the shopping cart? | Redis. Hash of product_id → quantity. TTL for guest carts. Fast read/write for frequent cart operations. |
| 3 | How does checkout work across microservices? | Saga pattern: Reserve inventory → Charge payment → Create order. If payment fails, compensate by unreserving inventory. |
| 4 | How do you handle variable product attributes? | PostgreSQL with JSONB column for flexible attributes, or hybrid PostgreSQL (core) + MongoDB (attributes). |

---

[← Previous: Proximity Service](./chapter-35-proximity-service.md) · [Next: Distributed Cache →](./chapter-37-distributed-cache.md)
