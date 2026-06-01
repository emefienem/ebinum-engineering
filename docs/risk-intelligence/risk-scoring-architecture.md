# Risk Scoring Architecture

## Overview

EBINUM currently implements a heuristic risk intelligence engine designed to provide interpretable transaction risk assessments while maintaining low-latency execution.

The objective is not only to produce a risk score, but to generate a transparent explanation describing why that score was produced.

Rather than relying on a black-box model, the system decomposes risk into multiple independent signal categories.

---

# Design Principles

The current architecture follows four principles:

1. Explainability First
2. Low-Latency Evaluation
3. Deterministic Scoring
4. Audit-Friendly Outputs

Every risk assessment can be reconstructed and inspected after execution.

---

# Risk Evaluation Pipeline

```text
Transaction Request
        ↓
Amount Analysis
        ↓
Velocity Analysis
        ↓
Device Analysis
        ↓
IP Reputation Analysis
        ↓
Location Analysis
        ↓
Actor History Analysis
        ↓
Weighted Score Calculation
        ↓
Risk Classification
        ↓
Recommendation Generation
```

---

# Signal Categories

## Amount Signals

The system evaluates transaction size relative to predefined thresholds.

Examples:

- normal_amount
- moderate_amount
- elevated_amount
- large_amount

The system also compares transaction size against historical user behavior.

Example:

```text
Current Amount: €1,500

Historical Average: €250

Spike Ratio: 6x
```

This may generate:

```text
amount_spike_extreme
```

---

## Velocity Signals

Velocity analysis measures transaction frequency.

Examples:

- payment attempts in last hour
- payment attempts in last 24 hours

Possible outputs:

```text
normal_velocity
elevated_velocity_hour
high_velocity_hour
```

---

## Device Trust Signals

Device intelligence evaluates whether the transaction originates from a previously observed device.

Examples:

```text
known_device_for_actor
new_device_for_actor
trusted_device
very_new_device
```

The system incorporates:

- device age
- trust status
- confidence scores

---

## IP Reputation Signals

External reputation services are used to enrich risk evaluation.

Examples:

```text
tor_network
vpn_proxy
datacenter_ip
high_fraud_score
known_good_ip
```

The goal is to distinguish trusted infrastructure from suspicious network origins.

---

## Location Signals

Location analysis evaluates:

- customer geography
- merchant geography
- country mismatch patterns

Examples:

```text
normal_location
country_mismatch
elevated_risk_country
```

---

## Actor History Signals

Historical transaction behavior contributes to trust estimation.

Examples:

```text
trusted_actor
low_success_rate
moderate_success_rate
blocked_actor
```

The system evaluates:

- success rates
- completed transaction history
- account status

---

# Risk Score Generation

Each signal contributes through a weighted scoring model.

Example weights:

```text
Amount          20%
Velocity        25%
Device          15%
IP              15%
Location        10%
Actor History   15%
```

The final score is calculated by aggregating weighted signal contributions.

---

# Explainable Outputs

Example response:

```text
Risk Score: 37/100

Risk Level: MEDIUM

Recommendation: REVIEW

Signals:

- New Device
- Elevated Amount
- Country Mismatch
```

Unlike opaque risk models, the output explicitly exposes contributing factors.

---

# Auditability

Every risk assessment stores:

- risk score
- signal breakdown
- triggered rules
- contextual metadata
- recommendation

This allows historical reconstruction of risk decisions for operational review.

---

# Current Limitations

The current engine is heuristic-based.

Limitations include:

- manually defined thresholds
- static weights
- limited behavioral adaptation

Future work explores more sophisticated explainable learning approaches while preserving transparency.
