# Sharding Strategy

## Overview

As transaction volume increases, centralized storage systems often become bottlenecks.

EBINUM explores deterministic sharding strategies to distribute state across multiple partitions while maintaining predictable routing behavior.

---

# Design Objectives

The sharding strategy aims to:

- distribute workload evenly
- reduce contention
- improve scalability
- simplify request routing
- minimize coordination overhead

---

# Deterministic Routing

Current routing exploration uses a deterministic hash-based approach.

```text
shard = hash(entity_id) % total_shards
```

This ensures that the same entity is consistently routed to the same partition.

---

# Benefits

## Predictable Routing

Requests can be routed without querying a centralized routing table.

---

## Reduced Lookup Cost

Routing decisions are computationally inexpensive.

---

## State Locality

Related entity state remains co-located.

Examples:

- merchant transactions
- customer activity
- ledger history

---

## Horizontal Scalability

Additional storage capacity can be introduced through additional shards.

---

# Tradeoffs

## Hot Shards

Popular entities may generate uneven traffic patterns.

Potential effects:

- higher latency
- increased contention
- resource imbalance

---

## Resharding Complexity

As shard counts increase, data redistribution becomes necessary.

Challenges include:

- state migration
- downtime avoidance
- consistency preservation

---

## Cross-Shard Operations

Operations involving multiple entities may require coordination across partitions.

Examples:

- transfers
- reconciliation
- aggregate reporting

---

# Future Exploration Areas

Potential future directions include:

- consistent hashing
- virtual nodes
- adaptive shard allocation
- workload-aware routing
- dynamic rebalancing

---

# Research Questions

Current architectural questions include:

- How can hot shard behavior be detected early?
- What level of routing determinism is optimal?
- When does resharding become economically justified?
- How can cross-shard coordination be minimized?

These questions continue to shape infrastructure experimentation within the project.
