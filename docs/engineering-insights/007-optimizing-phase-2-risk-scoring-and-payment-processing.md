# Engineering Insight #007

**Title:** Optimizing Phase 2 Risk Scoring and the Payment Processing Pipeline

**Date:** 2026-07-19

---

# Background

Engineering Insight #006 introduced stage-level instrumentation into the payment processing pipeline.

By measuring each major operation independently, it became possible to identify where execution time was actually being spent rather than treating payment processing as a single opaque operation.

The profiling confirmed that one of the largest contributors to overall latency was **Phase 2 Risk Scoring**.

Unlike the lightweight assessment performed during payment intent creation, Phase 2 executes during payment authorization and performs a far more comprehensive evaluation before a payment can proceed.

Improving this stage therefore became the next optimization target.

---

# Objective

Reduce the latency of Phase 2 risk scoring without reducing fraud detection quality.

At the same time, remove unnecessary synchronous work from the payment processing pipeline so that only operations required for transactional correctness remain on the critical execution path.

---

# Previous Pipeline

Before optimization, payment processing executed the following sequence synchronously:

```text
Request
    │
    ▼
System Flag Validation
    │
    ▼
Redis Lock
    │
    ▼
Global Payment Lookup
    │
    ▼
Risk Scoring
    │
    ▼
Merchant Validation
    │
    ▼
Gateway Processing
    │
    ▼
Database Transaction
    │
    ▼
Actor Metrics
    │
    ▼
Audit Logging
    │
    ▼
Kafka Publishing
    │
    ▼
Return Response
```

Although functionally correct, several operations unnecessarily extended request latency.

---

# Optimization 1 — Risk Scoring Improvements

The largest optimization focused on the `RiskScoringService`.

Rather than performing multiple independent lookups throughout the scoring process, the service was redesigned to operate with significantly less database interaction while producing the same final risk assessment.

The processing pipeline now supplies preloaded information directly into the risk engine, reducing repeated queries and allowing the scoring process to focus primarily on rule evaluation instead of data retrieval.

Risk scoring still evaluates:

- transaction context;
- payment actor;
- payment amount;
- currency;
- IP address;
- country;
- device fingerprint;
- merchant history.

However, the service now executes with considerably less overhead.

---

# Optimization 2 — Selective Database Reads

The payment lookup was also reduced.

Instead of loading entire relational objects, the query now retrieves only the fields required for payment execution.

For example:

```text
Payment Intent

Merchant

Current Risk Score
```

are fetched using selective projections rather than full entity loading.

Reducing unnecessary columns lowers serialization cost and decreases query execution time.

---

# Optimization 3 — Cached Country Resolution

IP geolocation previously represented another external dependency during processing.

Country resolution is now retrieved through a cache-first approach.

```text
IP Address

↓

Cache Lookup

↓

Country
```

Repeated requests originating from the same IP avoid repeated geolocation work, reducing latency for both the payment service and the risk engine.

---

# Optimization 4 — Atomic Status Transition

The payment intent state transition was simplified.

Previously, multiple operations were required to verify the current state before updating the payment intent.

The implementation now performs a single conditional update.

```text
UPDATE PaymentIntent
WHERE status IN (...)

↓

Rows Updated?

↓

Continue or Reject
```

Because Redis already guarantees single-flight processing, this removes redundant validation logic while preserving correctness.

---

# Optimization 5 — Atomic Balance Updates

Merchant balance updates previously required:

```text
Read Balance

↓

Calculate Balance

↓

Update Balance

↓

Write Balance Log
```

The balance update now relies on an atomic database increment.

```text
Increment Balance

↓

Read Updated Value

↓

Derive Previous Balance

↓

Write Balance Log
```

This eliminates an additional database read while maintaining accurate audit records.

---

# Optimization 6 — Background Event Publishing

Publishing Kafka events is no longer part of the synchronous request lifecycle.

Instead of waiting for event publication before responding, payment events are dispatched asynchronously after the database transaction successfully completes.

```text
Database Commit

↓

Return Response

↓

Publish Kafka Events
```

Failures to publish events are still logged for investigation but no longer delay customer responses.

---

# Optimization 7 — Asynchronous Actor Metrics

Updating payment actor statistics is also performed asynchronously.

Previously:

```text
Process Payment

↓

Update Actor Metrics

↓

Return Response
```

Now:

```text
Process Payment

↓

Return Response

↓

Update Actor Metrics
```

This keeps analytics consistent while reducing synchronous execution time.

---

# Optimization 8 — Fine-Grained Performance Instrumentation

Additional timing checkpoints were introduced throughout the payment pipeline.

Each major stage is now measured independently.

Current instrumentation includes:

- system flag validation;
- Redis lock acquisition;
- global payment lookup;
- country lookup;
- Phase 2 risk scoring;
- payment gateway execution;
- database transaction;
- background dispatch.

Rather than reporting only total request duration, the service now records the execution time of every stage.

Example output:

```text
flagCheck

lockAcquire

globalLookup

countryLookup

riskScoring

gatewayCall

dbTransaction

backgroundDispatched
```

This makes performance regressions immediately visible and significantly simplifies future optimization work.

---

# Updated Processing Pipeline

```text
Request
    │
    ▼
System Flag Validation
    │
    ▼
Redis Lock
    │
    ▼
Selective Global Lookup
    │
    ▼
Cached Country Lookup
    │
    ▼
Optimized Phase 2 Risk Scoring
    │
    ▼
Merchant Validation
    │
    ▼
Gateway Processing
    │
    ▼
Atomic Database Transaction
    │
    ├── Conditional Status Update
    ├── Create Transaction
    ├── Atomic Balance Increment
    └── Balance Log
    │
    ▼
Return Response
    │
    ├── Update Actor Metrics
    └── Publish Kafka Events
```

---

# Engineering Decisions

Several architectural principles guided these changes.

### Keep financial operations transactional

The payment transaction, balance updates and status transitions remain inside a single database transaction to preserve consistency.

---

### Move non-critical work off the critical path

Operations that are not required before responding to the client are executed asynchronously.

This includes:

- actor metrics;
- Kafka event publication.

---

### Reduce unnecessary database work

Queries now retrieve only the fields required for payment execution.

Atomic operations replace multiple read-modify-write sequences wherever possible.

---

### Measure before optimizing

Instrumentation remains a core part of the architecture.

Every major stage now exposes timing information, allowing future performance work to focus on measurable bottlenecks rather than assumptions.

---

# Analysis

The largest improvement in this iteration was not introducing new infrastructure, but simplifying existing execution paths.

Reducing unnecessary database reads, minimizing synchronous work, and restructuring the risk engine significantly decreased the amount of work performed before a payment response can be returned.

Equally important, the payment service now produces detailed timing information for every stage of execution.

Future optimization efforts can therefore be based on precise measurements instead of aggregate request latency.

---

# Key Takeaways

This iteration focused on improving efficiency while preserving transactional guarantees.

The primary improvements include:

- optimizing Phase 2 Risk Scoring;
- reducing database queries through selective loading;
- caching IP country resolution;
- replacing multi-step updates with atomic operations;
- moving event publication to the background;
- moving actor metric updates off the critical path;
- introducing comprehensive stage-level performance instrumentation.

Together, these changes reduce synchronous processing overhead while maintaining payment correctness, auditability, and fraud detection accuracy.

---

# Next Step

Engineering Insight #008 will benchmark the optimized payment processing pipeline to quantify the impact of these architectural changes.

With detailed stage-level metrics now available, the next objective is to compare latency before and after the optimization and determine which components remain the dominant contributors to request execution time.
