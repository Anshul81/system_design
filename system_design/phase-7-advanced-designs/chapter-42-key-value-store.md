# Chapter 42: Design a Key-Value Store

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Ad Click Aggregation](./chapter-41-ad-click-aggregation.md) · [Next: Leaderboard →](./chapter-43-leaderboard.md)

---

## Why This Problem Matters

Designing a key-value store (like DynamoDB or etcd) tests your knowledge of partitioning, replication, consistency models, conflict resolution, and failure handling all in one problem. It's the ultimate "distributed systems fundamentals" interview question — every concept from Phase 5 appears here.

---

## Step 1: Requirements

### Functional
| Feature | Description |
|---------|-------------|
| **PUT(key, value)** | Store a key-value pair |
| **GET(key)** | Retrieve value by key |
| **DELETE(key)** | Remove a key-value pair |
| **Tunable consistency** | Per-request consistency level (ONE, QUORUM, ALL) |

### Non-Functional
| Requirement | Target |
|-------------|--------|
| **Key size** | Up to 256 bytes |
| **Value size** | Up to 1 MB |
| **Latency** | < 10ms for single key ops (p99) |
| **Throughput** | 500K+ ops/sec across cluster |
| **Availability** | 99.99% (AP system, favoring availability) |
| **Durability** | Data replicated to 3 nodes |

---

## Step 2: Estimation

```
STORAGE:
  100B key-value pairs × 10 KB avg value = 1 PB total
  With 3x replication: 3 PB
  Per node (10 TB SSD): need ~300 nodes

THROUGHPUT:
  500K ops/sec across 300 nodes = ~1,700 ops/sec per node
  (well within SSD capability)

NETWORK:
  500K ops/sec × 10 KB avg = 5 GB/sec total cluster bandwidth
```

---

## Step 3: High-Level Design

### Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED KEY-VALUE STORE                    │
│                                                                  │
│  Client Library:                                                 │
│    1. Hash key → find coordinator node on ring                  │
│    2. Send request to coordinator                                │
│    3. Coordinator replicates to N-1 other nodes                 │
│                                                                  │
│         Consistent Hash Ring (virtual nodes)                     │
│                                                                  │
│              ┌──── Node A ────┐                                  │
│         ┌────┤               ├────┐                              │
│   Node F│    │   Key Range   │    │Node B                        │
│         │    │  [hash_1,     │    │                               │
│         │    │   hash_2)     │    │                               │
│   Node E│    └───────────────┘    │Node C                        │
│         └────┐               ┌────┘                              │
│              └──── Node D ────┘                                  │
│                                                                  │
│  Each key replicated to N=3 consecutive nodes on ring           │
│  Coordinator = first node clockwise from hash(key)              │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### API Design

```
PUT /api/v1/kv/{key}
  Headers: X-Consistency: QUORUM       // ONE, QUORUM, ALL
  Body:    { "value": "<base64_data>",
             "ttl": 3600 }              // optional TTL in seconds
  Response: { "version": 5, "timestamp": 1695600000 }

GET /api/v1/kv/{key}
  Headers: X-Consistency: QUORUM
  Response: { "key": "user:123",
              "value": "<base64_data>",
              "version": 5,
              "timestamp": 1695600000 }

DELETE /api/v1/kv/{key}
  Response: HTTP 204 No Content
```

---

## Step 4: Detailed Design

### Deep Dive 1: Data Partitioning (Consistent Hashing)

```
CONSISTENT HASHING WITH VIRTUAL NODES:

Physical nodes: A, B, C, D, E, F
Virtual nodes per physical: 150 (spread evenly on ring)

Hash ring: 0 ────────────────────────────── 2^128
           |  A1  B1  C1  A2  D1  B2 ... |

When key "user:123" is written:
  1. hash("user:123") = position on ring
  2. Walk clockwise → first 3 DISTINCT physical nodes = replicas
     e.g., nodes B, C, D hold replicas
  3. Coordinator = B (first node clockwise)

ADDING A NODE (G):
  Place G's virtual nodes on ring
  G takes over key ranges from its neighbors
  Only ~1/N of keys need to migrate

REMOVING A NODE (C fails):
  C's key ranges handled by next node clockwise
  C's replicas exist on 2 other nodes → data not lost
  Hinted handoff ensures C's writes are queued for when it recovers

WHY VIRTUAL NODES?
  Without: if nodes have unequal capacity, some are overloaded
  With: 150 virtual nodes per physical → even distribution
  Can give more virtual nodes to beefier machines
```

### Deep Dive 2: Replication & Consistency

```
REPLICATION FACTOR: N = 3

For each key, 3 nodes store a copy.
Write/read consistency is TUNABLE:

  W = number of nodes that must ACK a write
  R = number of nodes that must respond to a read
  
  W + R > N → guarantees strong consistency (read sees latest write)

PRESETS:
  ┌──────────┬─────┬─────┬────────────────────────────┐
  │ Level     │  W  │  R  │ Guarantees                  │
  ├──────────┼─────┼─────┼────────────────────────────┤
  │ ONE       │  1  │  1  │ Fastest, but may read stale │
  │ QUORUM    │  2  │  2  │ Strong consistency (2+2>3) │
  │ ALL       │  3  │  3  │ Strongest, but slowest      │
  │           │     │     │ Any node down → unavailable │
  └──────────┴─────┴─────┴────────────────────────────┘

WRITE FLOW (QUORUM):
  Client → Coordinator (Node B)
  B writes locally + sends to C and D (replicas)
  B waits for 1 more ACK (W=2 total including itself)
  B responds SUCCESS to client
  Third replica (D) writes async

READ FLOW (QUORUM):
  Client → Coordinator (Node B)
  B reads locally + requests from C and D
  B waits for 2 responses (R=2)
  If values differ → return the one with highest version/timestamp
  Trigger read repair on stale replicas (background)

HINTED HANDOFF:
  If Node C is down during a write:
  B writes to itself + a temporary node E (hint for C)
  When C comes back → E sends the hinted data to C
  Ensures writes succeed even during partial failures
```

### Deep Dive 3: Conflict Resolution

```
PROBLEM: Two clients write to the same key concurrently
  Client 1 → Node B: PUT key="X" value="Alice"
  Client 2 → Node C: PUT key="X" value="Bob"
  (during a network partition between B and C)

Both succeed locally. When partition heals → conflict!

SOLUTION 1: Last-Write-Wins (LWW)
  Each write has a timestamp.
  On conflict: keep the one with the latest timestamp.
  Simple but LOSES DATA (Alice's write disappears).
  Used by: Cassandra (default)

SOLUTION 2: Vector Clocks
  Each node maintains a version vector:
    Key "X": { B: 3, C: 1 }  (version 3 on B, version 1 on C)

  Detect conflicts:
    Version { B:3, C:1 } vs { B:1, C:2 }
    Neither dominates the other → CONFLICT

  Resolution: application resolves (e.g., merge both values)
  Client receives BOTH versions → chooses which to keep
  Used by: Amazon DynamoDB (original paper)

SOLUTION 3: CRDTs
  Use conflict-free data types that auto-merge:
    Counter: both increments are applied → correct sum
    Set: union of both additions
  
  No conflicts possible, but limited to specific data structures.

RECOMMENDATION FOR INTERVIEWS:
  Mention LWW as simple default.
  Explain vector clocks for detecting conflicts.
  Discuss trade-off: simplicity vs. data safety.
```

### Deep Dive 4: Storage Engine (Single-Node)

```
STORAGE ENGINE: LSM Tree (Log-Structured Merge Tree)

Why not B-Tree (like MySQL)?
  B-Tree: read-optimized, writes update in-place (random I/O)
  LSM Tree: write-optimized, sequential writes only
  KV stores are often write-heavy → LSM is better

LSM TREE STRUCTURE:
  ┌──────────────────┐
  │ MemTable (RAM)    │  ← writes go here first (sorted)
  │ (Red-Black Tree)  │     when full → flush to disk
  └────────┬─────────┘
           │ flush
  ┌────────▼─────────┐
  │ SSTable L0        │  ← immutable sorted file on disk
  ├──────────────────┤
  │ SSTable L1        │  ← compacted from L0 files
  ├──────────────────┤
  │ SSTable L2        │  ← compacted from L1 files
  └──────────────────┘

WRITE PATH:
  1. Append to WAL (Write-Ahead Log) for durability
  2. Insert into MemTable (in-memory sorted structure)
  3. When MemTable full → flush to disk as sorted SSTable file
  4. Background compaction merges SSTables (removes deleted/overwritten keys)

READ PATH:
  1. Check MemTable (most recent writes)
  2. Check Bloom filters for each SSTable → skip files that don't contain key
  3. Binary search within matching SSTable → return value

BLOOM FILTER for reads:
  Each SSTable has a Bloom filter index
  "Does this SSTable contain key X?"
  → "Definitely not" → skip (saves disk I/O)
  → "Probably yes" → read SSTable
  Reduces disk reads from O(N files) to O(1-2 files)
```

### Deep Dive 5: Failure Detection & Membership

```
GOSSIP PROTOCOL:

Each node periodically (every 1 sec) picks a random node and exchanges:
  "Here's what I know about the cluster"
  
  Node A's view:
    B: alive (heartbeat 2 sec ago)
    C: alive (heartbeat 1 sec ago)
    D: suspected (no heartbeat for 30 sec)

  After exchanging:
    Both nodes have merged view of cluster health

FAILURE DETECTION:
  If no heartbeat from Node D for 30 seconds:
    → D marked as "suspected" (temporary)
    → Hinted handoff activated (write D's data to other nodes)
  
  If no heartbeat for 5 minutes:
    → D marked as "down"
    → Anti-entropy repair: rebuild D's data on replacement node
    → Update hash ring: D's virtual nodes removed

MERKLE TREES (Anti-Entropy):
  Each node maintains a Merkle tree of its data:
    Root hash → hash of left/right subtrees → ... → hash of each key

  To sync two replicas:
    Compare root hashes → if different, compare children
    → Find exactly which keys differ → sync only those keys
  
  Efficient: compares O(log N) hashes instead of O(N) keys
```

---

## Complete Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED KEY-VALUE STORE                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────┐      ┌────────────────────────────────────────┐         │
│  │ Client  │─────▶│ Client Library                         │         │
│  │ (app)   │      │ • Consistent hashing (find coordinator)│         │
│  └────────┘      │ • Retry logic & failover               │         │
│                   └──────────────┬─────────────────────────┘         │
│                                  │                                   │
│                   ┌──────────────▼──────────────┐                    │
│                   │    Coordinator Node          │                    │
│                   │    (first on hash ring)      │                    │
│                   └──────────────┬──────────────┘                    │
│                        ┌────────┼────────┐                           │
│                        ▼        ▼        ▼                           │
│                   ┌────────┐ ┌────────┐ ┌────────┐                  │
│                   │Node A   │ │Node B   │ │Node C   │  N=3 replicas  │
│                   │┌──────┐│ │┌──────┐│ │┌──────┐│                  │
│                   ││MemTbl││ ││MemTbl││ ││MemTbl││                  │
│                   │├──────┤│ │├──────┤│ │├──────┤│                  │
│                   ││WAL   ││ ││WAL   ││ ││WAL   ││                  │
│                   │├──────┤│ │├──────┤│ │├──────┤│                  │
│                   ││SSTable│ ││SSTable│ ││SSTable│                  │
│                   ││(LSM)  │ ││(LSM)  │ ││(LSM)  │                  │
│                   │└──────┘│ │└──────┘│ │└──────┘│                  │
│                   └────────┘ └────────┘ └────────┘                  │
│                                                                      │
│  Cross-cutting:                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Gossip        │  │ Merkle Trees  │  │ Hinted       │              │
│  │ Protocol      │  │ (anti-entropy │  │ Handoff      │              │
│  │ (membership)  │  │  repair)      │  │ (temp writes)│              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Trade-offs

| Decision | Choice | Alternative | Why |
|----------|--------|-------------|-----|
| Consistency model | Tunable (W, R, N) | Fixed strong or eventual | Lets caller choose per-request: fast reads (R=1) or strong reads (R=quorum). |
| Partitioning | Consistent hashing + virtual nodes | Range partitioning | Even distribution, minimal data movement on node changes. Virtual nodes handle heterogeneous hardware. |
| Storage engine | LSM Tree | B-Tree | Write-optimized (sequential I/O). KV stores are write-heavy. Trade-off: reads slightly slower (check multiple SSTables). |
| Conflict resolution | LWW (default) + vector clocks (optional) | Always LWW | LWW is simple and works for most cases. Vector clocks for use cases where data loss is unacceptable. |
| Failure detection | Gossip protocol | Central heartbeat server | Decentralized, no SPOF. Each node learns about failures through epidemic information propagation. |
| Replica sync | Merkle trees | Full data comparison | O(log N) hash comparisons to find divergent keys, instead of O(N) key-by-key comparison. |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | How does consistent hashing work for a KV store? | Key hashed to position on ring. Walk clockwise to find N distinct physical nodes for replication. Virtual nodes ensure even distribution. |
| 2 | What is tunable consistency? | Client chooses W (write ACKs) and R (read responses). W+R > N guarantees strong consistency. W=1,R=1 is fastest but may be stale. |
| 3 | What is a quorum? | Majority of replicas. For N=3: quorum = 2. QUORUM reads/writes (W=2, R=2) guarantee seeing the latest write (2+2 > 3). |
| 4 | How does an LSM Tree work? | Writes go to in-memory MemTable → flush to disk as sorted SSTable → background compaction merges SSTables. WAL for durability. |
| 5 | What is hinted handoff? | When a replica is down, write to a temporary node with a "hint." When the replica recovers, the hint is forwarded. Ensures writes succeed during partial failures. |
| 6 | How do vector clocks detect conflicts? | Each replica tracks a version vector. If two versions where neither dominates the other → conflict detected. Application resolves. |
| 7 | What is a Merkle tree used for? | Anti-entropy repair: compare hash trees between replicas to find exactly which keys differ. O(log N) comparisons instead of O(N). |
| 8 | How does the gossip protocol work? | Each node periodically exchanges cluster state with a random peer. Information propagates like an epidemic. Detects node failures within seconds. |

---

[← Previous: Ad Click Aggregation](./chapter-41-ad-click-aggregation.md) · [Next: Leaderboard →](./chapter-43-leaderboard.md)
