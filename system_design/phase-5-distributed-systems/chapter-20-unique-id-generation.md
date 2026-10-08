# Chapter 20: Unique ID Generation

> **Phase 5 — Distributed Systems Concepts** · Estimated Time: 2 hours · Difficulty: 🟡 Medium

[← Previous: Rate Limiting](./chapter-19-rate-limiting.md) · [Next: Consensus & Leader Election →](./chapter-21-consensus-leader-election.md)

---

## Learning Objectives

- ✅ Compare UUID, auto-increment, Snowflake, and ULID for distributed ID generation
- ✅ Design a Snowflake-like ID generator with time-ordering and uniqueness
- ✅ Understand why database auto-increment doesn't work in distributed systems
- ✅ Choose the right ID strategy for a given system design

---

## Introduction — Why This Matters

**Interview relevance:** "How would you generate unique IDs across distributed servers?" is a common interview question and a component in many designs. You can't use database auto-increment when you have multiple shards. Snowflake IDs are the industry standard — understanding their bit layout shows deep systems knowledge.

---

## Core Concepts

### 1. Why Auto-Increment Fails at Scale

```
Single DB: Auto-increment works fine.
  INSERT → id=1, INSERT → id=2, INSERT → id=3

Sharded DB: Auto-increment breaks!
  Shard 1: INSERT → id=1
  Shard 2: INSERT → id=1  ← COLLISION!

Workarounds (all have problems):
  Odd/even: Shard 1 uses odd IDs, Shard 2 uses even
    → Hard to add new shards
  Range allocation: Shard 1 gets 1-1M, Shard 2 gets 1M-2M
    → Ranges can run out, rebalancing is complex
```

### 2. ID Generation Strategies

| Strategy | Format | Pros | Cons | Best For |
|----------|--------|------|------|----------|
| **UUID v4** | `550e8400-e29b-41d4-a716-446655440000` | No coordination needed, globally unique | 128 bits (large), not sortable, bad index perf | Simple systems, no ordering needed |
| **Snowflake** | 64-bit integer | Sortable by time, compact, no coordination | Need to manage machine IDs | Most distributed systems |
| **ULID** | `01ARZ3NDEKTSV4RRFFQ69G5FAV` | Sortable, URL-safe, UUID-compatible | 128 bits | When you need sortable + string format |
| **Database ticket server** | Auto-increment on a central DB | Simple, sequential | SPOF, bottleneck | Small-scale, non-critical |

### 3. Twitter's Snowflake ID (The Standard)

```
64-bit Snowflake ID layout:

┌─────────────────────────────────────────────────────────────┐
│ 0 │    41 bits timestamp    │ 10 bits │   12 bits sequence  │
│   │   (milliseconds since   │ machine │   (per-machine      │
│   │    custom epoch)        │ ID      │    counter)          │
│ 1 │          41             │   10    │        12            │
│bit│                         │         │                      │
└─────────────────────────────────────────────────────────────┘

Bit 0:      Sign bit (always 0, unused)
Bits 1-41:  Timestamp (ms since epoch) → ~69 years of IDs
Bits 42-51: Machine/datacenter ID → 1024 unique machines
Bits 52-63: Sequence number → 4096 IDs per ms per machine

Capacity: 4096 IDs/ms × 1024 machines = 4M IDs/ms = 4B IDs/sec

Properties:
  ✅ Globally unique (machine ID prevents collisions)
  ✅ Roughly time-sorted (timestamp is the most significant bits)
  ✅ 64 bits (fits in a BIGINT, efficient B-tree index)
  ✅ No coordination needed (each machine generates independently)
  ✅ High throughput (4096 IDs/ms per machine)
```

```python
# Snowflake ID Generator (pseudocode)
class SnowflakeGenerator:
    EPOCH = 1288834974657  # Custom epoch (Twitter's)
    MACHINE_BITS = 10
    SEQUENCE_BITS = 12
    
    def __init__(self, machine_id):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
    
    def next_id(self):
        timestamp = current_time_ms() - self.EPOCH
        
        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & 0xFFF  # 4095 max
            if self.sequence == 0:
                timestamp = wait_next_ms(timestamp)  # Exhausted sequence
        else:
            self.sequence = 0
        
        self.last_timestamp = timestamp
        
        return (timestamp << 22) | (self.machine_id << 12) | self.sequence
```

### 4. Clock Skew Problem

```
Problem: If a server's clock goes BACKWARD (NTP correction),
Snowflake could generate duplicate timestamps.

Server A at time T=1000: generates ID with timestamp 1000
Server A's clock corrects backward to T=998
Server A at time T=998: generates ID with timestamp 998
  → Could collide with IDs generated at actual T=998!

Solutions:
  1. Refuse to generate IDs if clock goes backward (wait)
  2. Use logical clocks instead of wall clocks
  3. Use NTP with slew mode (gradual correction, never jumps back)
```

---

## Common Interview Questions

### Q1: "How would you generate unique IDs for a URL shortener?"

**Model answer:**
"I'd use a Snowflake-like ID generator. Each application server has a unique machine ID and generates 64-bit IDs independently — no coordination needed. The IDs are roughly time-sorted which helps with database indexing. I'd then Base62-encode the 64-bit integer to get a short alphanumeric string for the URL. 64 bits in Base62 gives about 11 characters, but I'd truncate to 7 characters which gives 62^7 ≈ 3.5 trillion possible URLs — more than enough."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 20 CHEAT SHEET                        │
│                  Unique ID Generation                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SNOWFLAKE (64-bit): timestamp(41) + machine(10) + seq(12)     │
│  → Time-sortable, no coordination, 4K IDs/ms/machine           │
│  → Used by Twitter, Discord, Instagram                          │
│                                                                  │
│  UUID v4 (128-bit): Random, no coordination, not sortable      │
│  ULID (128-bit): Sortable UUID alternative                      │
│                                                                  │
│  INTERVIEW DEFAULT: Snowflake for most distributed systems      │
│  URL SHORTENER: Snowflake ID → Base62 encode → short string    │
│                                                                  │
│  PITFALL: Clock skew can cause duplicate timestamps.            │
│  Solution: Refuse to generate if clock goes backward.           │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | Why doesn't auto-increment work for distributed systems? | Multiple shards generate the same IDs independently, causing collisions. |
| 2 | What is a Snowflake ID? | A 64-bit ID: 41 bits timestamp + 10 bits machine ID + 12 bits sequence. Time-sortable, no coordination, high throughput. |
| 3 | How many IDs can one Snowflake machine generate per millisecond? | 4096 (2^12, the sequence bits). |
| 4 | UUID vs Snowflake — when to use each? | Snowflake when you need sortable, compact IDs (most cases). UUID when simplicity is paramount and sorting/compactness don't matter. |
| 5 | What is the clock skew problem? | If a server's clock goes backward, Snowflake could generate IDs with past timestamps, risking collisions. Solved by refusing to generate IDs until clock catches up. |

---

[← Previous: Rate Limiting](./chapter-19-rate-limiting.md) · [Next: Consensus & Leader Election →](./chapter-21-consensus-leader-election.md)
