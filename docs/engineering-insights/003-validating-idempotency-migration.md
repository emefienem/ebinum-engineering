# Engineering Insight #003

**Title:** Validating the Idempotency Migration

**Date:** 2026-07-13

---

## Background

After replacing the merchant-wide distributed lock with request-level idempotency (Engineering Insight #002), the `POST /create-intent` endpoint was benchmarked again using the same workload.

The objective was to verify that removing lock contention improved concurrent request handling without introducing regressions.

---

## Benchmark Setup

Tool: **k6**

Scenario:

- Virtual Users: **2**
- Duration: **90 seconds**
- Endpoint: `POST /create-intent`

---

## Results (After)

| Metric              |   Value |
| ------------------- | ------: |
| Total Requests      |      16 |
| Successful Requests |      16 |
| Failed Requests     |       0 |
| Success Rate        |    100% |
| Average Latency     | 10.98 s |
| P95 Latency         | 12.12 s |
| Maximum Latency     | 12.20 s |

All requests completed successfully.

No requests returned:

```text
Intent creation is locked
```

The merchant-wide contention observed in the previous benchmark was eliminated.

---

## Comparison

| Metric       | Before |   After |
| ------------ | -----: | ------: |
| Success Rate |  28.1% |    100% |
| Lock Errors  |     23 |       0 |
| P95 Latency  |  ~14 s | 12.12 s |

The architectural migration successfully solved the concurrency issue.

However, overall request latency remained significantly higher than the target threshold.

The benchmark still failed the configured performance objective:

```text
p(95) < 500 ms
```

Measured:

```text
P95 = 12.12 s
```

---

## Analysis

The benchmark demonstrates that lock contention was not the primary contributor to request latency.

Removing synchronization delays improved correctness and concurrency, but each request still spends approximately 10–12 seconds executing business logic.

This indicates the remaining latency originates elsewhere in the request pipeline.

Potential contributors include:

- external network calls;
- database operations;
- synchronous risk scoring;
- webhook execution;
- cache lookups;
- unnecessary sequential operations.

Further investigation is required before additional optimizations can be made.

---

## Key Finding

The migration to request-level idempotency achieved its intended goal:

- duplicate request protection is preserved;
- concurrent requests no longer block one another;
- lock-related failures have been eliminated.

The remaining bottleneck is execution time rather than synchronization.

---

## Next Step

The next phase focuses on latency profiling.

Timing instrumentation already exists throughout the payment intent creation pipeline, making it possible to identify the most expensive operations and optimize them individually.

The objective is to reduce end-to-end request latency while maintaining the concurrency improvements introduced by the idempotency migration.
