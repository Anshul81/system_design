# Chapter 22: Consistent Hashing

> **Phase 5 — Distributed Systems Concepts** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium

[← Previous: Consensus & Leader Election](./chapter-21-consensus-leader-election.md) · [Next: Distributed Transactions →](./chapter-23-distributed-transactions.md)

---

## Learning Objectives

- ✅ Explain why naive modular hashing fails when nodes change
- ✅ Describe consistent hashing with a hash ring and virtual nodes
- ✅ Apply consistent hashing to cache clusters and database sharding
- ✅ Understand virtual nodes and why they're needed for load balance

---

## Introduction — Why This Matters

**Interview relevance:** "How do you distribute data across cache servers?" and "How do you handle adding/removing shards?" are common questions. Consistent hashing is the answer — it minimizes data movement when the number of nodes changes. It's used by DynamoDB, Cassandra, Memcached, and CDN routing. Understanding it shows you can think beyond simple `hash % N`.

---

## Core Concepts

### 1. The Problem with Modular Hashing

```
hash(key) % N = server

With 3 servers:
  hash("A") % 3 = 0 → Server 0
  hash("B") % 3 = 1 → Server 1
  hash("C") % 3 = 2 → Server 2
  hash("D") % 3 = 0 → Server 0

Add a 4th server (N changes from 3 to 4):
  hash("A") % 4 = 2 → Server 2 ← MOVED!
  hash("B") % 4 = 1 → Server 1 (stayed)
  hash("C") % 4 = 0 → Server 0 ← MOVED!
  hash("D") % 4 = 2 → Server 2 ← MOVED!

75% of keys remapped! With millions of keys,
this means a massive cache miss storm (thundering herd).
```

### 2. Consistent Hashing — The Hash Ring

```
Hash space: 0 to 2^32 (mapped to a circle)

            0
           ╱ ╲
         ╱     ╲
       S1        ╲
      ╱            S2
     │              │
     │   HASH RING  │
     │              │
      ╲            S3
       ╲          ╱
         ╲      ╱
           ╲  ╱
          2^32/2

Nodes are placed on the ring at hash(node_id).
Keys are placed on the ring at hash(key).
A key belongs to the FIRST node encountered
going CLOCKWISE from the key's position.

Adding a new node (S4):

            0
           ╱ ╲
         ╱     ╲
       S1       S4   ← New node
      ╱            S2
     │              │
     │              │
      ╲            S3
       ╲          ╱
         ╲      ╱
           ╲  ╱

Only keys between S1 and S4 are remapped to S4.
All other keys stay on their current nodes.
On average: only K/N keys move (K = total keys, N = nodes)
With 4 nodes and 1M keys: ~250K keys move (vs ~750K with modular)
```

### 3. Virtual Nodes

Problem: With few physical nodes, data distribution can be very uneven.

```
WITHOUT virtual nodes (3 servers, uneven):

            0
           ╱ ╲         S1 handles 60% of the ring
         ╱     ╲        S2 handles 10%
       S1        ╲      S3 handles 30%
      ╱            S2    → Very unbalanced!
     │              │
      ╲            S3
         ╲      ╱
           ╲  ╱

WITH virtual nodes (3 servers, each has 4 virtual nodes):

            0
           ╱ ╲
       S1_1   S2_3
      ╱          ╲       Each server appears 4 times
    S3_2         S1_3    on the ring at different positions.
     │              │    
    S2_1         S3_4    Result: Much more even distribution!
      ╲          ╱
       S1_4   S3_1       In practice: use 100-200 virtual nodes
         S2_2            per physical node.
```

**With 100+ virtual nodes per physical node, each node handles approximately 1/N of the keys (nearly perfect balance).**

### 4. Applications of Consistent Hashing

| System | How It Uses Consistent Hashing |
|--------|-------------------------------|
| **DynamoDB** | Distributes data across storage nodes |
| **Cassandra** | Assigns data to nodes in the cluster |
| **Memcached** | Distributes cache keys across servers |
| **Akamai CDN** | Routes requests to edge servers |
| **Discord** | Routes messages to the right server |

---

## Common Interview Questions

### Q1: "How would you distribute cache keys across a Redis cluster?"

**Model answer:**
"I'd use consistent hashing with virtual nodes. Each Redis instance is mapped to 100-200 positions on a hash ring. When a cache key needs to be stored, we hash the key and find the first Redis instance clockwise on the ring. When we add a new Redis instance, only K/N keys need to be remapped (where K is total keys and N is the number of nodes), minimizing cache misses. Without consistent hashing, adding a server would invalidate most cached keys and cause a thundering herd on the database."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 22 CHEAT SHEET                        │
│                  Consistent Hashing                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PROBLEM: hash(key) % N → ~75% keys move when N changes        │
│  SOLUTION: Hash ring → only K/N keys move when nodes change     │
│                                                                  │
│  HASH RING: Nodes + keys mapped to circular hash space          │
│  Key → first node clockwise on the ring                         │
│                                                                  │
│  VIRTUAL NODES: Each physical node maps to 100-200 positions    │
│  → Even distribution across physical nodes                      │
│                                                                  │
│  USED BY: DynamoDB, Cassandra, Memcached, CDNs                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | Why does `hash(key) % N` fail when N changes? | Almost all keys remap to different servers, causing massive cache misses or data migration. |
| 2 | How does consistent hashing work? | Nodes and keys are placed on a circular hash space. A key is assigned to the first node clockwise from its position. |
| 3 | How many keys move when adding a node with consistent hashing? | K/N on average (K = total keys, N = new total nodes). Much better than ~75% with modular hashing. |
| 4 | What are virtual nodes? | Multiple hash positions per physical node (100-200 typically). Ensures even key distribution across nodes. |
| 5 | Name 3 systems that use consistent hashing. | DynamoDB, Cassandra, Memcached (also Akamai CDN, Discord) |

---

[← Previous: Consensus & Leader Election](./chapter-21-consensus-leader-election.md) · [Next: Distributed Transactions →](./chapter-23-distributed-transactions.md)
