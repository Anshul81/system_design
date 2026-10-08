# Chapter 18: Monitoring, Logging & Alerting

> **Phase 4 — Infrastructure & Middleware** · Estimated Time: 2 hours · Difficulty: 🟡 Medium

[← Previous: Search Systems](./chapter-17-search-systems.md) · [Next: Rate Limiting →](../phase-5-distributed-systems/chapter-19-rate-limiting.md)

---

## Learning Objectives

- ✅ Distinguish metrics, logs, and traces (the three pillars of observability)
- ✅ Design a monitoring pipeline for a distributed system
- ✅ Know key metrics to track (latency, error rate, throughput, saturation)
- ✅ Understand structured logging and log aggregation

---

## Introduction — Why This Matters

**Interview relevance:** When the interviewer asks "How would you know if something goes wrong?", they expect you to discuss monitoring, alerting, and logging. Mentioning observability unprompted shows you think like a production engineer, not just a system designer.

---

## Core Concepts

### 1. Three Pillars of Observability

```
METRICS:                  LOGS:                    TRACES:
Numeric time-series       Structured event records  Request path across services
                          
CPU: 72%                  {"time":"...",            Service A → Service B →
Memory: 85%                "level":"ERROR",            Service C → DB
QPS: 12,000                "msg":"Payment            (with timing for each hop)
P99 Latency: 230ms         failed",
Error Rate: 0.3%           "user_id":"123",
                           "error":"timeout"}
```

| Pillar | What It Answers | Tools |
|--------|----------------|-------|
| **Metrics** | "Is the system healthy? What's the trend?" | Prometheus, Grafana, CloudWatch, Datadog |
| **Logs** | "What happened? Why did this request fail?" | ELK Stack (Elasticsearch, Logstash, Kibana), Splunk |
| **Traces** | "Where is the latency? Which service is slow?" | Jaeger, Zipkin, OpenTelemetry, Datadog APM |

### 2. Key Metrics (The RED and USE Methods)

```
RED Method (for services):
  Rate:    Requests per second
  Errors:  Error rate (% of requests that fail)
  Duration: Latency (P50, P95, P99)

USE Method (for resources):
  Utilization: % of resource being used (CPU 72%)
  Saturation: How queued/overloaded (queue depth)
  Errors:     Error count for the resource

GOLDEN SIGNALS (Google SRE):
  Latency:    How long requests take
  Traffic:    How many requests
  Errors:     How many fail
  Saturation: How full is the system
```

**P50, P95, P99 latency:**
```
If P99 latency = 500ms:
  99% of requests complete in ≤ 500ms
  1% of requests take > 500ms

P50 = median (typical user experience)
P95 = 95th percentile (most users)
P99 = 99th percentile (worst case for almost all users)

Interview tip: Always mention P99, not just average.
Average hides tail latency — a few 10-second requests
can hide behind a 100ms average.
```

### 3. Monitoring Architecture

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│Service A │  │Service B │  │Service C │
│ metrics  │  │ metrics  │  │ metrics  │
│ logs     │  │ logs     │  │ logs     │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌──────────┐
│Prometheus│  │ Kafka/   │  │ Jaeger    │
│(metrics) │  │ Fluentd  │  │ (traces)  │
│          │  │ (logs)   │  │           │
└────┬─────┘  └────┬─────┘  └─────┬────┘
     │             │              │
     ▼             ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────┐
│ Grafana  │  │ Elastic  │  │ Jaeger   │
│(dashboards│  │ search   │  │ UI       │
│ + alerts)│  │ + Kibana │  │          │
└─────────┘  └──────────┘  └──────────┘
```

### 4. Alerting Best Practices

| Rule | Explanation |
|------|-------------|
| Alert on symptoms, not causes | "Error rate > 5%" not "CPU > 90%" — CPU can be high and fine |
| Use severity levels | P1: customer-facing outage. P2: degraded. P3: non-urgent |
| Avoid alert fatigue | Too many alerts = ignoring alerts. Only alert on actionable issues. |
| Include runbooks | Link to a document explaining how to diagnose and fix the issue |

---

## Common Interview Questions

### Q1: "How would you monitor this system?"

**Model answer:**
"I'd implement all three pillars of observability. Metrics: Prometheus collecting request rate, error rate, and P99 latency from each service, visualized in Grafana dashboards with alerts on error rate > 1% and P99 > 500ms. Logs: Structured JSON logs shipped via Fluentd to Elasticsearch, queryable through Kibana for debugging. Traces: OpenTelemetry distributed tracing so I can follow a request across services and identify which service is causing latency. Alerts go to PagerDuty, with severity levels to avoid alert fatigue."

---

## Cheat Sheet

```
┌──────────────────────────────────────────────────────────────────┐
│                    CHAPTER 18 CHEAT SHEET                        │
│                  Monitoring, Logging & Alerting                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  THREE PILLARS: Metrics, Logs, Traces                           │
│  GOLDEN SIGNALS: Latency, Traffic, Errors, Saturation           │
│  USE P99 LATENCY, not average (hides tail latency)              │
│                                                                  │
│  STACK: Prometheus + Grafana (metrics)                          │
│         ELK (logs)     Jaeger/Zipkin (traces)                   │
│                                                                  │
│  ALERT ON: Symptoms (error rate, latency) not causes (CPU)      │
│  ALWAYS MENTION: "We'd add monitoring and alerting"             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Flashcards

| # | Question | Answer |
|---|----------|--------|
| 1 | What are the 3 pillars of observability? | Metrics (numeric time-series), Logs (event records), Traces (request paths across services) |
| 2 | What are the 4 golden signals? | Latency, Traffic, Errors, Saturation |
| 3 | Why use P99 instead of average latency? | Average hides tail latency. A few very slow requests can be masked by many fast ones. P99 shows the worst case for 99% of users. |
| 4 | What is distributed tracing? | Following a single request's path across multiple services, with timing for each hop, to identify where latency occurs. |
| 5 | What is the ELK stack? | Elasticsearch (storage), Logstash (processing), Kibana (visualization) — a popular log aggregation platform. |
| 6 | What should you alert on? | Symptoms (error rate > threshold, latency > SLA) not causes (CPU > 90%). Alerts should be actionable. |

---

[← Previous: Search Systems](./chapter-17-search-systems.md) · [Next: Rate Limiting →](../phase-5-distributed-systems/chapter-19-rate-limiting.md)
