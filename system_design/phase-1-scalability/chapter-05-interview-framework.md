# Chapter 5: The System Design Interview Framework

> **Phase 1 — Scalability Fundamentals** · Estimated Time: 2 hours · Difficulty: 🟢 Beginner

[← Previous: Back-of-the-Envelope Estimation](./chapter-04-estimation.md) · [Next: SQL Databases →](../phase-2-databases/chapter-06-sql-databases.md)

---

## Learning Objectives

After this chapter, you will be able to:

- ✅ Structure a 45-minute system design interview using the 4-step framework
- ✅ Allocate time effectively across each step
- ✅ Know what interviewers evaluate at each stage
- ✅ Avoid the most common structural mistakes
- ✅ Drive the conversation and manage scope confidently
- ✅ Transition smoothly between steps

---

## Prerequisites & Time Estimate

| Prerequisite | Required? |
|-------------|-----------|
| Chapters 0-4 (Networking, APIs, Scaling, Estimation) | ✅ Yes |

| Component | Time |
|-----------|------|
| Reading & concepts | 1 hr |
| Practice walkthrough | 30 min |
| Flashcard review | 30 min |

---

## Introduction — Why This Matters

**Real-world analogy:** A chef doesn't start cooking without a recipe and a plan. They first understand what dish to make (requirements), estimate portions (back-of-envelope), plan the overall meal (high-level design), then execute the details (deep dive). Jumping straight to cooking without a plan leads to chaos.

**Interview relevance:** This is THE most important chapter for your interview performance. A brilliant design with poor structure gets a "no hire." A reasonable design delivered with clear structure gets an offer. Interviewers have a rubric, and the rubric rewards structured thinking. This chapter gives you the exact structure that maps to how interviewers evaluate candidates.

---

## Visual Diagram — The 4-Step Framework

```
┌─────────────────────────────────────────────────────────────────┐
│              THE SYSTEM DESIGN INTERVIEW FRAMEWORK              │
│                     (45 minutes total)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1: REQUIREMENTS (5-7 min)                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • Clarify functional requirements (features)             │    │
│  │ • Define non-functional requirements (scale, latency)    │    │
│  │ • Scope the problem (what's in, what's out)              │    │
│  │ • Define users, scale, and constraints                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│          │                                                       │
│          ▼                                                       │
│  STEP 2: ESTIMATION (3-5 min)                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • QPS (reads and writes)                                 │    │
│  │ • Storage (daily, yearly, total)                         │    │
│  │ • Bandwidth                                              │    │
│  │ • Memory (cache sizing)                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│          │                                                       │
│          ▼                                                       │
│  STEP 3: HIGH-LEVEL DESIGN (10-15 min)                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • Draw the architecture diagram                          │    │
│  │ • Identify major components                              │    │
│  │ • Show data flow between components                      │    │
│  │ • Define APIs for key operations                         │    │
│  │ • Choose databases and storage                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│          │                                                       │
│          ▼                                                       │
│  STEP 4: DETAILED DESIGN (15-20 min)                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ • Deep dive into 2-3 critical components                 │    │
│  │ • Data model (schema design)                             │    │
│  │ • Algorithms and trade-off analysis                      │    │
│  │ • Scaling strategies and bottleneck resolution            │    │
│  │ • Error handling, edge cases, monitoring                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### Step 1: Requirements Gathering (5-7 minutes)

This is where most candidates fail. They jump into drawing boxes immediately without understanding what they're building.

**What to ask:**

```
FUNCTIONAL REQUIREMENTS (Features):
"What are the core features?"

Example — Design Twitter:
✅ Post a tweet (text, images)
✅ View home timeline (feed of tweets from people you follow)
✅ Follow/unfollow users
✅ Like and retweet
❌ Direct messages (out of scope — say so explicitly)
❌ Search (might be out of scope — ask the interviewer)
❌ Trending topics (out of scope unless interviewer says otherwise)


NON-FUNCTIONAL REQUIREMENTS (Quality attributes):
"What are the scale and quality expectations?"

• Scale: How many users? DAU?
• Availability: 99.9%? 99.99%?
• Latency: < 200ms for reads? < 500ms for writes?
• Consistency: Can we tolerate eventual consistency for feeds?
• Durability: Can we ever lose a tweet?
```

**How to structure the conversation:**

```
YOU:   "Let me make sure I understand the problem. We're designing
       a Twitter-like social media platform. Let me clarify the
       requirements."

YOU:   "For core features, I'm thinking: users can post tweets,
       view a home timeline of tweets from people they follow,
       and follow/unfollow other users. Should I also include
       features like search, DMs, or trending topics?"

THEM:  "Let's keep it to posting, timeline, and follow. Skip DMs
       and trending for now."

YOU:   "Great. For non-functional requirements: What scale are we
       targeting? Something like Twitter's 250M DAU?"

THEM:  "Yes, let's design for that scale."

YOU:   "OK, and for the timeline, is eventual consistency acceptable?
       Meaning a tweet might take a few seconds to appear in all
       followers' feeds?"

THEM:  "Yes, that's fine."

YOU:   "Perfect. Let me quickly estimate the numbers before we
       start the design."
```

**What interviewers evaluate in Step 1:**
- Can you identify the core problem vs nice-to-haves?
- Do you ask clarifying questions or make assumptions?
- Can you distinguish functional from non-functional requirements?
- Do you scope the problem appropriately?

---

### Step 2: Back-of-the-Envelope Estimation (3-5 minutes)

Apply the formulas from Chapter 4. Keep it fast.

```
YOU: "Let me estimate the key numbers:
     - 250M DAU
     - Average user reads 100 tweets/day, writes 0.5 tweets/day

     Reads: 250M × 100 / 100K = 250K reads/sec
     Writes: 250M × 0.5 / 100K = 1,250 writes/sec
     Read:Write ratio = ~200:1 — this is heavily read-heavy.

     Storage: 125M tweets/day × 560 bytes = 70 GB/day text
     With media (10% have images): ~10 TB/day total

     Cache: 20% of 70 GB = 14 GB for hot tweets — fits in one
     Redis instance.

     Key insight: This is read-heavy, so we'll optimize for reads.
     Caching will be critical."
```

**Time allocation tip:** Don't spend more than 3 minutes on estimation unless the interviewer wants to dig deeper. The goal is to set the stage for design decisions, not to solve a math problem.

---

### Step 3: High-Level Design (10-15 minutes)

This is where you draw the architecture. Start with the client, go through each layer, and end at the database.

**How to draw the diagram:**

```
Step 1: Start with the client and API
┌────────┐      ┌─────────────┐
│ Client  │─────▶│ API Gateway  │
└────────┘      └──────┬──────┘

Step 2: Add the core services
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
        ┌──────────┐ ┌──────────┐ ┌──────────────┐
        │ Tweet     │ │ Timeline │ │ User          │
        │ Service   │ │ Service  │ │ Service       │
        └────┬─────┘ └────┬─────┘ └──────┬───────┘

Step 3: Add storage
              │            │              │
              ▼            ▼              ▼
        ┌──────────┐ ┌──────────┐ ┌──────────────┐
        │ Tweet DB  │ │ Cache    │ │ User DB       │
        │ (MySQL)   │ │ (Redis)  │ │ (MySQL)       │
        └──────────┘ └──────────┘ └──────────────┘

Step 4: Add supporting infrastructure
  ┌──────────┐
  │ CDN       │ ← Static assets, images
  └──────────┘
  ┌──────────┐
  │ Object    │ ← Image/video storage (S3)
  │ Storage   │
  └──────────┘
  ┌──────────┐
  │ Message   │ ← Async processing (fan-out, notifications)
  │ Queue     │
  └──────────┘
```

**What to explain for each component:**
1. **What** it does (one sentence)
2. **Why** it's needed (what problem it solves)
3. **How** data flows through it

**What interviewers evaluate in Step 3:**
- Can you identify the right components?
- Do you understand the data flow?
- Is your diagram reasonable for the scale?
- Did you mention load balancers, caches, and CDN where appropriate?

---

### Step 4: Detailed Design (15-20 minutes)

This is where you go deep on 2-3 critical components. The interviewer usually guides this: "Tell me more about how the timeline works" or "How would you handle the database schema?"

**What to deep dive on:**

| Component | What to Discuss |
|-----------|----------------|
| **Data model** | Tables, columns, indexes, relationships. SQL vs NoSQL choice. |
| **API design** | Key endpoints with request/response. Pagination. |
| **Algorithms** | Fan-out strategy, hashing, ranking, rate limiting |
| **Scaling** | How to handle 10x traffic. Sharding strategy. Caching. |
| **Bottlenecks** | What breaks first? How to fix it? |
| **Trade-offs** | Every decision should have a stated trade-off |

**Example deep dive — Twitter fan-out:**

```
YOU: "The most interesting design decision here is how we generate
     the home timeline. There are two approaches:

     OPTION 1: Fan-out on WRITE (push model)
     When a user posts a tweet, we immediately push it to all
     their followers' timeline caches.

     ┌─────┐  post   ┌─────────┐  push to  ┌───────────────┐
     │User A│───────▶│Tweet Svc │──────────▶│All followers' │
     │      │        │          │  each     │timeline caches│
     └─────┘        └─────────┘  follower  └───────────────┘

     Pro: Timeline reads are fast (pre-computed).
     Con: Expensive for users with millions of followers.
          A celebrity posting → millions of cache writes.

     OPTION 2: Fan-out on READ (pull model)
     When a user opens their timeline, we fetch recent tweets
     from everyone they follow and merge them.

     ┌─────┐  read   ┌──────────┐  query each  ┌─────────┐
     │User B│───────▶│Timeline  │─────────────▶│Follow DB │
     │      │        │Service   │  followed    │Tweet DB  │
     └─────┘        └──────────┘  user's tweet └─────────┘

     Pro: Simple writes. No celebrity problem.
     Con: Slow reads (must query many users and merge).

     HYBRID APPROACH (what Twitter actually does):
     Fan-out on write for most users (< 10K followers).
     Fan-out on read for celebrities (> 10K followers).
     This is the trade-off I'd recommend."
```

**What interviewers evaluate in Step 4:**
- Can you go deep on a component with technical accuracy?
- Do you analyze trade-offs for every decision?
- Do you consider edge cases and failure scenarios?
- Can you adapt when the interviewer steers the conversation?

---

### Time Allocation Summary

```
┌──────────────────────────────────────────────────────────────┐
│           TIME ALLOCATION (45-minute interview)              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────┐ ┌──────┐ ┌────────────────┐ ┌──────────────────┐ │
│  │ REQ  │ │ EST  │ │  HIGH-LEVEL    │ │  DETAILED DESIGN │ │
│  │      │ │      │ │  DESIGN        │ │  + SCALING       │ │
│  │5-7min│ │3-5min│ │  10-15 min     │ │  15-20 min       │ │
│  └──────┘ └──────┘ └────────────────┘ └──────────────────┘ │
│  │◄────▶│ │◄───▶│ │◄──────────────▶│ │◄────────────────▶│ │
│                                                              │
│  DANGER ZONE: If you're still on Step 1 after 10 minutes,   │
│  you're spending too long. Move to Step 2.                   │
│                                                              │
│  FLEXIBILITY: The interviewer may skip Step 2 or expand      │
│  Step 4. Read the room and adjust.                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## What Interviewers Actually Evaluate

```
┌──────────────────────────────────────────────────────────────────┐
│                  INTERVIEWER EVALUATION RUBRIC                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. PROBLEM EXPLORATION (15%)                                    │
│     • Asked clarifying questions                                 │
│     • Scoped the problem appropriately                          │
│     • Identified core vs nice-to-have features                  │
│                                                                  │
│  2. HIGH-LEVEL DESIGN (25%)                                      │
│     • Identified the right components                            │
│     • Reasonable architecture for the scale                      │
│     • Clear data flow between components                        │
│                                                                  │
│  3. DETAILED DESIGN (30%)                                        │
│     • Deep understanding of chosen components                    │
│     • Correct data models and API design                        │
│     • Solid algorithm choices                                    │
│                                                                  │
│  4. TRADE-OFF ANALYSIS (20%)                                     │
│     • Discussed alternatives for key decisions                   │
│     • Explained WHY they chose one approach over another        │
│     • Considered consistency, availability, latency trade-offs  │
│                                                                  │
│  5. COMMUNICATION (10%)                                          │
│     • Clear, structured presentation                             │
│     • Good use of diagrams                                       │
│     • Responded well to hints and pushback                      │
│     • Drove the conversation proactively                        │
│                                                                  │
│  RED FLAGS:                                                      │
│  ❌ Jumped into design without understanding requirements       │
│  ❌ No estimation (can't reason about scale)                    │
│  ❌ Drew boxes without explaining data flow                     │
│  ❌ Made decisions without stating trade-offs                   │
│  ❌ Got defensive when challenged                               │
│  ❌ Couldn't go deep when asked about a component               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Example Full Walkthrough — URL Shortener (Compressed)

Here's the entire framework applied in condensed form:

```
STEP 1: REQUIREMENTS (3 min)
────────────────────────────
Functional:
  • Shorten long URLs → short URLs
  • Redirect short URL → original URL
  • Custom aliases (optional)
  • Link expiration

Non-functional:
  • 100M new URLs/day
  • 10B redirects/day (100:1 read:write)
  • < 100ms redirect latency
  • 99.99% availability

STEP 2: ESTIMATION (2 min)
──────────────────────────
  Write QPS: 100M / 86.4K ≈ 1,200/sec
  Read QPS: 1,200 × 100 = 120K/sec
  Storage: 100M × 200 bytes = 20 GB/day → 73 TB / 10 years
  Cache: 20% of daily reads ≈ 8 GB → 1 Redis instance

STEP 3: HIGH-LEVEL DESIGN (8 min)
─────────────────────────────────
  Client → LB → App Server → Cache (Redis) → DB (MySQL)
                    ↓
              URL Generator
              (Base62 encode)

  Write path: Client → POST /api/v1/shorten → generate short code →
              store in DB → return short URL
  Read path: Client → GET /s/abc123 → check cache → if miss, check DB →
             301 redirect → cache the result

STEP 4: DETAILED DESIGN (15 min)
────────────────────────────────
  Deep dive 1: URL generation (Base62 vs hash vs counter)
  Deep dive 2: Database schema and sharding strategy
  Deep dive 3: Caching strategy for hot URLs
  Scaling: Shard DB by hash of short URL, multiple app servers,
           Redis cluster for caching
```

---

## Common Interview Questions

### Q1: "How do you handle it when you don't know the answer to something the interviewer asks?"

**Model answer:**
"I'd be honest and say 'I'm not deeply familiar with the specifics of X, but here's how I'd approach it based on what I know.' Then I'd reason through it using first principles. Interviewers value honest reasoning over bluffing. I'd also ask if they could give me a hint or point me in the right direction — collaboration is what they're testing."

### Q2: "What's the most important part of a system design interview?"

**Model answer:**
"Trade-off analysis. Every design decision has alternatives, and the interviewer wants to see that I can reason about the pros and cons. It's not about getting the 'right' answer — it's about showing I can evaluate options, make a decision, and defend it with clear reasoning. Second most important is requirements — if I build the wrong thing, the whole design is wasted."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| Jump into drawing immediately | Spend 5-7 minutes on requirements and estimation first |
| Build for 1B users from the start | Start simple, then scale. "Here's the basic design, now let's scale it." |
| One-way monologue | Make it conversational. Check in: "Does this approach make sense?" |
| Draw boxes without explaining | Every box needs: what it does, why it's there, how data flows through it |
| Only one solution | Present 2-3 options, compare trade-offs, then choose one with justification |
| Ignore the interviewer's hints | Hints are gifts. If they say "what about X?", that's where they want you to go. |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 5 CHEAT SHEET                         │
│                  The 4-Step Interview Framework                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1 — REQUIREMENTS (5-7 min):                               │
│  Ask: Features? Scale? Latency? Consistency? Availability?      │
│  Output: Scoped feature list + non-functional requirements      │
│                                                                  │
│  STEP 2 — ESTIMATION (3-5 min):                                 │
│  Calculate: QPS, storage, bandwidth, cache                      │
│  Output: Key numbers that shape the design                      │
│                                                                  │
│  STEP 3 — HIGH-LEVEL DESIGN (10-15 min):                        │
│  Draw: Client → LB → Services → Cache → DB                     │
│  Explain: Data flow for key operations (read/write)             │
│  Output: Architecture diagram + API endpoints                   │
│                                                                  │
│  STEP 4 — DETAILED DESIGN (15-20 min):                          │
│  Deep dive: 2-3 critical components                             │
│  Discuss: Schema, algorithms, trade-offs, scaling               │
│  Output: Detailed component design + scaling strategy           │
│                                                                  │
│  GOLDEN RULES:                                                   │
│  • Drive the conversation, don't wait to be led                 │
│  • State trade-offs for EVERY decision                          │
│  • Listen to hints — they're steering you                       │
│  • "It depends" is OK if followed by "here are the options"    │
│  • Start simple, then scale                                     │
│  • Check in with the interviewer periodically                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 4 steps of the system design interview framework? | 1. Requirements (5-7 min), 2. Estimation (3-5 min), 3. High-Level Design (10-15 min), 4. Detailed Design (15-20 min) |
| 2 | What's the biggest mistake candidates make? | Jumping into drawing the architecture without understanding requirements first |
| 3 | What are non-functional requirements? | Quality attributes: latency, availability, consistency, durability, scalability — not features |
| 4 | How much time should you spend on estimation? | 3-5 minutes max. Get the order of magnitude right, don't obsess over precision. |
| 5 | What should every component in your diagram have? | An explanation of: what it does, why it's needed, and how data flows through it |
| 6 | What is the most important evaluation criteria? | Trade-off analysis (20-30% of the grade). Every decision should have stated alternatives and reasoning. |
| 7 | How should you handle interviewer hints? | Treat them as steering. If they ask "what about X?", that's what they want you to discuss. Follow the lead. |
| 8 | What's the ideal level of detail for Step 3 (high-level)? | Identify all major components, show data flow for key operations, define APIs. Don't go deep — save that for Step 4. |
| 9 | How many components should you deep dive on in Step 4? | 2-3 critical components. Go deep on the most interesting/challenging parts, not everything. |
| 10 | What should you do if you're running out of time? | Summarize what you've covered, list what you would have discussed with more time, and offer to go deep on the most important remaining topic. |

---

[← Previous: Back-of-the-Envelope Estimation](./chapter-04-estimation.md) · [Next: SQL Databases →](../phase-2-databases/chapter-06-sql-databases.md)
