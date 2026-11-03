# Data Architecture, Transaction Integrity and Disaster Recovery

## 1. Data classification and placement

Every data set must have an owner, classification, legal entity, residency rule, retention period, encryption/key policy, recovery tier and approved replication destinations.

| Data class | Example | Preferred store | Replication / recovery |
|---|---|---|---|
| Authoritative financial ledger | Posted debit/credit journal | Aurora PostgreSQL (regional ACID) or approved core ledger | Multi-AZ synchronous regional durability; cross-Region Aurora Global DB is asynchronous and has seconds-level RPO typical, not zero |
| Payment workflow/idempotency item | Payment command state, request key, status | DynamoDB where access model fits | MRSC for suitable single-item records; zero-RPO item replication with 3-Region topology; no transaction APIs |
| Event stream | Payment lifecycle events | MSK / EventBridge / SQS depending semantics | Replication and replay strategy per event type; never assume delivery is exactly once end-to-end |
| Documents / statements | Reports, evidence, files | S3 | Versioning, Object Lock where required, cross-Region copy only if approved |
| Analytics | Operational and regulatory reporting | S3 data lake / governed analytics | Curated, minimised, residency-aligned copies; separate from posting path |
| Secrets / keys | DB credentials, signing keys | Secrets Manager / KMS | Multi-Region replication only where policy approves; key disable/deletion controls |

## 2. Transaction integrity pattern

A payment state machine should make the authoritative state transition explicit:

`RECEIVED → VALIDATED → AUTHORISED → POSTED/COMMITTED → DISPATCHED → SETTLED`

Additional terminal or compensating states include `REJECTED`, `EXPIRED`, `REVERSED`, `RETURNED` and `MANUAL_REVIEW`. State transitions are validated by domain rules; a reversal is a new compensating journal entry, not deletion or mutation of a posted transaction.

### Processing controls

1. Authenticate caller and derive legal entity from trusted identity/entitlement.
2. Validate request schema, currency, amount, beneficiary, limits, sanctions/risk result and business date.
3. Require an idempotency key. Store key, request hash, entity, status and result under a uniqueness constraint. Same key + same payload returns original outcome; same key + different payload is rejected and alerted.
4. Reserve funds / authorise using the authoritative ledger or core banking API. Define timeout semantics: timeout is **unknown outcome**, not proof of failure.
5. Commit balanced double-entry journal in a single ACID boundary. Enforce unique external reference and ledger sequence. Use serialisation/locking or optimistic concurrency for balance invariants.
6. Persist outbox event in the same database transaction as the ledger state change.
7. Publish asynchronously; consumers use inbox/deduplication and idempotent effects.
8. Respond success only after the authoritative commit point is durable.
9. Reconcile internal journal, external scheme acknowledgements, settlement files and core banking positions using independent controls.

Do not promise exactly-once message delivery. Design for at-least-once delivery plus idempotent processing and reconciliation.

## 3. Near-zero RPO: what is credible

### Credible within one Region

Aurora's regional storage is designed across multiple AZs. Multi-AZ deployment, synchronous storage durability, failover and tested connection recovery provide a strong regional HA pattern. This does not remove application retry, failover and commit-acknowledgement considerations.

### Cross-Region relational ledger

Aurora Global Database uses asynchronous cross-Region replication. AWS documentation describes typical RPO in seconds for an unplanned regional failover; lag varies. Planned switchover can synchronise before transition, but that is not the same as an unplanned disaster. Therefore **zero RPO is not guaranteed for an Aurora relational ledger after sudden Region loss**.

### DynamoDB MRSC

DynamoDB Global Tables multi-Region strong consistency synchronously replicates an item to at least one other Region before acknowledging success and supports RPO zero for the table's item changes. It requires exactly three Regions (three replicas or two replicas plus a witness), adds cross-Region latency, and **does not support TransactWriteItems/TransactGetItems**. It is suitable only where the business invariant can be represented and enforced at the item level, or where non-transactional item semantics are acceptable. It is not a drop-in replacement for a multi-row ACID ledger.

### Recommended decision

- Keep the relational ledger in a single-writer, multi-AZ regional ACID boundary unless the bank proves a distributed ledger model is required.
- Use MRSC for suitable per-payment canonical workflow/commit records where one item contains the complete immutable payment decision and no multi-item transaction is required. The ledger must consume this event idempotently and reconciliation must detect any downstream posting gap.
- If the bank's interpretation requires **zero loss of every committed multi-row ledger posting after complete Region loss**, treat it as a hard architecture gate. Select and prove a synchronous cross-Region consensus/ledger design (including quorum, latency, partition behaviour, write availability and country residency) before claiming compliance. Do not silently substitute Aurora Global Database.
- If the business accepts a bounded seconds-level regional RPO for relational ledger, formally record that as a deviation from “near zero” and obtain Risk/Business/Regulatory approval.

## 4. Active-active versus active-passive

Use active-active stateless application capacity in multiple approved Regions only where the data ownership and request routing are explicit. For a given legal entity and ledger partition, use a **single-writer Region/leader** for relational postings to avoid split-brain and conflicting balance updates. A second Region is warm standby for the ledger and can serve safe reads or non-authoritative functions.

If active-active payment intake is required, route each payment to a deterministic home Region/partition and reject or queue writes when ownership/quorum is uncertain. Use fencing tokens/epochs so a stale former primary cannot resume writes after failover. Do not use DNS health checks as a substitute for writer fencing.

## 5. Failover and reconciliation

### Failure declaration
- SRE detects Region impairment from independent health signals.
- Incident Commander and business service owner declare disaster under pre-agreed criteria.
- Validate that the primary is fenced (prevent old writer from accepting writes).
- Select secondary only if replication state and legal residency are approved.
- Promote database/ledger authority using documented runbook.
- Enable application capacity, secrets, KMS permissions, endpoints, queues, DNS/global routing and on-premises routes.
- Run synthetic payments and balance/journal checks before reopening full traffic.
- Reconcile in-flight, unknown-outcome and recently accepted payments using idempotency keys, journal sequence and external scheme status.
- Maintain a controlled queue for uncertain transactions; do not replay blindly.

### RTO budget (target allocation)
| Activity | Target budget |
|---|---:|
| Detection, triage and declaration | 5 min |
| Fence old writer / choose recovery point | 3 min |
| Promote/enable data services | 7 min |
| Application, network, secrets and routing activation | 7 min |
| Synthetic payment, ledger validation and controlled reopen | 8 min |
| **Total** | **30 min** |

This is a planning budget. Actual RTO must be proven with repeated full-stack exercises and service-specific timings.

## 6. Backup and cyber recovery

- AWS Backup policies centrally define frequency, retention, copy and restore controls for supported services.
- Enable point-in-time recovery for DynamoDB and automated backups/PITR for Aurora as appropriate.
- Copy backups to a separate backup account and approved Region only where residency allows.
- Use AWS Backup Vault Lock in compliance mode where policy requires immutable retention; evaluate logically air-gapped vaults for supported resource types and Regions.
- S3 versioning, Object Lock, MFA delete where appropriate, separate administration and restrictive bucket policies protect evidence/archive.
- Backup account has separate credentials, SCPs, KMS key policy and no routine workload-admin access.
- Protect Terraform state, artefact repositories, secrets and identity recovery procedures; a database backup alone is not cyber recovery.
- Regularly restore into an isolated recovery account and scan restored systems before reconnection.
- Keep offline or logically isolated copies of critical runbooks, configuration, signing materials and recovery contacts.

## 7. DR testing

| Test | Frequency | Evidence / pass condition |
|---|---|---|
| Database point-in-time restore | Quarterly | Restore to isolated account; validate journal totals and integrity |
| AZ impairment game day | Quarterly | Critical journey remains within SLO/error budget |
| Region failover simulation | At least annually; preferably semi-annually | Measured RTO ≤30 min; measured RPO meets approved class target |
| Network/Direct Connect loss | Semi-annually | VPN/alternate DX path works; no route leak or asymmetric firewall bypass |
| Queue replay / duplicate event | Quarterly | No duplicate financial posting; poison messages isolated |
| KMS/key compromise simulation | Annually | Key disable/recovery and break-glass process works |
| Backup vault compromise exercise | Annually | Independent restore path available |
| Full business continuity exercise | Annually | Operations, Finance, Risk and external dependencies participate |

## 8. Recovery limitations

- AWS service availability differs by Region; validate exact service/feature availability.
- MRSC Region sets are constrained and may not align with a country's approved data residency.
- A witness Region is a quorum participant, not a serving application Region.
- Asynchronous replication lag is workload- and failure-dependent.
- External clearing, correspondent banks, on-premises core systems and telecom carriers may dominate end-to-end recovery.
- Recovery automation can create split-brain if fencing and authority transfer are not correct.
