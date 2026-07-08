# Engineering Insight #001

**Title:** Distributed Lock Contention vs Idempotency

**Date:** 2026-07-08

---

## Background

The `POST /create-intent` endpoint creates a new payment intent for a merchant.

To prevent duplicate requests, the endpoint originally used a distributed Redis lock that serialized payment intent creation for each merchant.

The lock key was:

```text
intent:create:{merchantId}:{environment}
```

---

## Benchmark Setup

Tool: **k6**

Scenario:

- Virtual Users: **2**
- Duration: **90 seconds**
- Endpoint: `POST /create-intent`

---

## Results (Before)

| Metric              |       Value |
| ------------------- | ----------: |
| Total Requests      |          32 |
| Successful Requests |           9 |
| Failed Requests     |          23 |
| Success Rate        |       28.1% |
| P95 Latency         | ~14 seconds |

Most failed requests returned:

```text
Intent creation is locked
```

---

## Investigation

The distributed lock was scoped to the merchant rather than the request.

```ts
intent:create:{merchantId}:{environment}
```

As a result:

- every payment intent creation waited for the previous one to complete;
- unrelated payment requests competed for the same lock;
- throughput was artificially limited;
- latency increased as requests queued behind the lock.

---

## Root Cause

Payment intent creation is not a shared mutable resource.

Each request creates a separate payment intent and therefore does not require merchant-wide serialization.

The real requirement was preventing duplicate execution of the **same client request**, not blocking all concurrent requests.

---

## Engineering Decision

The investigation concluded that merchant-wide locking was unnecessary for payment intent creation.

The proposed solution was to replace the distributed lock with request-level idempotency, allowing independent payment requests to execute concurrently while still preventing duplicate execution of client retries.

The implementation of this redesign is covered in Engineering Insight #002.

---

## Expected Benefits

- Eliminate unnecessary lock contention.
- Improve concurrent request throughput.
- Reduce request latency.
- Preserve safe retry semantics through idempotency.
- Align the implementation with payment industry best practices.

---

## Follow-up

A second benchmark will be executed after migrating to idempotency to compare:

- throughput
- success rate
- latency
- concurrent behaviour

against the original locking implementation.
