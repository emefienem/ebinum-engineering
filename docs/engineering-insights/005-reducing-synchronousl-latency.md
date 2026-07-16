# Engineering Insight #005

**Title:** Eliminating Infrastructure Overhead from the Payment Creation Path

**Date:** 2026-07-16

---

## Background

Engineering Insight #004 focused on removing non-essential work from the synchronous payment creation path.

Risk scoring, webhook delivery, device linking, enrichment, and other post-processing tasks were moved into asynchronous background execution.

Although this significantly reduced the amount of work performed before returning a response, benchmarking still showed latency well above the target.

| Metric          | Insight #004 |
| --------------- | -----------: |
| Average Latency |       8.07 s |
| P95 Latency     |      12.25 s |

This indicated that the remaining bottleneck was no longer background processing but infrastructure overhead and synchronous service execution.

---

## Investigation

Detailed instrumentation was added throughout the payment creation pipeline.

Instead of measuring only total request duration, individual stages were timed independently.

Examples included:

- system flag evaluation;
- merchant lookup;
- payment actor resolution;
- shard execution;
- velocity checks;
- payment intent persistence;
- background task dispatch.

This made it possible to isolate infrastructure costs from business logic.

---

## Findings

Several unnecessary infrastructure costs were discovered.

### 1. Database-backed System Flags

Every payment request queried the database to determine whether:

- maintenance mode was enabled;
- payments were enabled;
- webhooks were enabled;
- risk scoring was enabled.

Although these values rarely change, every request incurred multiple database reads.

---

### 2. Database-backed Idempotency

The idempotency implementation executed several database operations for every request.

Typical flow:

```text
Request

↓

SELECT idempotency key

↓

INSERT in-progress record

↓

Execute payment creation

↓

UPDATE response

↓

Return response
```

Even though idempotency protects against duplicate requests, the implementation placed multiple synchronous database round trips directly on the critical request path.

---

### 3. Repeated ShardRouter Construction

Authentication helpers created a new `ShardRouter` instance every time a merchant or customer was resolved.

```ts
const router = new ShardRouter();
```

Although relatively inexpensive individually, repeated initialization added unnecessary object construction and connection management overhead on every authenticated request.

---

## Optimizations

### In-Memory System Flag Cache

System flags are now loaded once during service startup.

```text
Application Startup

↓

Load all system flags

↓

Store in memory

↓

Refresh periodically
```

Runtime flag checks now read directly from memory.

```ts
SystemFlagGuard.isRiskEngineEnabled();
SystemFlagGuard.isWebhooksEnabled();
```

Database access is no longer required for every payment request.

---

### Redis-first Idempotency

Idempotency was redesigned to use Redis as the primary execution path.

Instead of performing multiple database operations before every response, Redis now performs atomic lock acquisition.

```text
SET NX

↓

Execute operation

↓

Store response in Redis

↓

Return response

↓

Persist to Postgres asynchronously
```

PostgreSQL remains the durable audit store but no longer blocks client responses.

This removes multiple synchronous database round trips from the critical path.

---

### Reusing ShardRouter

Authentication now shares a single `ShardRouter` instance.

Before:

```ts
const router = new ShardRouter();
```

executed inside every helper.

After:

```ts
const shardRouter = new ShardRouter();
```

is created once and reused across requests.

This eliminates repeated infrastructure initialization during authentication.

---

### Pipeline Instrumentation

Fine-grained performance markers were introduced throughout the payment creation flow.

Each major stage now records execution time independently.

Examples include:

- flag evaluation;
- merchant resolution;
- actor creation;
- database execution;
- payment persistence;
- background dispatch.

Rather than reporting only total latency, the service now produces stage-level timing information that identifies the exact location of remaining bottlenecks.

---

## Before

```text
Request
    │
    ▼
Database Flag Lookup
    │
    ▼
Database Idempotency
    │
    ▼
Authentication
(New ShardRouter)
    │
    ▼
Payment Creation
    │
    ▼
Return Response
```

---

## After

```text
Request
    │
    ▼
Memory Flag Lookup
    │
    ▼
Redis Idempotency
    │
    ▼
Authentication
(Shared ShardRouter)
    │
    ▼
Payment Creation
    │
    ▼
Return Response

           │
           ▼

Asynchronous Postgres Persistence
```

---

## Benchmark Results

Following these infrastructure optimizations, benchmark performance improved substantially.

| Metric             |  Before |       After |
| ------------------ | ------: | ----------: |
| Requests Completed |      23 |          72 |
| Success Rate       |    100% |        100% |
| Average Latency    |  7.01 s |  **1.51 s** |
| Median Latency     |  5.36 s |  **1.10 s** |
| P95 Latency        | 15.90 s |  **2.25 s** |
| Maximum Latency    | 16.50 s | **11.47 s** |

Compared with Engineering Insight #004:

| Metric          | Insight #004 |    Insight #005 |
| --------------- | -----------: | --------------: |
| Average Latency |       8.07 s |      **1.51 s** |
| P95 Latency     |      12.25 s |      **2.25 s** |
| Throughput      |  23 requests | **72 requests** |

Average latency decreased by approximately **81%**, while throughput increased by more than **3×** under the same benchmark conditions.

---

## Analysis

The benchmark demonstrates that a significant portion of request latency originated from infrastructure rather than business logic.

Replacing synchronous database interactions with memory and Redis dramatically shortened the critical request path without changing payment behavior.

Instrumentation also confirmed that background task dispatch contributes almost no measurable latency, validating the architectural changes introduced in Engineering Insight #004.

The remaining latency is now concentrated primarily within the synchronous payment creation pipeline itself, including:

- payment actor resolution;
- merchant validation;
- shard routing;
- velocity calculations;
- payment intent persistence.

These operations now represent the majority of request execution time.

---

## Key Finding

The largest remaining performance gains were achieved not by changing payment logic but by eliminating unnecessary infrastructure work from the request path.

Caching immutable configuration in memory, moving idempotency to Redis, and reusing shared infrastructure components significantly reduced request latency while maintaining correctness and durability.

The payment service now spends considerably less time coordinating infrastructure and more time executing core payment operations.

---

## Next Step

The next phase focuses on optimizing the synchronous business logic that remains inside the payment creation pipeline.

Particular attention will be given to:

- payment actor resolution;
- merchant validation;
- velocity statistics retrieval;
- shard routing efficiency;
- database write performance.

The objective is to reduce end-to-end latency below the target of **500 ms** while preserving correctness, consistency, and scalability.
