# Engineering Insight #004

**Title:** Removing Synchronous Work from the Payment Path

**Date:** 2026-07-16

---

## Background

Engineering Insight #003 confirmed that replacing merchant-wide distributed locking with request-level idempotency eliminated lock contention and improved concurrent request handling.

Although the migration removed synchronization bottlenecks, benchmark results still showed significantly higher latency than the target.

| Metric          |    Value |
| --------------- | -------: |
| Average Latency |   8.07 s |
| P95 Latency     |  12.25 s |
| Target          | < 500 ms |

The benchmark demonstrated that the remaining bottleneck was no longer concurrency but the amount of work performed before returning a response.

---

## Investigation

Profiling the payment intent creation pipeline showed that the endpoint executed several expensive operations synchronously before responding to the client.

These included:

- payment actor creation and updates;
- risk scoring;
- payment enrichment;
- device linking;
- webhook dispatch;
- multiple cache operations.

Many of these tasks improve payment intelligence but are not required to successfully create a payment intent.

Keeping them on the critical request path unnecessarily increased end-to-end latency.

---

## Previous Design

Before the optimization, nearly every operation completed before the API returned a response.

```text
Create Intent Request
        │
        ▼
Merchant Lookup
        │
        ▼
Payment Actor Resolution
        │
        ▼
Create Payment Intent
        │
        ▼
Risk Scoring
        │
        ▼
Payment Enrichment
        │
        ▼
Device Linking
        │
        ▼
Cache Updates
        │
        ▼
Webhook Dispatch
        │
        ▼
Return Response
```

The client waited for the entire processing pipeline to complete, even though several steps were independent of payment intent creation.

---

## Optimization

The payment creation flow was redesigned so that only operations required for correctness remain synchronous.

After the payment intent is successfully persisted, the response is immediately returned to the client.

All non-critical work is executed asynchronously.

Examples include:

- risk scoring;
- payment enrichment;
- device linking;
- webhook delivery;
- cache refreshes.

The implementation uses fire-and-forget background execution.

```ts
this.runRiskScoringAndEnrichment(...).catch(...);

this.linkDeviceToActor(...).catch(...);

WebhookService.sendWebhook(...).catch(...);
```

In addition, merchant retrieval was optimized by introducing caching and parallelizing independent operations using `Promise.all()`.

---

## Before

```text
Request
   │
   ▼
Merchant Lookup
   │
Actor Resolution
   │
Create Payment Intent
   │
Risk Scoring
   │
Payment Enrichment
   │
Device Linking
   │
Cache Updates
   │
Webhook Dispatch
   │
Return Response
```

---

## After

```text
Request
   │
   ▼
Merchant Lookup (Cached)
        │
        ▼
Actor Resolution
        │
        ▼
Create Payment Intent
        │
        ▼
Return Response
        │
        ├────────► Risk Scoring
        │
        ├────────► Payment Enrichment
        │
        ├────────► Device Linking
        │
        ├────────► Cache Updates
        │
        └────────► Webhook Dispatch
```

---

## What Was Optimized

### Merchant Retrieval

- Cached merchant information.
- Loaded tier limits and velocity limits concurrently.
- Removed unnecessary database lookups.

### Parallel Execution

Independent operations now execute concurrently instead of sequentially.

Examples include:

- merchant lookup;
- IP geolocation;
- configuration retrieval.

### Background Processing

The following operations were removed from the synchronous request path:

- risk scoring;
- payment enrichment;
- device linking;
- webhook delivery;
- actor metric updates.

### Caching

Additional caching was introduced for:

- merchants;
- payment actors;
- devices;
- payment intents.

This reduces repeated database access for subsequent requests.

---

## Architectural Impact

The payment service now separates **payment creation** from **payment enrichment**.

The synchronous path is responsible only for creating a valid payment intent.

Everything related to analytics, intelligence, enrichment, and background maintenance executes independently after the client has already received a successful response.

This significantly reduces coupling between the core payment flow and auxiliary services while improving scalability.

---

## Benchmark Results

After introducing these optimizations, the benchmark produced the following results.

| Metric              |   Value |
| ------------------- | ------: |
| Total Requests      |      20 |
| Successful Requests |      20 |
| Failed Requests     |       0 |
| Success Rate        |    100% |
| Average Latency     |  8.07 s |
| P95 Latency         | 12.25 s |
| Maximum Latency     | 13.50 s |

Compared to Engineering Insight #003:

| Metric          | Insight #003 | Insight #004 |
| --------------- | -----------: | -----------: |
| Average Latency |      10.98 s |       8.07 s |
| P95 Latency     |      12.12 s |      12.25 s |
| Success Rate    |         100% |         100% |

Average request latency decreased by approximately **27%**, indicating that removing synchronous work reduced the time spent on the critical request path.

However, the benchmark still failed the performance target.

```text
Target:
P95 < 500 ms

Measured:
P95 = 12.25 s
```

---

## Analysis

The optimization successfully reduced the amount of work executed before returning a response.

The remaining latency is concentrated in the core payment creation pipeline rather than enrichment tasks.

Benchmark metrics also show that nearly all request time is spent waiting for the server to respond.

```
http_req_waiting (avg): 8.07 s
```

This suggests that additional optimization should focus on:

- database operations;
- shard routing;
- payment actor resolution;
- velocity limit checks;
- synchronous business logic still executed before persistence.

---

## Key Finding

The largest latency contributor was no longer lock contention but synchronous processing.

Moving enrichment work into background execution shortened the critical request path while preserving the correctness of payment creation.

The architecture now follows a clearer separation between transaction processing and transaction enrichment.

---

## Next Step

The next phase focuses on profiling the remaining synchronous business logic inside the payment creation pipeline.

The objective is to identify the most expensive database queries and service calls responsible for the remaining server-side execution time and continue reducing end-to-end latency.
