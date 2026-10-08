# Chapter 15: Message Queues & Event Streaming

> **Phase 4 — Infrastructure & Middleware** · Estimated Time: 3.5 hours · Difficulty: 🟡 Medium

[← Previous: Reverse Proxies & API Gateways](../phase-3-load-balancing-caching/chapter-14-reverse-proxies-api-gateways.md) · [Next: Microservices vs Monolith →](./chapter-16-microservices-vs-monolith.md)

---

## Learning Objectives

- ✅ Compare message queues (point-to-point) vs pub/sub (fan-out) vs event streaming (log)
- ✅ Explain Kafka architecture: topics, partitions, consumer groups, offsets
- ✅ Design async processing pipelines for order processing, notifications, and ETL
- ✅ Handle delivery guarantees: at-most-once, at-least-once, exactly-once
- ✅ Know when to introduce a message queue in system design

---

## Introduction — Why This Matters

**Real-world analogy:** A message queue is like a postal service. The sender drops a letter (message) in the mailbox (queue) and continues their day. They don't wait for the recipient to read it. The postal service (broker) ensures delivery. The recipient picks it up when they're ready.

**Interview relevance:** Almost every system design needs async processing. When a user places an order: validate synchronously, then process payment, update inventory, send confirmation email, and notify analytics asynchronously via a message queue. If you design everything as synchronous API calls, the interviewer will push back: "What if the email service is slow? Does the user wait?"

---

## Visual Diagram — Message Queue vs Pub/Sub vs Event Streaming

```
1. MESSAGE QUEUE (Point-to-Point):
   Each message consumed by exactly ONE consumer.

   Producer ──── msg ────▶ [  Queue  ] ──── msg ────▶ Consumer A
                                                    or Consumer B
                                                    (one gets it)

   Use: Task distribution (email sending, image processing)
   Example: RabbitMQ, Amazon SQS, Redis Lists

2. PUB/SUB (Fan-out):
   Each message delivered to ALL subscribers.

   Publisher ──── msg ──▶ [ Topic ] ──┬── msg ──▶ Subscriber 1
                                       ├── msg ──▶ Subscriber 2
                                       └── msg ──▶ Subscriber 3

   Use: Event broadcasting (order placed → notify inventory,
        billing, shipping, analytics ALL at once)
   Example: Redis Pub/Sub, Google Pub/Sub, SNS

3. EVENT STREAMING (Log):
   Messages are persisted in an ordered, replayable log.
   Consumers read at their own pace using offsets.

   Producer ──── events ──▶ [ Log/Stream ]
                             offset 0: event_a
                             offset 1: event_b
                             offset 2: event_c ◀── Consumer Group 1
                             offset 3: event_d     (at offset 2)
                             offset 4: event_e
                                                ◀── Consumer Group 2
                                                    (at offset 4)

   Use: Event sourcing, data pipelines, real-time analytics
   Example: Apache Kafka, Amazon Kinesis, Redpanda
```

---

## Core Concepts

### 1. Kafka Architecture (Most Important for Interviews)

```
KAFKA CLUSTER:

┌──────────────────────────────────────────────────────────────┐
│                     KAFKA CLUSTER                             │
│                                                              │
│  Topic: "orders"                                             │
│  ┌────────────────────────────────────────┐                  │
│  │ Partition 0: [msg0] [msg3] [msg6] [msg9]│                  │
│  │ Partition 1: [msg1] [msg4] [msg7]       │                  │
│  │ Partition 2: [msg2] [msg5] [msg8]       │                  │
│  └────────────────────────────────────────┘                  │
│                                                              │
│  Each partition:                                              │
│  • Ordered, append-only log                                  │
│  • Assigned to one broker (leader)                           │
│  • Replicated to other brokers (followers)                   │
│  • Messages retained for configurable time (e.g., 7 days)   │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                     │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │                    │
│  │ P0 lead  │  │ P1 lead  │  │ P2 lead  │                    │
│  │ P1 rep   │  │ P2 rep   │  │ P0 rep   │                    │
│  └─────────┘  └─────────┘  └─────────┘                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘

CONSUMER GROUPS:

Consumer Group A (payment processing):
  Consumer A1 ← reads Partition 0
  Consumer A2 ← reads Partition 1
  Consumer A3 ← reads Partition 2
  (Each partition read by exactly ONE consumer in the group)

Consumer Group B (analytics):
  Consumer B1 ← reads ALL partitions
  (Independent of Group A — reads ALL messages too)
```

**Kafka key concepts:**
- **Topic:** A named category of messages (like "orders", "user-events")
- **Partition:** A topic is split into partitions for parallelism. Messages within a partition are ordered.
- **Producer:** Writes messages to topics. Can specify partition key.
- **Consumer Group:** A set of consumers that share the work of reading a topic. Each partition is consumed by exactly one consumer in the group.
- **Offset:** A consumer's position in a partition. Consumers commit offsets to track progress.
- **Retention:** Messages are kept for a configurable duration (e.g., 7 days), not deleted after consumption.

### 2. Delivery Guarantees

| Guarantee | Meaning | How | Risk |
|-----------|---------|-----|------|
| **At-most-once** | Message delivered 0 or 1 time | Commit offset before processing | Messages can be lost |
| **At-least-once** | Message delivered 1 or more times | Commit offset after processing | Duplicates possible |
| **Exactly-once** | Message delivered exactly 1 time | Idempotent consumers + transactional offsets | Complex, slower |

```
AT-LEAST-ONCE (most common):

1. Consumer reads message
2. Consumer processes message (e.g., charge payment)
3. Consumer commits offset

What if consumer crashes after step 2, before step 3?
→ Consumer restarts, re-reads the same message, processes AGAIN
→ Payment charged TWICE! (duplicate)

Fix: Make processing IDEMPOTENT
→ Use an idempotency key (e.g., order_id)
→ Before charging: check if order_id was already charged
→ If yes, skip. If no, charge and record.
```

**Interview tip:** "I'd use at-least-once delivery with idempotent consumers. Each message has a unique ID, and the consumer checks if it's already processed before acting. This is simpler and more reliable than exactly-once."

### 3. Common Message Queue Patterns

#### Work Queue (Task Distribution)

```
                    ┌──────────┐
Producer ──msg────▶│          │──msg──▶ Worker 1 (processes msg A)
Producer ──msg────▶│  Queue   │──msg──▶ Worker 2 (processes msg B)
Producer ──msg────▶│          │──msg──▶ Worker 3 (processes msg C)
                    └──────────┘

Use: Image processing, email sending, video transcoding
Scale: Add more workers for more throughput
```

#### Event-Driven Architecture

```
Order Placed Event:

Order Service ──── OrderPlaced ──▶ [Event Bus / Kafka]
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
              ┌──────────┐      ┌──────────┐      ┌──────────┐
              │ Payment   │      │ Inventory │      │ Notif.   │
              │ Service   │      │ Service   │      │ Service  │
              │ (charge)  │      │ (reserve) │      │ (email)  │
              └──────────┘      └──────────┘      └──────────┘

Each service reacts independently. Loosely coupled.
If Email Service is down, orders still process.
```

### 4. Kafka vs RabbitMQ vs SQS

| Feature | Kafka | RabbitMQ | Amazon SQS |
|---------|-------|----------|-----------|
| **Model** | Event log (streaming) | Message queue + pub/sub | Message queue |
| **Ordering** | Per-partition | Per-queue | Best-effort (FIFO optional) |
| **Retention** | Configurable (days/weeks) | Until consumed | 14 days max |
| **Replay** | ✅ Yes (re-read old messages) | ❌ No | ❌ No |
| **Throughput** | Very high (100K+ msg/sec) | Moderate (10K-50K msg/sec) | Moderate |
| **Best for** | Event streaming, data pipelines | Task queues, RPC | Serverless, simple queues |
| **Complexity** | High (manage cluster) | Medium | Low (managed) |

**Interview default:** Kafka for event streaming and data pipelines. SQS for simple task queues in AWS. RabbitMQ when you need advanced routing patterns.

---

## Real-World Example — Uber's Event-Driven Architecture

```
Uber uses Kafka for real-time event processing:

┌──────────┐  ┌──────────┐  ┌──────────┐
│ Rider App │  │ Driver   │  │ Backend  │
│           │  │ App      │  │ Services │
└─────┬────┘  └────┬─────┘  └────┬─────┘
      │            │              │
      └────────────┼──────────────┘
                   │
            ┌──────▼──────┐
            │   Kafka      │  1 TRILLION+ messages/day
            │   Cluster    │  Hundreds of topics
            └──────┬──────┘
      ┌────────────┼────────────┐
      ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Trip      │ │ ETA      │ │ Fraud    │
│ Processing│ │ Engine   │ │ Detection│
│ (matching │ │ (real-   │ │ (pattern │
│  riders   │ │  time    │ │  analysis│
│  to       │ │  pred.)  │ │  on every│
│  drivers) │ │          │ │  trip)   │
└──────────┘ └──────────┘ └──────────┘
```

---

## Common Interview Questions

### Q1: "When would you add a message queue to your design?"

**Model answer:**
"I'd add a message queue whenever I need to decouple components or process work asynchronously. Specific triggers: first, when a synchronous operation doesn't need an immediate response — like sending an email after signup. Second, when downstream services might be slow or unreliable — the queue buffers messages and retries. Third, when I need to fan out one event to multiple consumers — like an order event going to payment, inventory, and notification services. Fourth, when I need to handle traffic spikes — the queue absorbs bursts that would overwhelm downstream services."

### Q2: "Explain Kafka consumer groups."

**Model answer:**
"A consumer group is a set of consumers that collectively read a topic. Each partition within the topic is assigned to exactly one consumer in the group, so messages are load-balanced. If I have a topic with 6 partitions and 3 consumers in a group, each consumer reads 2 partitions. If a consumer dies, its partitions are reassigned to the remaining consumers. Multiple consumer groups can read the same topic independently — for example, one group processes payments while another updates analytics — each group gets ALL messages."

---

## Common Mistakes

| ❌ Wrong | ✅ Right |
|----------|---------|
| Using sync API calls for everything | Identify operations that don't need immediate response and use async messaging |
| "Kafka guarantees exactly-once" | Kafka supports exactly-once semantics, but in practice most systems use at-least-once with idempotent consumers — simpler and more reliable |
| More consumers than partitions | Extra consumers sit idle! Number of consumers in a group ≤ number of partitions |
| Ignoring dead letter queues | Failed messages need somewhere to go. Always configure a dead letter queue for messages that fail processing repeatedly |

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 15 CHEAT SHEET                        │
│                  Message Queues & Event Streaming                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  QUEUE (1-to-1): Each message → one consumer (SQS, RabbitMQ)   │
│  PUB/SUB (1-to-N): Each message → all subscribers              │
│  STREAMING (log): Ordered, replayable log (Kafka, Kinesis)     │
│                                                                  │
│  KAFKA: Topics → Partitions → Consumer Groups                  │
│  • Partitions = parallelism unit                                │
│  • Consumer per partition per group                              │
│  • Messages retained, replayable                                │
│                                                                  │
│  DELIVERY: At-least-once + idempotent consumers (default)       │
│  DEAD LETTER QUEUE: For messages that fail repeatedly            │
│                                                                  │
│  WHEN TO USE:                                                    │
│  • Async processing (email, video transcoding)                  │
│  • Decoupling services                                          │
│  • Traffic spike buffering                                      │
│  • Event-driven architecture (fan-out)                          │
│                                                                  │
│  INTERVIEW: Kafka for streaming, SQS for simple tasks          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What's the difference between a message queue and event streaming? | Queue: message consumed once, then deleted. Streaming (Kafka): messages are retained in an ordered log, replayable by multiple consumers. |
| 2 | What is a Kafka consumer group? | A set of consumers that share the work of reading a topic. Each partition is read by exactly one consumer in the group. |
| 3 | What is at-least-once delivery? | Messages are delivered one or more times. Consumer commits offset after processing. If it crashes before committing, it reprocesses — possible duplicates. |
| 4 | How do you handle message duplicates? | Idempotent consumers: check a unique message ID/idempotency key before processing. Skip if already processed. |
| 5 | What is a dead letter queue? | A special queue where messages that fail processing repeatedly are sent, for later investigation and manual handling. |
| 6 | Why can't you have more consumers than partitions in Kafka? | Each partition is assigned to exactly one consumer in a group. Extra consumers have no partitions and sit idle. |
| 7 | Kafka vs SQS — when to use each? | Kafka: event streaming, data pipelines, replay needed. SQS: simple task queues, serverless, no infra management. |
| 8 | What is event-driven architecture? | Services communicate through events published to a message bus. Each service reacts independently to events it cares about. Loosely coupled. |

---

[← Previous: Reverse Proxies & API Gateways](../phase-3-load-balancing-caching/chapter-14-reverse-proxies-api-gateways.md) · [Next: Microservices vs Monolith →](./chapter-16-microservices-vs-monolith.md)
