# ADR-002: Separate relational ledger integrity from multi-Region item replication

- **Status:** Proposed — critical business validation required
- **Date:** 3 November 2026
- **Decision owners:** Payments Domain Owner, Chief Data Architect, SRE, Risk and Compliance
- **Context:** The requirement states RTO 30 minutes and near-zero RPO for committed payment transactions. Teams disagree on database strategy and active-active design.

## Decision

1. Use a single-writer relational ledger boundary per legal entity/partition, deployed Multi-AZ in its approved primary Region, where ACID multi-row journal invariants are required.
2. Use Aurora PostgreSQL where the bank's schema, extensions, performance and operations fit; use Aurora Global Database as a warm-standby cross-Region recovery mechanism only with its asynchronous replication and seconds-level typical RPO explicitly accepted.
3. Use DynamoDB Global Tables MRSC only for data models whose consistency invariant can be enforced at the **item level** (for example, a canonical per-payment workflow/commit record or idempotency record), and only where its supported Region topology and latency fit.
4. Do not use DynamoDB MRSC as a drop-in relational ledger: it requires exactly three Regions and does not support transaction APIs.
5. If the bank interprets near-zero RPO as zero loss of every committed multi-row ledger posting after complete Region loss, the current design is **not yet proven compliant**. A synchronous cross-Region consensus/ledger design must be selected, prototyped and performance-tested, or the business must formally accept a bounded RPO exception.

## Decision drivers

- No loss or duplication of committed payments.
- Double-entry balance integrity and atomic posting.
- RTO no more than 30 minutes.
- Approved data residency and country restrictions.
- Latency impact on payment authorisation.
- Operational simplicity and recoverability.
- Avoid unsupported claims about managed-service guarantees.

## Options considered

### Option A — Aurora Multi-AZ + Aurora Global Database
**Benefits:** Relational ACID model, familiar SQL, Multi-AZ regional resilience, managed operations, physical cross-Region replication, fast promotion.

**Limitations:** Global replication is asynchronous. AWS documents RPO typically measured in seconds during unplanned failover; actual lag varies. Managed planned switchover synchronises before promotion but is not a catastrophic-failure guarantee.

**Decision:** Selected for relational ledger where seconds-level cross-Region RPO is formally accepted; not sufficient by itself for a zero-RPO Region-loss requirement.

### Option B — DynamoDB Global Tables MREC
**Benefits:** Multi-Region availability and low-latency local writes; broad Region availability.

**Limitations:** Asynchronous replication; typically seconds or less but not zero. Concurrent updates use last-writer-wins behaviour. Transaction APIs are atomic only in the source Region and are not replicated as a unit.

**Decision:** Rejected for authoritative zero-RPO financial commit.

### Option C — DynamoDB Global Tables MRSC
**Benefits:** Synchronous replication to at least one other Region before write success; strongly consistent reads; RPO zero for item changes.

**Limitations:** Exactly three Regions; supported Region sets are constrained; cross-Region round-trip increases latency; no TransactWriteItems/TransactGetItems; item-level replication does not create multi-item ACID transaction semantics. Witness does not serve reads/writes.

**Decision:** Accepted for suitable per-item payment state and idempotency records only, not as a general relational ledger.

### Option D — Distributed SQL / consensus ledger across Regions
**Benefits:** Potential to provide atomic transaction semantics with quorum-based synchronous replication, subject to product/architecture capabilities.

**Limitations:** Cross-Region commit latency, quorum availability, partition trade-offs, operational complexity, licensing/vendor dependency and data-residency challenges. Requires workload-specific proof and support model.

**Decision:** Conditional option if the bank insists on zero RPO for multi-row ledger commits. Do not select without a measured prototype, legal review, failure testing and exit strategy.

### Option E — Active-active relational writers in two Regions
**Benefits:** Local write availability and potentially lower client latency.

**Limitations:** Conflict resolution, double posting, sequence allocation, balance races, split-brain and reconciliation complexity. Asynchronous replication is not a safe ledger consistency model.

**Decision:** Rejected for the same ledger partition absent a proven distributed consensus and partition ownership design.

## Operational model

- Each payment has an immutable unique ID, idempotency key, entity ID, ledger reference and correlation ID.
- Ledger posts use a single authoritative writer and enforce double-entry, uniqueness and balance invariants transactionally.
- Transactional outbox ensures publication follows commit without a dual-write gap.
- Unknown outcomes are resolved by querying the authoritative reference, not by blind retry.
- On failover, fence old writer using epoch/lease and prevent split-brain.
- Reconcile internal journal with core banking and external scheme acknowledgements before releasing held transactions.
- Monitor Aurora Global DB lag and MRSC latency/availability; alerts do not imply a strict bound on data loss.

## Risks

| Risk | Treatment |
|---|---|
| Requirement owner assumes Aurora Global DB is zero RPO | Publish explicit RPO limitation and obtain signed acceptance or fund alternative |
| MRSC latency breaches payment p99 | Benchmark Region pairs and model client geography |
| MRSC model cannot enforce multi-row invariant | Keep ledger in ACID store; use MRSC only for aggregate/record boundary |
| Residency prevents third Region | Country-specific design; do not replicate restricted data |
| Quorum loss blocks strong writes | Define degraded mode, queue intake or reject safely; never silently weaken consistency |

## Review triggers

- Country/regulator rejects cross-Region data placement.
- Bank changes RPO from “near zero” to strict zero or changes tolerance.
- Payment p99 latency is not met.
- MRSC service capabilities or supported Region sets change.
- Aurora replication lag or DR exercise fails target.
- The bank's core ledger remains authoritative and changes integration semantics.
