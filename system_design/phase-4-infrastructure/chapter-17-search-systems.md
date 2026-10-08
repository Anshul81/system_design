# Chapter 17: Search Systems

> **Phase 4 — Infrastructure & Middleware** · Estimated Time: 2.5 hours · Difficulty: 🟡 Medium

[← Previous: Microservices vs Monolith](./chapter-16-microservices-vs-monolith.md) · [Next: Monitoring & Logging →](./chapter-18-monitoring-logging.md)

---

## Learning Objectives

- ✅ Explain inverted indexes and how full-text search works
- ✅ Understand Elasticsearch architecture and when to use it
- ✅ Design a search pipeline: indexing, querying, ranking
- ✅ Discuss autocomplete/typeahead implementation with tries
- ✅ Know when to add a search engine to your system design

---

## Introduction — Why This Matters

**Interview relevance:** If your system needs search (products, messages, tweets, documents), you need to explain how. The answer is almost always "Elasticsearch with an inverted index." Understanding how data gets from the primary database into the search index (sync pipeline) is critical.

---

## Core Concepts

### 1. Inverted Index

```
FORWARD INDEX (what a database has):
  Document 1: "the quick brown fox"
  Document 2: "the lazy brown dog"
  Document 3: "the quick lazy fox"

INVERTED INDEX (what a search engine builds):
  "the"   → [Doc 1, Doc 2, Doc 3]
  "quick" → [Doc 1, Doc 3]
  "brown" → [Doc 1, Doc 2]
  "fox"   → [Doc 1, Doc 3]
  "lazy"  → [Doc 2, Doc 3]
  "dog"   → [Doc 2]

Search "quick fox" →
  "quick" → [Doc 1, Doc 3]
  "fox"   → [Doc 1, Doc 3]
  Intersection: [Doc 1, Doc 3] ← Results!
```

### 2. Elasticsearch Architecture

```
┌────────────────────────────────────────────────────┐
│              ELASTICSEARCH CLUSTER                  │
│                                                    │
│  Index: "products" (like a database table)          │
│  ┌──────────────────────────────────────┐          │
│  │ Shard 0 (Primary) │ Shard 0 (Replica)│          │
│  │ Node 1            │ Node 2           │          │
│  ├──────────────────────────────────────┤          │
│  │ Shard 1 (Primary) │ Shard 1 (Replica)│          │
│  │ Node 2            │ Node 3           │          │
│  ├──────────────────────────────────────┤          │
│  │ Shard 2 (Primary) │ Shard 2 (Replica)│          │
│  │ Node 3            │ Node 1           │          │
│  └──────────────────────────────────────┘          │
│                                                    │
│  Each shard = inverted index + stored documents    │
│  Shards distributed across nodes for scale         │
│  Replicas for availability                         │
└────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Index** = collection of documents (like a DB table)
- **Shard** = horizontal partition of an index
- **Replica** = copy of a shard for redundancy
- **Analyzer** = tokenizer + filters (lowercase, stemming, stopwords)

### 3. Search Data Pipeline

```
How data gets into the search index:

┌──────────┐     ┌─────────┐     ┌──────────────┐
│ Primary   │────▶│ Kafka    │────▶│ Indexer       │
│ Database  │ CDC │ (change  │     │ (writes to    │
│ (MySQL)   │     │  stream) │     │  Elasticsearch)│
└──────────┘     └─────────┘     └──────┬───────┘
                                         │
                                  ┌──────▼───────┐
                                  │ Elasticsearch │
                                  │ (search index)│
                                  └──────────────┘

CDC = Change Data Capture
When a product is updated in MySQL, the change event flows
through Kafka to the indexer, which updates Elasticsearch.

This ensures the search index is eventually consistent
with the primary database (usually seconds of delay).
```

### 4. Autocomplete / Typeahead

```
Trie (Prefix Tree) for Autocomplete:

           (root)
          / | \
         t  a  s
        /   |   \
       tw  ap   sy
      /    |     \
    twi   app   sys
    /      |      \
  twit   appl   syst
  /       |       \
twitt   apple   syste
  |               |
twitter          system

Search "tw" → traverse to "tw" node → return all children:
  → twitter, twitch, tweet, ...

Optimization: Store top-K completions at each node
  "tw" node → ["twitter", "twitch", "tweet"]
  Pre-computed, no traversal needed → O(1) lookup
```

---

## Common Interview Questions

### Q1: "How would you design search for an e-commerce platform?"

**Model answer:**
"I'd use Elasticsearch for the search index. Products from the primary database (PostgreSQL) flow through a CDC pipeline (Debezium → Kafka → Indexer) into Elasticsearch. When a user searches, the query goes to Elasticsearch which uses an inverted index for fast full-text matching, then applies filters (category, price range, rating) and sorts by relevance score. For autocomplete, I'd use Elasticsearch's completion suggester or a separate trie-based service. The key design choice is that the primary database remains the source of truth — Elasticsearch is a derived, eventually-consistent search index."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 17 CHEAT SHEET                        │
│                  Search Systems                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  INVERTED INDEX: word → list of documents containing it         │
│  ELASTICSEARCH: Distributed search engine, sharded + replicated │
│                                                                  │
│  SEARCH PIPELINE: DB → CDC → Kafka → Indexer → Elasticsearch  │
│  Source of truth: Primary DB. ES is a derived index.            │
│                                                                  │
│  AUTOCOMPLETE: Trie (prefix tree) or ES completion suggester   │
│  RANKING: TF-IDF, BM25, personalization, click-through rate    │
│                                                                  │
│  WHEN TO USE: Product search, message search, log search,      │
│  full-text search, faceted search, autocomplete                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What is an inverted index? | A mapping from each word/token to the list of documents containing it. Enables fast full-text search. |
| 2 | How does data get from the primary DB to Elasticsearch? | CDC (Change Data Capture) pipeline: DB changes → Kafka → Indexer service → writes to Elasticsearch. Eventually consistent. |
| 3 | What data structure is used for autocomplete? | Trie (prefix tree). Each node represents a character, and paths from root to leaves represent words/completions. |
| 4 | Is Elasticsearch the source of truth? | No. The primary database is the source of truth. Elasticsearch is a derived, eventually-consistent search index. |
| 5 | What is BM25? | A ranking algorithm used by Elasticsearch to score document relevance based on term frequency, inverse document frequency, and document length. |
| 6 | When would you use Elasticsearch vs a database's built-in search? | ES for full-text search at scale, complex queries with facets/filters, fuzzy matching, relevance scoring. DB search (LIKE) for simple, small-scale text matching. |

---

[← Previous: Microservices vs Monolith](./chapter-16-microservices-vs-monolith.md) · [Next: Monitoring & Logging →](./chapter-18-monitoring-logging.md)
