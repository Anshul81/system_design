# Chapter 38: Design Google Docs / Collaborative Editing

> **Phase 7 — Advanced Design Problems** · Estimated Time: 3.5 hours · Difficulty: 🔴 Hard · Type: 🏗️ DESIGN

[← Previous: Distributed Cache](./chapter-37-distributed-cache.md) · [Next: Metrics & Logging Pipeline →](./chapter-39-metrics-logging-pipeline.md)

---

## Why This Problem Matters

Collaborative editing introduces Operational Transformation (OT) or CRDTs — algorithms for merging concurrent edits without conflicts. This is the hardest design problem in this course. Understanding the high-level approach is sufficient for most interviews.

---

## Requirements

| Requirement | Target |
|-------------|--------|
| **Concurrent editors** | Up to 50 per document |
| **Latency** | < 100ms for edits to appear to other users |
| **Conflict resolution** | Automatic (no merge conflicts) |
| **Offline support** | Edit offline, sync when reconnected |

---

## Core Challenge: Concurrent Edits

```
THE PROBLEM:
  User A (position 3): INSERT "X" at position 3
  User B (position 5): DELETE character at position 5

  Both edits sent simultaneously. If applied naively:
  User A's insert shifts all positions after 3
  User B's delete at position 5 is now wrong (should be 6)

  → Text diverges between users!

SOLUTION 1: Operational Transformation (OT)
  Transform operations against each other:
  A: INSERT("X", pos=3)
  B: DELETE(pos=5) → transform against A's insert → DELETE(pos=6)

  Server maintains a single operation history.
  Each client sends operations. Server transforms and broadcasts.
  Guarantees all clients converge to same document state.

  Used by: Google Docs

SOLUTION 2: CRDTs (Conflict-free Replicated Data Types)
  Each character has a unique, globally-ordered ID.
  Operations reference character IDs, not positions.
  Merging is automatic — no transformation needed.
  
  More complex data structure, but works peer-to-peer (no central server).
  Used by: Figma, some newer collaborative editors
```

### Architecture

```
┌────────┐   WebSocket    ┌─────────────────┐
│Editor A │══════════════▶│  Collaboration    │
│         │◀══════════════│  Server           │
└────────┘               │                   │
                          │  1. Receive op    │
┌────────┐   WebSocket    │  2. Transform     │
│Editor B │══════════════▶│  3. Apply to doc  │
│         │◀══════════════│  4. Broadcast to  │
└────────┘               │     all clients   │
                          │                   │
┌────────┐   WebSocket    │  Doc state stored │
│Editor C │══════════════▶│  in DB + Redis    │
│         │◀══════════════│                   │
└────────┘               └─────────┬─────────┘
                                    │
                          ┌─────────▼─────────┐
                          │  Document Store     │
                          │  (operations log +  │
                          │   document snapshot) │
                          └───────────────────┘
```

### OT Flow (Simplified)

```
Document: "ABCDEF"
Both users start from revision 5.

User A: INSERT "X" at pos 2 → "ABXCDEF" (sends op to server)
User B: DELETE pos 4 → "ABCDF" (sends op to server)

Server receives A's op first:
  Apply: "ABCDEF" → "ABXCDEF" (revision 6)
  
Server receives B's op (still based on revision 5):
  Transform B's op against A's op:
    B's DELETE(pos=4) + A's INSERT(pos=2) → DELETE(pos=5)
    (A's insert shifted B's position by 1)
  Apply: "ABXCDEF" → "ABXCDF" (revision 7)

Server broadcasts transformed ops to all clients.
All clients converge to "ABXCDF" ✅
```

### Storage Strategy

```
Documents stored as:
  1. Operations log (all edits, ordered by revision)
  2. Periodic snapshots (full document at revision N)

To load a document:
  1. Find latest snapshot (e.g., revision 100)
  2. Replay operations 101-150 on top of snapshot
  3. Result: current document at revision 150

Snapshots taken every ~100 operations to bound replay time.

Version history: users can view/restore any previous version
by replaying operations up to that revision.
```

---

## Key Trade-offs

| Decision | Choice | Why |
|----------|--------|-----|
| Conflict resolution | OT (centralized server) | Simpler than CRDTs. Central server makes transformation deterministic. Google Docs approach. |
| Transport | WebSocket | Low-latency bidirectional for real-time edits. Each keystroke sent as an operation. |
| Storage | Operation log + snapshots | Full edit history for version control. Snapshots prevent long replay chains. |
| Cursor sharing | Broadcast cursor positions to all editors | Shows who's editing where. Sent at lower frequency than edits (every 500ms). |

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is Operational Transformation (OT)? | An algorithm that transforms concurrent edit operations against each other so all clients converge to the same document state, regardless of operation order. |
| 2 | OT vs CRDT? | OT: central server transforms ops (Google Docs). CRDT: operations are inherently mergeable without transformation (Figma). OT is simpler with a server. CRDT works peer-to-peer. |
| 3 | How are documents stored? | Operations log (every edit, ordered by revision) + periodic snapshots. Load = latest snapshot + replay recent ops. |
| 4 | Why WebSocket for collaborative editing? | Real-time bidirectional needed. Each keystroke is an operation sent immediately and broadcast to all other editors. < 100ms latency expected. |

---

[← Previous: Distributed Cache](./chapter-37-distributed-cache.md) · [Next: Metrics & Logging Pipeline →](./chapter-39-metrics-logging-pipeline.md)
