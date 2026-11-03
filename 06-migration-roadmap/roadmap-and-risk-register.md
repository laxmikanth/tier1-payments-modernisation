# Migration Roadmap, Coexistence and Risk Register

## 1. Migration strategy

Use a strangler migration by bounded business capability and legal entity. Avoid a single big-bang cutover. Keep the legacy platform authoritative for a payment population until the new path has passed reconciliation, operational readiness and business sign-off. A payment must have one authoritative writer at any point in time.

## 2. Waves

| Wave | Scope | Key deliverables | Exit criteria |
|---|---|---|---|
| 0 — Discover and classify | Inventory apps, interfaces, jobs, data, certificates, batch, dependencies and owners | Dependency map, data classification, performance baseline, risk and residency matrix | 100% critical workloads and interfaces have owner, data class, dependency and recovery tier |
| 1 — Foundation | Control Tower, accounts, OUs, SCPs, identity, IPAM, DX/VPN, TGW, inspection, logging, KMS | Landing zone and network patterns as code | Security controls pass; connectivity failover tested; accounts/Region guardrails proven |
| 2 — Platform enablement | EKS/ECS patterns, ECR, CI/CD, secrets, observability, policy, backup | Golden paths, service templates, SLO dashboard, runbooks | Product team deploys sample service securely and recovers it |
| 3 — Non-critical pilot | Reporting, notifications, read-only APIs or low-risk adapters | First workload migration, cost and operational baseline | No data leakage; SLO achieved; restore and rollback tested |
| 4 — Domain coexistence | Selected payment product/entity; API facade and routing; outbox/event integration | Dual-run controls, shadow traffic, reconciliation | Reconciliation within tolerance for agreed observation period; no duplicate posting |
| 5 — Critical cutover | Payment intake and ledger integration for a bounded entity/flow | Controlled routing switch, freeze window, business sign-off | Synthetic payments, settlement and reconciliation pass; rollback authority remains available |
| 6 — Optimise/decommission | Retire old components and duplicated infrastructure | Licence/contract closure, data archive, CMDB updates | Retention obligations met; no residual traffic/dependency; approved decommission certificate |

## 3. Coexistence pattern

- API facade routes each payment using a deterministic key (entity, product, account or transaction population) to exactly one authoritative platform.
- Maintain an explicit routing registry with version, effective time, owner and audit history.
- Shadow mode may compare decisions but must not post a second financial transaction.
- Use CDC/outbox for downstream replication where required; do not dual-write to two ledgers.
- Reconcile legacy and target outputs using business keys, amounts, currency, value date, status and journal totals.
- For unknown outcomes, query authoritative state by idempotency/reference before retrying.
- Freeze schema changes during cutover unless compatibility has been proven.
- Run parallel operations for reporting/monitoring, not parallel financial authority.

## 4. Rollback points

| Stage | Rollback approach | Stop condition |
|---|---|---|
| Foundation | Revert IaC change / detach test VPC; preserve audit | Guardrail, route or security regression |
| Platform | Roll back service image/config; retain previous signed digest | Error budget burn, latency or vulnerability |
| Shadow | Disable mirror traffic | Any unintended write or sensitive-data exposure |
| Canary | Shift routing to legacy before target becomes authoritative | Reconciliation mismatch, duplicate, unexplained pending |
| Cutover | Use pre-approved route rollback only while old ledger remains consistent and no irreversible posting divergence exists | Any uncertainty about which platform owns a transaction |
| Post-cutover | Prefer forward fix if rollback would split financial history; freeze new intake and reconcile | Ledger divergence or incomplete settlement |

A rollback is not automatically safe after a financial posting. Define a transaction boundary and authority transfer procedure before cutover.

## 5. Technical debt controls

- Every exception has owner, business reason, risk, compensating control, expiry and remediation milestone.
- Maintain an architecture decision log and dependency inventory.
- Prohibit unapproved database sharing, hard-coded Region names, static credentials and undocumented point-to-point interfaces.
- Track unsupported runtimes, container base images, EKS versions, end-of-life databases and obsolete cipher suites.
- Set a maximum age for migration exceptions and report them at ARB.
- Measure portability and recovery through periodic tests, not documentation alone.

## 6. Organisational dependencies

- Country Compliance/Legal: data residency, retention, cross-border and regulator notification requirements.
- Payments Operations/Finance: ledger semantics, settlement, reversals, reconciliation and business-day calendars.
- Network: Direct Connect, carriers, BGP, IP overlap, DNS and firewall rule ownership.
- Identity/Security: IdP integration, privileged access, key custody, SIEM and incident response.
- Data Engineering: schema ownership, CDC, data quality and archive.
- Product teams: service SLOs, domain ownership, tests, on-call and migration execution.
- Vendor management: support models, licence transfer, exit clauses and third-party recovery.
- Change management: market-by-market release windows, scheme certification and customer communications.

## 7. Risk register

| ID | Risk | Likelihood / impact | Mitigation | Owner | Trigger |
|---|---|---|---|---|---|
| R1 | Data residency prohibits planned DR Region | Medium / Critical | Country-by-country data map; approved Region matrix; alternate local recovery design | Compliance | Region approval rejected |
| R2 | Cross-Region relational replication loses recent commits | Medium / Critical | Explicit RPO acceptance or synchronous consensus ledger proof; reconcile | Data Architect | Aurora lag exceeds threshold / region outage |
| R3 | Split-brain after failover | Low / Critical | Fencing token, single-writer, authority lease, failover runbook | SRE/Data | Conflicting writer epoch |
| R4 | Hybrid link becomes single point of failure | Medium / High | Multi-site DX, diverse carriers/devices, VPN backup and test | Network | Circuit or provider outage |
| R5 | Legacy dependency discovered late | High / High | Discovery, traffic analysis, interface contracts and dependency owner sign-off | Migration Lead | Undocumented call path |
| R6 | Duplicate financial posting during retry | Medium / Critical | Idempotency, unique constraints, outbox/inbox and reconciliation | Payments Product | Duplicate rate >0 |
| R7 | Kubernetes operational complexity | Medium / Medium | Prefer ECS where K8s not needed; platform SRE; paved road | Platform | High toil / repeated incidents |
| R8 | Cost growth from cross-AZ/Region traffic and idle DR | Medium / High | FinOps tags, traffic design, right sizing, DR capacity tests | FinOps | Budget variance threshold |
| R9 | Key or backup compromise | Low / Critical | Separate backup account, Vault Lock, dual control, restore test | Security | Destructive action alert |
| R10 | Cutover cannot meet 30-min RTO | Medium / Critical | Rehearsal, automation, pre-provisioned capacity and timed runbooks | SRE | Test >30 minutes |
| R11 | Third-party scheme outage | Medium / High | Pending state, durable queue, timeout semantics and reconciliation | Payments Ops | Partner SLA breach |
| R12 | Service quota/capacity blocks failover | Medium / High | Quota review, pre-provisioned standby, load test, support plan | Platform | Capacity test failure |

## 8. Programme gates

- **Gate A — Discovery complete:** inventory, data lineage, residency and service ownership accepted.
- **Gate B — Foundation certified:** security, network, logging, IAM and recovery foundations tested.
- **Gate C — Pilot accepted:** business and technical SLOs proven.
- **Gate D — Migration readiness:** reconciliation, rollback, capacity, support and compliance approved.
- **Gate E — Cutover approval:** business owner, Operations, Security, Compliance and Architecture sign-off.
- **Gate F — Decommission:** data retention, legal hold, archive, access revocation and dependency removal verified.
