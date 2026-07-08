# Engineering Insight #002

**Title:** Replacing Merchant-Wide Locks with Request-Level Idempotency

**Date:** 2026-07-08

---

## Background

Engineering Insight #001 identified merchant-wide distributed locking as the primary cause of contention during concurrent payment intent creation.

The original implementation prevented duplicate execution by serializing every `POST /create-intent` request for a merchant.

Although functionally correct, the design unnecessarily blocked independent payment requests and significantly reduced throughput.

The goal was to preserve duplicate request protection without sacrificing concurrency.

---

## Previous Design

Each payment intent creation attempted to acquire a Redis lock using the merchant as the synchronization boundary.

```text
intent:create:{merchantId}:{environment}
```

Only one request could create a payment intent for a merchant at a time.

Under concurrent load:

- unrelated payment requests competed for the same lock;
- requests were rejected while the lock was held;
- throughput decreased as concurrency increased.

The locking strategy treated every payment intent as a shared resource, even though each request produced an independent payment intent.

---

## New Design

The distributed lock was removed from the payment intent creation flow.

Instead, duplicate request protection is now implemented through request-level idempotency.

Each client includes an `Idempotency-Key` header when creating a payment intent.

```http
POST /create-intent

Idempotency-Key: 7e4f5d7d-6d8f-4f0d-a8c7-9f5bfa2c71bd
```

The request is processed by the idempotency service before the business logic executes.

```ts
await idempotency.process(
    idempotencyKey,
    `${merchantId}:${environment}:create-intent`,
    "/create-intent",
    req.body,
    () => paymentService.createPaymentIntent(...)
);
```

The first request executes normally and its response is persisted.

Subsequent requests with the same idempotency key and identical payload return the previously stored response instead of executing the payment creation logic again.

---

## Why Idempotency Is a Better Fit

Creating a payment intent is not a shared mutable operation.

Each successful request creates a new resource with its own identifier.

The actual problem to solve is duplicate execution caused by client retries, not concurrent execution by different clients.

Request-level idempotency addresses this requirement directly by guaranteeing that the same logical request is processed only once while allowing unrelated requests to execute concurrently.

This aligns more closely with how payment APIs are expected to behave in production environments.

---

## Architectural Benefits

The new design provides several improvements:

- Independent payment intent requests can execute concurrently.
- Safe retries are preserved through idempotency.
- Merchant-wide lock contention is eliminated.
- The payment creation flow scales horizontally without artificial serialization.
- The implementation more closely follows industry practices for payment APIs.

---

## Trade-offs

The new approach introduces one additional responsibility.

Clients must generate an `Idempotency-Key` for every payment intent request and reuse the same key when retrying that request.

This shifts duplicate request detection from infrastructure-level locking to application-level request semantics while keeping the payment flow safe and deterministic.

---

## Implementation Status

- ✅ Merchant-wide distributed lock removed.
- ✅ Request-level idempotency implemented.
- ✅ Duplicate request handling migrated to the idempotency service.
- ✅ Client support for `Idempotency-Key` added.

---

## Next Step

With the architectural migration complete, the next step is to benchmark the updated implementation and compare it against the original locking strategy.

Engineering Insight #003 documents the performance measurements, including throughput, latency, and concurrent request behaviour after adopting request-level idempotency.
