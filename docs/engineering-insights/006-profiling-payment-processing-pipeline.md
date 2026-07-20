# Engineering Insight #006

**Title:** Profiling the Payment Processing Pipeline

**Date:** 2026-07-19

---

## Background

Engineering Insight #005 focused on reducing latency during payment intent creation by eliminating unnecessary infrastructure overhead.

System flags were moved into memory, idempotency became Redis-first, and shared infrastructure components were reused across requests.

These changes reduced average create-intent latency from over **7 seconds** to approximately **1.5 seconds**, shifting the primary performance bottleneck to the next stage of the payment lifecycle:

**Payment Processing.**

Unlike payment intent creation, processing a payment performs considerably more work before a response can be returned.

It must:

- validate the payment intent;
- execute Phase 2 fraud scoring;
- select the payment rail;
- process the payment gateway;
- write multiple database records;
- update merchant balances;
- publish domain events.

Understanding where time was being spent became the next priority.

---

## Objective

Before optimizing the payment processing pipeline, detailed instrumentation was introduced to identify where latency originated.

Rather than treating payment processing as a single operation, the goal was to measure each stage independently.

This would allow future optimizations to target the actual bottlenecks instead of relying on assumptions.

---

## Investigation

Performance checkpoints were added throughout the processing pipeline.

Examples include:

- system flag validation;
- distributed lock acquisition;
- payment intent lookup;
- Phase 2 risk scoring;
- payment gateway execution;
- database transaction;
- merchant balance updates;
- Kafka event publishing.

Instead of logging only total request duration, individual stages could now be measured independently.

---

## Initial Benchmark

A dedicated benchmark was created to measure only the payment processing endpoint.

Because payment processing depends on a valid payment intent, every benchmark iteration executed the complete setup sequence:

```text
Create Payment Intent

↓

Confirm Payment Intent

↓

Measure Process Payment
```

The setup requests were executed only to prepare valid test data.

Latency measurements were recorded exclusively for the **Process Payment** endpoint.

---

## Benchmark Results

Initial measurements revealed that payment processing remained significantly slower than payment intent creation.

| Metric              |      Result |
| ------------------- | ----------: |
| Successful Requests |          10 |
| Average Latency     | **11.22 s** |
| Median Latency      |  **9.25 s** |
| P90 Latency         | **15.43 s** |
| P95 Latency         | **16.32 s** |
| Maximum Latency     | **17.21 s** |

The benchmark clearly exceeded the current performance target.

---

## Findings

Unlike payment intent creation, payment processing executes a much larger synchronous workflow before returning a response.

Several operations occur sequentially.

### Phase 2 Risk Scoring

The payment service performs a second, more comprehensive fraud analysis before authorizing the payment.

This stage includes:

- behavioral analysis;
- device evaluation;
- IP analysis;
- payment actor history;
- fraud rule execution.

Unlike the lightweight risk estimate used during intent creation, this operation performs substantially more computation and database interaction.

---

### Transaction Processing

Payment execution occurs inside a database transaction.

Within a single transaction the service:

- validates payment state;
- creates the transaction record;
- updates the payment intent;
- updates merchant balances;
- writes balance logs.

Although this guarantees consistency, it also increases synchronous execution time.

---

### Merchant Balance Updates

Successful payments immediately update merchant balances.

The processing pipeline performs:

```text
Read Balance

↓

Calculate New Balance

↓

Update Merchant

↓

Write Balance Log
```

These operations occur before the response is returned.

---

### Event Publishing

After payment completion, payment events are published for downstream consumers.

Examples include:

- payment succeeded;
- payment failed.

Although Kafka publishing occurs after core payment processing, failures are still logged and handled before the request fully completes.

---

## Current Pipeline

```text
Request
    │
    ▼
System Flag Validation
    │
    ▼
Distributed Lock
    │
    ▼
Payment Intent Lookup
    │
    ▼
Phase 2 Risk Scoring
    │
    ▼
Merchant Validation
    │
    ▼
Payment Gateway
    │
    ▼
Database Transaction
    │
    ├── Create Transaction
    ├── Update Payment Intent
    ├── Update Merchant Balance
    └── Write Balance Log
    │
    ▼
Publish Events
    │
    ▼
Return Response
```

---

## Analysis

The benchmark demonstrates that payment processing is substantially more complex than payment intent creation.

Where create-intent primarily prepares a payment for execution, process-payment performs the actual financial workflow.

As a result, additional latency is expected.

However, the benchmark also indicates that the current implementation performs too much work synchronously before responding.

At this stage it is not yet clear which individual component contributes the largest portion of the overall latency.

The next step is therefore to introduce stage-level timing throughout the processing pipeline, allowing every major operation to be measured independently.

---

## Key Finding

The payment processing endpoint currently averages more than **11 seconds** under benchmark conditions.

This confirms that the primary performance bottleneck has shifted away from payment creation and into the payment execution pipeline.

Before introducing architectural changes, detailed instrumentation is required to accurately identify the most expensive synchronous operations.

---

## Next Step

Engineering Insight #007 will focus on instrumenting every stage of the payment processing pipeline.

This includes measuring:

- distributed lock acquisition;
- payment intent retrieval;
- Phase 2 risk scoring;
- payment gateway execution;
- database transaction duration;
- merchant balance updates;
- event publication.

With stage-level metrics available, future optimizations can target the largest contributors to latency while preserving transactional correctness and payment consistency.
