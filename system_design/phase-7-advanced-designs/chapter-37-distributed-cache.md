# Chapter 37: Design a Distributed Cache

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: E-Commerce](./chapter-36-ecommerce.md) · [Next: Collaborative Editing →](./chapter-38-collaborative-editing.md)

---

## Why This Problem Matters

This question asks you to design Redis/Memcached from scratch. It tests consistent hashing, eviction policies, replication, and data partitioning — all in one problem.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Operations** | GET, PUT, DELETE |
| **Latency** | < 1ms for cache hits |
| **Throughput** | 1M+ ops/sec per node |
| **Size** | 100s of GB to TBs across cluster |
| **Availability** | 99.99% |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED CACHE CLUSTER                   │
│                                                              │
│  Client Library (consistent hashing):                        │
│    key = "user:123"                                          │
│    hash(key) → node on hash ring → Cache Node 3             │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │ Cache Node 1│  │ Cache Node 2│  │ Cache Node 3│             │
│  │ (Primary)   │  │ (Primary)   │  │ (Primary)   │             │
│  │ 64 GB RAM   │  │ 64 GB RAM   │  │ 64 GB RAM   │             │
│  │             │  │             │  │ user:123 ──▶│             │
│  ├────────────┤  ├────────────┤  ├────────────┤             │
│  │ Hash table  │  │ Hash table  │  │ Hash table  │             │
│  │ (in-memory) │  │ (in-memory) │  │ (in-memory) │             │
│  │             │  │             │  │             │             │
│  │ LRU eviction│  │ LRU eviction│  │ LRU eviction│             │
│  │ chain       │  │ chain       │  │ chain       │             │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘             │
│         │               │               │                     │
│  ┌──────▼─────┐  ┌──────▼─────┐  ┌──────▼─────┐             │
│  │ Replica     │  │ Replica     │  │ Replica     │             │
│  └────────────┘  └────────────┘  └────────────┘             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Key Components

**1. Data Partitioning (Consistent Hashing):**
```
Hash ring with virtual nodes (150 per physical node).
Client library computes hash(key) → finds node on ring.
Adding/removing a node: only ~1/N keys remigrate.
```

**2. In-Memory Hash Table:**
```
Each node: hash table (key → value + metadata)

struct CacheEntry {
    key:        string
    value:      bytes
    ttl:        timestamp    // expiration
    created_at: timestamp
    access_at:  timestamp    // for LRU
    prev:       *CacheEntry  // doubly-linked list for LRU
    next:       *CacheEntry
}

Operations:
  GET: O(1) hash lookup → return value, update LRU position
  PUT: O(1) hash insert → add to LRU head → evict tail if full
  DEL: O(1) hash delete → remove from LRU chain
```

**3. Eviction (LRU with Hash Map + Doubly Linked List):**
```
LRU IMPLEMENTATION:

Hash Map: key → pointer to linked list node (O(1) lookup)
Doubly Linked List: ordered by access time (head = most recent)

GET "key_A":
  Hash lookup → find node → move to HEAD of list → return value

PUT "key_X" (cache full):
  Evict TAIL of list (least recently used)
  Insert "key_X" at HEAD
  Add to hash map

  HEAD ←→ [key_C] ←→ [key_A] ←→ [key_B] ←→ TAIL
                                      ↑
                                 evict this
```

**4. Replication:**
```
Async replication to 1-2 replicas:
  Primary handles all reads and writes.
  Replicas are for failover (not read scaling in cache).
  
If primary dies:
  Promote replica → becomes new primary.
  Clients re-hash to find new node on ring.
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Partitioning | Consistent hashing (client-side) | Minimal data movement on node changes. Client routes directly — no proxy bottleneck. |
| Eviction | LRU (hash map + doubly linked list) | O(1) for all operations. Best general-purpose policy. |
| Replication | Async to replicas | Cache data is ephemeral — async is fast. Losing a few entries on failover is acceptable. |
| Thread safety | Single-threaded event loop (like Redis) | No lock contention. All ops are in-memory and fast. Simpler correctness. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How do you implement LRU cache with O(1) operations? | Hash map (O(1) key lookup) + doubly linked list (O(1) move to head, evict tail). |
| 2 | How are keys distributed across cache nodes? | Consistent hashing with virtual nodes. Client library hashes the key to find the responsible node on the ring. |
| 3 | Why is Redis single-threaded? | In-memory operations are sub-microsecond. Lock contention from multi-threading would be more expensive than the operations themselves. |
| 4 | What happens when a cache node fails? | Replica promoted to primary. Clients may get cache misses for keys that weren't replicated. They fall through to the database and repopulate. |

---

[← Previous: E-Commerce](./chapter-36-ecommerce.md) · [Next: Collaborative Editing →](./chapter-38-collaborative-editing.md)
