# Chapter 21: Consensus & Leader Election

> **Phase 5 — Distributed Systems Concepts** · Estimated Time: 2.5 hours · Difficulty: 🔴 Hard

[← Previous: Unique ID Generation](./chapter-20-unique-id-generation.md) · [Next: Consistent Hashing →](./chapter-22-consistent-hashing.md)

---

## Learning Objectives

- ✅ Explain why consensus is needed in distributed systems
- ✅ Understand the Raft algorithm at a high level (leader election, log replication)
- ✅ Know how ZooKeeper and etcd provide coordination primitives
- ✅ Discuss leader election patterns and split-brain prevention

---

## Introduction — Why This Matters

**Interview relevance:** You don't need to implement Raft, but you need to know WHEN and WHY consensus matters. When the interviewer asks "What if the primary database fails?", "How do you ensure only one consumer processes a message?", or "How do you elect a new leader?" — you need to reference consensus. ZooKeeper and etcd are commonly mentioned as coordination services.

---

## Core Concepts

### 1. The Consensus Problem

```
Problem: Multiple nodes need to agree on a single value
(who is the leader? what is the committed state?)

Without consensus:
  Node A thinks it's the leader → processes writes
  Node B also thinks it's the leader → processes writes
  → SPLIT BRAIN: conflicting data, data corruption!

With consensus (e.g., Raft):
  Nodes vote → majority agrees → Node A is leader
  If Node A dies → new election → Node C becomes leader
  Only ONE leader at any time (guaranteed by majority quorum)
```

### 2. Raft Algorithm (Simplified)

```
RAFT — Three roles:
  Leader:    Handles all client requests, replicates log
  Follower:  Passive, receives log entries from leader
  Candidate: Follower that starts an election

STATE MACHINE:
  Follower ──(timeout, no heartbeat)──▶ Candidate
  Candidate ──(wins election)──▶ Leader
  Candidate ──(loses/timeout)──▶ Follower or Candidate
  Leader ──(discovers higher term)──▶ Follower

LEADER ELECTION:
  1. Follower hasn't heard from leader (timeout)
  2. Becomes Candidate, increments term, votes for self
  3. Requests votes from other nodes
  4. If receives majority of votes → becomes Leader
  5. If another leader found → becomes Follower
  6. Leader sends heartbeats to prevent new elections

LOG REPLICATION:
  1. Client sends write to Leader
  2. Leader appends to its log
  3. Leader replicates entry to all Followers
  4. When majority acknowledge → entry is COMMITTED
  5. Leader responds to client: "write successful"

  ┌────────┐     ┌────────┐     ┌────────┐
  │Leader   │────▶│Follower│     │Follower│
  │log:[A,B]│────▶│log:[A,B]│    │log:[A] │ ← lagging
  └────────┘     └────────┘     └────────┘
                                (will catch up)

  Entry B committed when Leader + 1 Follower have it
  (majority of 3 nodes = 2)
```

### 3. ZooKeeper & etcd

| Service | Used By | Provides |
|---------|---------|----------|
| **ZooKeeper** | Kafka, HBase, Hadoop | Leader election, config management, distributed locks, service discovery |
| **etcd** | Kubernetes | Key-value config store, leader election, service discovery, distributed locks |

```
Common coordination patterns:

1. LEADER ELECTION:
   Multiple instances compete to create an ephemeral node in ZooKeeper.
   Only one succeeds → becomes leader.
   If leader dies → ephemeral node disappears → new election.

2. DISTRIBUTED LOCK:
   Create ephemeral sequential node → lowest number gets the lock.
   When done → delete node → next gets the lock.

3. SERVICE DISCOVERY:
   Services register themselves in ZooKeeper/etcd.
   Clients query to find available service instances.
```

### 4. Split-Brain Prevention

```
Split-Brain Scenario:

Network partition separates nodes:

  ┌──────────────────┐     ╳     ┌──────────────────┐
  │ Node A (was leader)│   NETWORK  │ Node C            │
  │ Node B             │  PARTITION │ Node D            │
  │                    │     ╳     │ Node E            │
  └──────────────────┘           └──────────────────┘
  
  Partition 1: Nodes A, B (2/5 = minority)
  Partition 2: Nodes C, D, E (3/5 = majority)

  With quorum-based consensus:
  Partition 1 CANNOT commit (needs 3 of 5 nodes)
  Partition 2 CAN elect a new leader and commit
  → No split-brain! Only the majority partition operates.

  Key insight: Quorum requires (N/2 + 1) nodes to agree.
  Use ODD number of nodes (3, 5, 7) for clear majority.
```

---

## Common Interview Questions

### Q1: "How does leader election work?"

**Model answer:**
"I'd use a coordination service like ZooKeeper or etcd. Multiple instances attempt to create an ephemeral leader node. Only one succeeds and becomes the leader. All others watch for changes. If the leader dies, its ephemeral node disappears, triggering a new election. The key guarantee is that the consensus protocol (Raft in etcd, ZAB in ZooKeeper) ensures exactly one leader at a time, preventing split-brain through majority quorum."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 21 CHEAT SHEET                        │
│                  Consensus & Leader Election                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CONSENSUS: Nodes agree on a single value/leader                │
│  RAFT: Leader election + log replication via majority quorum    │
│  QUORUM: (N/2 + 1) nodes must agree. Use odd N (3, 5, 7).     │
│                                                                  │
│  SPLIT-BRAIN: Two leaders → data corruption. Prevented by      │
│  requiring majority quorum for any operation.                   │
│                                                                  │
│  COORDINATION SERVICES:                                          │
│  ZooKeeper: Kafka, HBase (leader election, locks, config)      │
│  etcd: Kubernetes (config store, service discovery)             │
│                                                                  │
│  INTERVIEW: "We'd use ZooKeeper/etcd for leader election"      │
│  Don't implement Raft — use existing tools.                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is split-brain? | Two nodes both think they're the leader, processing conflicting writes. Prevented by majority quorum. |
| 2 | What is the quorum formula? | (N/2 + 1) — more than half the nodes must agree. With 5 nodes, quorum = 3. |
| 3 | Why use odd numbers of nodes for consensus? | Odd numbers provide a clear majority. Even numbers waste a node (4 nodes still needs 3 for quorum, same as 3 nodes). |
| 4 | What is Raft? | A consensus algorithm with 3 roles (leader, follower, candidate). Leader handles writes, replicates to followers, commits when majority acknowledges. |
| 5 | ZooKeeper vs etcd? | Both provide coordination (leader election, locks, config). ZooKeeper: Java, used by Kafka. etcd: Go, used by Kubernetes. |

---

[← Previous: Unique ID Generation](./chapter-20-unique-id-generation.md) · [Next: Consistent Hashing →](./chapter-22-consistent-hashing.md)
