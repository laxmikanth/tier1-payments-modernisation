# Architecture Principles, Quality Attributes and Governance

## 1. Principles

| Principle | Rule | Practical consequence |
|---|---|---|
| Business/entity isolation | Separate legal entities by account, Region, keys, data stores and access policy where required | Avoid shared databases across entities unless approved and demonstrably isolated |
| Secure by default | Deny by default, least privilege, private networking, encryption and auditable access | Central guardrails; no direct public data-plane exposure |
| Multi-AZ first | Critical synchronous services span at least three AZs where supported | AZ loss must not require a Region failover |
| Explicit consistency | Choose consistency per invariant and data class | No “eventual consistency everywhere” for money movement |
| Idempotent processing | Commands can be safely retried without duplicate financial effects | Durable idempotency state and duplicate detection |
| API/event contracts | Versioned, documented and backward-compatible interfaces | OpenAPI, AsyncAPI/CloudEvents-compatible envelopes, schema registry |
| Portability by design | Keep business logic and deployment artefacts independent of one orchestrator | OCI images, standard runtimes, Terraform, SQL where feasible, portable tests |
| Managed services selectively | Prefer managed services when their operational benefit outweighs coupling | Document exit cost and data export for each strategic managed service |
| Progressive migration | Migrate bounded domains with coexistence and reconciliation | Strangler routing; avoid big-bang cutover |
| Observable by default | Metrics, logs, traces, business events and audit evidence are built in | SLO dashboards and runbooks are release criteria |
| FinOps as architecture | Attribute cost to entity/product/environment | Mandatory tags, budgets, anomaly alerts and unit economics |
| Recoverability is tested | Backups and DR are proven through restore/failover exercises | Evidence-based recovery objectives, not paper RTOs |

## 2. Quality attributes and measurable acceptance

| Attribute | Target / acceptance evidence |
|---|---|
| Availability | 99.99% monthly for agreed critical journeys; error budget and exclusions formally defined |
| Recovery time | RTO ≤30 minutes for a declared Region disaster, including traffic, dependencies, data promotion and business validation |
| Recovery point | Zero loss for data explicitly engineered with a synchronous/quorum commit; seconds-level RPO for asynchronous replicas; never conflate these |
| Integrity | No duplicate debit/credit; balanced journal; idempotent retry; deterministic reconciliation |
| Latency | Business owner to define p95/p99 by journey and country; test at peak and degraded mode |
| Security | MFA, federation, least privilege, encryption, private endpoints, immutable audit, vulnerability SLAs |
| Residency | Data inventory mapped to approved Region and transfer rules; automated policy evidence |
| Scalability | Peak + agreed headroom test; queue depth and back-pressure thresholds |
| Operability | Named service owner, runbook, on-call, dashboards, SLO, capacity and DR test |
| Portability | Build once as OCI image; deploy to at least one alternative conformant runtime in a portability test where required |
| Cost | Tagged spend, per-payment unit cost, budget variance alert and monthly review |

Availability calculation: 99.99% corresponds to approximately 4.38 minutes of downtime in a 30.4-day month. This is a service objective, not an AWS service guarantee. The SLO must define measurement window, planned maintenance treatment, dependency attribution and customer-visible success criteria.

## 3. Hard constraints

- AWS only for the cloud target; on-premises systems are explicit hybrid dependencies.
- Regulated data stays in approved locations. Do not replicate across Regions until the data owner and Compliance approve the destination.
- RTO 30 minutes; near-zero RPO for committed payment transactions is a design requirement to validate, not a blanket property of every AWS database.
- Central platform team controls landing zones and preventive guardrails.
- Product teams own applications, business logic, SLOs, runbooks and remediation.
- No direct production access by developers as a routine operating model.
- No secrets in source control, images, Terraform state or CI logs.
- Migration must preserve reconciliation and a controlled rollback path.

## 4. Assumptions and exclusions

### Assumptions
- The bank has multiple legal entities and each entity has a documented approved Region set.
- Existing core banking, clearing, sanctions, fraud and selected customer systems remain on-premises during early migration waves.
- Payment processing has a durable authoritative ledger boundary and a unique transaction/reference model.
- At least three AZs are available in the selected primary Region; secondary Region and service availability are validated before design freeze.
- AWS Organizations and an enterprise identity provider are available.
- The bank has a central SIEM and a security incident response function.

### Exclusions pending discovery
- Specific countries, regulators, AWS Regions, payment schemes and data-retention periods.
- Exact TPS, payload sizes, latency budgets, concurrency, batch and settlement volumes.
- Detailed PCI DSS scope, cardholder data flow and tokenisation requirements.
- Exact RDS/Aurora engine version, schema, extensions and licensing.
- Commercial pricing, reserved capacity and Savings Plans.
- Existing network topology, IP overlap, Direct Connect locations and carrier contracts.
- Formal certification of any proposed service for the bank's jurisdiction.

## 5. Decision ownership / RACI

| Decision / activity | Accountable | Responsible | Consulted |
|---|---|---|---|
| Landing zone, OUs, SCPs, shared networking | Head of Cloud Platform | Cloud Platform | Security, Network, FinOps |
| Entity residency and cross-border transfer | Country Data Owner / Compliance | Data Governance | Legal, Security, Architecture |
| Application design and service SLO | Product Owner | Product Engineering | Architecture, SRE |
| Payment ledger semantics and reconciliation | Payments Domain Owner | Ledger/Data Engineering | Finance, Risk, Operations |
| Threat model and control acceptance | CISO delegate | Security Architecture | Product, Platform, Audit |
| DR strategy and recovery declaration | Business Service Owner | SRE / Operations | Data, Platform, Risk |
| Architecture exception | Architecture Review Board (ARB) | Design Authority | Security, Risk, Operations |
| Migration go/no-go | Programme Executive | Migration Lead | Product, Operations, Compliance |

## 6. Governance cadence

- **Architecture Review Board:** fortnightly; approves target patterns, exceptions, cross-entity design and material risk acceptance.
- **Design Authority:** weekly; reviews solution designs, ADRs, NFR traceability and technical debt.
- **Security design review:** threat model before build, pre-production penetration test and annual/major-change reassessment.
- **Operational readiness review:** before production; SLO, on-call, runbook, dashboards, backup restore, DR test, capacity and cost.
- **FinOps review:** monthly; cost allocation, anomaly, idle resources, data transfer and per-payment unit cost.
- **Exception expiry:** every exception has an owner, compensating control, expiry date and remediation plan.
