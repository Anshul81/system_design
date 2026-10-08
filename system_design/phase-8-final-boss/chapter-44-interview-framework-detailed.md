# Chapter 44: The System Design Interview Framework (Detailed)

> **Phase 8 — Interview Strategy** · Estimated Time: 2.5 hours · Difficulty: 🟢 Beginner · Type: 📋 STRATEGY

[← Previous: Leaderboard](../phase-7-advanced-designs/chapter-43-leaderboard.md) · [Next: Requirements Deep Dive →](./chapter-45-requirements-deep-dive.md)

---

## Why This Chapter Matters

You've learned 40+ system design topics. Now you need to **present** them in a 35-45 minute interview. This chapter gives you a precise minute-by-minute framework. The biggest failure mode in system design interviews isn't technical knowledge — it's **time management** and **communication**.

---

## The 4-Step Framework (35-40 minutes)

```
┌──────────────────────────────────────────────────────────────────┐
│                SYSTEM DESIGN INTERVIEW TIMELINE                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌────────┐  ┌────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ Step 1  │  │ Step 2      │  │ Step 3        │  │ Step 4       │ │
│  │REQUIRE- │  │ESTIMATION   │  │HIGH-LEVEL     │  │DETAILED      │ │
│  │MENTS    │  │             │  │DESIGN         │  │DESIGN        │ │
│  │         │  │             │  │               │  │              │ │
│  │ 5 min   │  │ 3-5 min     │  │ 10-15 min     │  │ 15-20 min    │ │
│  └────────┘  └────────────┘  └──────────────┘  └─────────────┘ │
│                                                                    │
│  Minutes: 0──5──8──────────23──────────────────40                │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Requirements Gathering (5 minutes)

### What to Do

```
ASK QUESTIONS. Don't jump into design!

The interviewer is testing: Can you define what you're building?

Questions to ask:
  1. "What are the core features?" (nail down the scope)
  2. "What's the scale?" (DAU, QPS, data size)
  3. "What are the priorities?" (latency vs consistency?)
  4. "Any specific constraints?" (mobile-first? global?)

TEMPLATE:

  Functional Requirements:
    "So we need to support [feature 1], [feature 2], and [feature 3].
     Are there other features I should consider, or should I focus on these?"

  Non-Functional Requirements:
    "For scale, I'll assume [X DAU], [Y QPS], [Z storage].
     Does that sound right?"
    "The system should be highly available (99.99%) with 
     low latency reads. Is consistency critical or is eventual OK?"
```

### Common Pitfall

```
❌ BAD: "Let me design Twitter. So first I'll have a database..."
   → You just skipped requirements. Interviewer doesn't know 
     what you're building, and YOU might build the wrong thing.

✅ GOOD: "Before I start, let me clarify the scope. Are we focusing
   on the home timeline and tweeting? Or should I also include
   search, trending, and DMs?"
   → Shows product thinking. Scopes the problem. Saves time.
```

---

## Step 2: Back-of-the-Envelope Estimation (3-5 minutes)

### What to Do

```
Show your math. Use round numbers. Don't get bogged down.

FRAMEWORK:
  1. QPS: DAU × actions per user / 86,400
  2. Storage: daily_items × item_size × retention_period
  3. Bandwidth: QPS × average_response_size
  4. Cache: hot_data_percentage × daily_data_size

EXAMPLE (Twitter):
  "250M DAU, 2 tweets each = 500M tweets/day
   Write QPS: 500M / 86,400 ≈ 6K/sec
   Timeline reads: 250M × 100 = 25B/day → 290K/sec
   
   Storage: 500M × 500B = 250 GB/day text
   With media: ~10 TB/day
   
   So this is a read-heavy system at 50:1 read:write ratio."

SAY THE IMPLICATION:
  Don't just do math. State what it means:
  "This is read-heavy, so I'll focus on caching and 
   pre-computing timelines rather than optimizing writes."
```

### Reference Numbers (Memorize These)

```
  1 day = 86,400 sec ≈ 100K sec (for mental math)
  1 M/day = ~12/sec
  1 B/day = ~12K/sec
  
  1 KB = 1,000 bytes
  1 MB = 1,000 KB
  1 GB = 1,000 MB
  1 TB = 1,000 GB
  1 PB = 1,000 TB
  
  Single MySQL: ~10K QPS (simple queries)
  Single Redis: ~100K QPS
  Single server: ~10K concurrent connections
```

---

## Step 3: High-Level Design (10-15 minutes)

### What to Do

```
Draw the architecture. Keep it simple. Cover the main flows.

TEMPLATE:
  1. Start with the client
  2. Add API layer (load balancer, API gateway)
  3. Add core services (1-3 main services)
  4. Add data stores (DB, cache, queue)
  5. Walk through the main flows on the diagram

  "Let me walk through the two main flows:
   WRITE: User posts tweet → Tweet Service → DB → Fan-out to timelines
   READ: User opens timeline → Timeline Service → Redis cache → return"

WHAT TO INCLUDE:
  ✅ APIs (show the key endpoints)
  ✅ Main services and their responsibilities
  ✅ Data stores and why each was chosen
  ✅ Data flow arrows
  ✅ Walk through 1-2 key flows

WHAT TO SKIP (for now):
  ❌ Error handling details
  ❌ Deployment specifics
  ❌ Every edge case
  → These belong in Step 4 if the interviewer asks
```

### Signal: Explaining Your Choices

```
DON'T just draw boxes. Explain WHY.

❌ "I'll use MySQL here."
✅ "I'll use MySQL for tweet storage because we need ACID for 
    dedup (no double tweets) and the write QPS of 6K is well 
    within MySQL's capacity. I'll shard by user_id later if needed."

Every technology choice should have a one-sentence justification.
```

---

## Step 4: Detailed Design (15-20 minutes)

### What to Do

```
The interviewer will guide you. They'll pick 1-3 areas to dive deep.
Common signals:
  "Tell me more about how the timeline works"
  "What happens when a celebrity tweets?"
  "How do you handle failures here?"

WHEN DIVING DEEP:
  1. State the challenge clearly
  2. Present 2-3 options
  3. Pick one and justify
  4. Show the detailed flow or data structure

EXAMPLE:
  "The key challenge with the timeline is fan-out. Let me compare
   fan-out-on-write vs fan-out-on-read..."
   [explain both]
   "I'd use a hybrid approach because..."
   [justify with numbers from estimation]

TOPICS THEY COMMONLY PROBE:
  • Database choice & schema
  • Caching strategy
  • How specific component works in detail
  • Failure scenarios
  • Scaling bottlenecks
  • Consistency model
```

### Signal: Trade-off Discussion

```
THE #1 DIFFERENTIATOR between strong and weak candidates:

WEAK: "I'll use Redis for caching."
STRONG: "I'll use Redis with cache-aside pattern. The alternative 
  is write-through, but most URLs may never be read, so caching on
  write wastes memory. Cache-aside with LRU eviction naturally keeps
  hot data in cache."

Always mention:
  1. What you chose
  2. What the alternative was
  3. Why you chose your approach over the alternative
```

---

## Communication Strategies

### The "Think-Aloud" Protocol

```
NARRATE your reasoning. Don't go silent.

"I'm thinking about where to store the messages. Let me consider 
the access patterns — we need to query by conversation_id and 
order by time. That's a partition key + sort key pattern, which 
makes Cassandra a good fit. MySQL could work too, but at 50B 
messages/day, the write throughput favors Cassandra."

WHY THIS MATTERS:
  The interviewer can only evaluate what they hear.
  If you think silently for 2 minutes → they see nothing.
  If you narrate → they see your reasoning process, even if 
  you make a minor mistake.
```

### Handling Questions You Don't Know

```
DON'T PANIC. DON'T BULLSHIT.

"I'm not sure about the exact implementation of CRDTs, but my 
understanding is they're data structures designed for conflict-free 
merging in distributed systems. For this design, I'd use OT since 
we have a central server, which makes it simpler."

WHAT THE INTERVIEWER HEARS:
  "This candidate is honest, has directional knowledge, and can 
   make practical decisions even with incomplete information."
```

### Time Management Signals

```
CHECK THE TIME. Adjust your depth.

At 15 minutes: You should be past estimation, into high-level design.
At 25 minutes: You should be starting detailed design.
At 35 minutes: Wrap up with trade-offs and potential improvements.

IF RUNNING OUT OF TIME:
  "I notice we have about 5 minutes left. Let me quickly cover 
   the scaling strategy and key trade-offs, and I'm happy to 
   dive deeper into any area."
```

---

## The Interview Scorecard

```
WHAT INTERVIEWERS EVALUATE:

┌────────────────────┬───────┬──────────────────────────────────────┐
│ Dimension           │ Weight│ What "good" looks like                │
├────────────────────┼───────┼──────────────────────────────────────┤
│ Requirements        │ 15%  │ Asks clarifying questions, scopes     │
│                    │       │ problem appropriately                │
├────────────────────┼───────┼──────────────────────────────────────┤
│ High-Level Design  │ 25%  │ Clean architecture, correct data     │
│                    │       │ flow, reasonable tech choices        │
├────────────────────┼───────┼──────────────────────────────────────┤
│ Detailed Design    │ 30%  │ Deep understanding of 1-2 components,│
│                    │       │ handles edge cases                   │
├────────────────────┼───────┼──────────────────────────────────────┤
│ Trade-offs         │ 20%  │ Discusses alternatives, explains     │
│                    │       │ WHY decisions were made              │
├────────────────────┼───────┼──────────────────────────────────────┤
│ Communication      │ 10%  │ Clear, structured, collaborative,    │
│                    │       │ good time management                 │
└────────────────────┴───────┴──────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 4 steps of the system design interview? | 1. Requirements (5 min), 2. Estimation (3-5 min), 3. High-Level Design (10-15 min), 4. Detailed Design (15-20 min). |
| 2 | What's the biggest mistake in Step 1? | Skipping requirements and jumping straight into design. You might build the wrong thing, and you miss a chance to show product thinking. |
| 3 | What should you say after doing estimation math? | State the implication: "This is read-heavy at 50:1 ratio, so I'll focus on caching and pre-computing." The math alone isn't enough — the insight matters. |
| 4 | How do you show strong trade-off analysis? | For every decision: state what you chose, what the alternative was, and why you chose yours. "I'll use X because Y. The alternative Z doesn't work here because W." |
| 5 | What should you do if you don't know something? | Be honest. State what you do know directionally, then make a practical decision. Never bullshit — interviewers detect it instantly. |
| 6 | How do you handle running out of time? | Acknowledge it: "We have 5 minutes left, let me cover scaling and trade-offs." Then summarize key points. Shows self-awareness. |

---

[← Previous: Leaderboard](../phase-7-advanced-designs/chapter-43-leaderboard.md) · [Next: Requirements Deep Dive →](./chapter-45-requirements-deep-dive.md)
