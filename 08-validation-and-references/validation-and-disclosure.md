# Validation Plan, Known Limitations and AI-Assistance Disclosure

## 1. Validation plan

| Area | Validation | Acceptance evidence |
|---|---|---|
| Landing zone | Test SCPs, account vending, Config, CloudTrail, delegated admin and Region deny | Automated control report; no unapproved drift |
| Entity isolation | Attempt cross-account, cross-VPC, cross-key, cross-bucket and cross-tenant access | All unauthorised attempts denied and logged |
| Hybrid network | Fail each Direct Connect circuit, device, location and VPN path | Route convergence within agreed window; no asymmetric bypass |
| Payment integrity | Duplicate request, same key/different payload, timeout after commit, out-of-order event, consumer retry | No duplicate posting; deterministic outcome and reconciliation |
| Availability | AZ impairment, node loss, task crash, queue backlog, dependency timeout | Critical journey meets SLO or controlled degraded state |
| DR | Full Region failover with database promotion, routing, secrets, KMS, queues and on-prem dependencies | Measured RTO ≤30 min; RPO within approved class |
| MRSC | Measure write/read latency, quorum loss and Region failure for chosen Region triplet | Zero-RPO item behaviour confirmed; latency and degraded-mode acceptable |
| Backup | Restore Aurora, DynamoDB, S3 and configuration into isolated recovery account | Data integrity and restore-time evidence |
| Security | Pen test, threat-model walkthrough, IAM review, image and IaC scanning | No unresolved critical/high findings without signed risk acceptance |
| Supply chain | Tampered image, unsigned image, vulnerable dependency and leaked secret test | Pipeline/admission blocks release and alerts |
| Performance | Peak, burst, soak, failover and back-pressure tests | p95/p99, TPS and saturation within approved budgets |
| Cost | Cost allocation, data transfer, idle DR, logs and request volume | Unit cost per payment and forecast within approved envelope |

## 2. Known limitations / open decisions

1. Region and country residency are not specified; no specific Region is selected in this paper.
2. AWS service availability and feature parity vary by Region and must be confirmed.
3. “Near zero” is ambiguous. Aurora Global Database is asynchronous; MRSC supports zero-RPO item replication but lacks DynamoDB transaction APIs.
4. RTO 30 minutes cannot be guaranteed until full-stack exercises include external/on-premises dependencies.
5. 99.99% end-to-end availability depends on payment schemes, telecom, core banking, identity and network dependencies outside the cloud platform.
6. Cost estimates cannot be produced without TPS, storage, retention, data egress, log volume, DR capacity and commercial terms.
7. This design does not claim PCI DSS, local regulatory or data protection compliance; controls must be mapped to the bank's control framework and jurisdiction.
8. The selected relational engine and schema require compatibility, performance and licensing validation.
9. Kubernetes/ECS/Lambda quotas, regional capacity, service limits and support arrangements must be checked before cutover.
10. The diagrams are logical reference diagrams, not detailed route tables, subnet CIDRs, firewall rule sets or production IaC.

## 3. Pre-production readiness checklist

- [ ] All critical journeys and dependencies have named owners and SLOs.
- [ ] Legal entity, data classification, residency and retention are approved.
- [ ] RPO is defined separately for ledger, payment state, events, documents and analytics.
- [ ] Database consistency and failover behaviour are proven under fault injection.
- [ ] Writer fencing and split-brain prevention are tested.
- [ ] Idempotency, outbox, inbox, replay and reconciliation are tested.
- [ ] Direct Connect and VPN failover are tested from each on-premises site.
- [ ] Backup vault immutability and isolated restore are proven.
- [ ] IAM, KMS, logging, SIEM, break-glass and supply-chain controls pass.
- [ ] Capacity, quotas, p95/p99 latency and cost forecast are approved.
- [ ] Runbooks, on-call, incident command and business continuity are rehearsed.
- [ ] Architecture exceptions and residual risks are signed by accountable owners.

## 4. AI assistance disclosure

This assessment was prepared with assistance from OpenAI ChatGPT (GPT-6) for architecture structuring, drafting, comparison of AWS design options, risk identification and diagram source generation. The candidate is responsible for reviewing, validating and owning all technical decisions. AWS documentation is cited in `references.md`; service capabilities, regional availability, pricing, security controls and regulatory applicability must be independently verified against the bank's current requirements before implementation.
