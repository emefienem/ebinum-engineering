# Explainability Tradeoffs

## Overview

Financial institutions increasingly rely on machine learning systems for fraud detection and transaction monitoring.

However, greater predictive complexity often comes at the cost of reduced interpretability.

This document explores the tradeoff between explainability and throughput in real-time payment infrastructure.

---

# The Core Tension

Two competing objectives frequently emerge:

```text
More Explainability
        ↓
More Computation
        ↓
Higher Latency
```

and

```text
Lower Latency
        ↓
Less Computation
        ↓
Reduced Transparency
```

The challenge is finding a practical balance.

---

# Why Explainability Matters

Risk systems influence operational decisions including:

- payment approval
- payment review
- payment rejection

When a transaction is flagged, operators often require an explanation.

Questions include:

- Why was this transaction flagged?
- Which factors contributed most?
- What triggered the review process?

Without explainability, these questions become difficult to answer.

---

# Problems With Black-Box Models

Highly complex models may produce accurate predictions while providing limited transparency.

Operational challenges include:

- difficult investigations
- reduced analyst trust
- regulatory concerns
- audit complexity

In financial environments, accuracy alone is often insufficient.

---

# Explainability Costs

Generating explanations introduces overhead.

Examples include:

## Additional Feature Evaluation

Each signal requires computation.

Examples:

- velocity checks
- device history lookups
- reputation lookups

---

## Additional Storage

The platform stores:

- triggered signals
- explanations
- historical snapshots

This increases persistence requirements.

---

## Additional Coordination

Explainability frequently requires data from multiple services.

Examples:

```text
Device Service
IP Service
Transaction Service
Actor Service
```

Cross-service lookups increase coordination overhead.

---

# Real-Time Constraints

Payment systems operate under strict latency expectations.

Users expect near-immediate responses.

As transaction volume increases, even lightweight explanation logic can become expensive.

The challenge is preserving transparency without introducing unacceptable latency.

---

# Current Architectural Approach

EBINUM currently uses:

- deterministic rules
- weighted signals
- stored explanations

This provides:

- low complexity
- reproducibility
- auditability

while maintaining predictable execution behavior.

---

# Future Questions

Areas of exploration include:

- feature attribution methods
- explainable clustering
- interpretable anomaly detection
- scalable explanation generation

The long-term objective is to improve predictive capability without sacrificing transparency.
