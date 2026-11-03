# Executive Architecture Paper

## 1. Executive summary

**Selected cloud: Amazon Web Services (AWS).**

The bank should adopt a multi-account AWS landing zone and a country/entity-aware payments platform deployed across at least three Availability Zones in each approved operating Region. Product teams own domain services, APIs, data contracts, application SLOs and on-call response. Central platform teams own the landing zone, identity federation, network foundations, security guardrails, approved service catalogue, shared observability patterns and paved-road deployment capabilities.

The architecture separates four concerns:

1. **Control plane:** AWS Organizations, AWS Control Tower, IAM Identity Center, Organizations SCPs, AWS Config, CloudTrail, Security Hub and central audit accounts.
2. **Connectivity and trust:** redundant Direct Connect plus Site-to-Site VPN backup, Transit Gateway, centralized AWS Network Firewall inspection, private endpoints and explicit egress controls.
3. **Payment execution:** API ingress, authentication/authorization, payment orchestration, risk/fraud integration, ledger boundary, durable event publication and downstream adapters.
4. **Data and resilience:** entity-scoped transactional stores, immutable audit events, regional backups and cross-Region recovery. Consistency is selected by data class and business invariant.

The design uses EKS for complex multi-service workloads where Kubernetes capabilities are justified; ECS/Fargate for simpler container services; Lambda for event-driven, bounded tasks; and EC2 only for legacy or specialised workloads that cannot reasonably be containerised or managed otherwise. These are workload choices, not competing enterprise standards.

## 2. Business outcomes

- Maintain safe processing of domestic and cross-border payments across multiple legal entities.
- Preserve transaction integrity, idempotency, ordering where required, traceability and reconciliation.
- Isolate regulated data and operational access by legal entity and approved geography.
- Integrate with existing on-premises core banking, sanctions, clearing, fraud and reporting systems.
- Achieve 99.99% availability for defined critical journeys and a 30-minute recovery time objective.
- Make the RPO explicit per data class; do not misrepresent asynchronous replication as zero data loss.
- Migrate progressively with coexistence, controlled routing, reconciliation and tested rollback.
- Provide cost allocation and showback by entity, product, environment and shared platform.

## 3. Architecture at a glance

- AWS Organizations and Control Tower landing zone; separate security/log archive, network, shared services, and workload accounts.
- Production workload accounts segmented by legal entity and approved Region; separate non-production accounts and OU.
- VPCs spanning three AZs; private application/data subnets; public exposure only at controlled edge ingress.
- AWS WAF and Shield at edge, API Gateway or ALB ingress, AWS Network Firewall for routed inspection, GuardDuty, Security Hub, Config, CloudTrail and centralized logs.
- EKS/ECS workloads use private networking, workload identity via IAM Roles for Service Accounts (IRSA) or EKS Pod Identity, Secrets Manager and KMS.
- Aurora PostgreSQL-compatible clusters for relational ACID workloads within a Region; DynamoDB MRSC only for suitable item-level records whose access pattern fits its limitations; S3 for immutable/archive and data lake use cases.
- EventBridge, SQS, SNS and MSK selected by eventing semantics; transactional outbox or CDC prevents dual-write gaps.
- CloudWatch, X-Ray/OpenTelemetry, CloudTrail and centralized SIEM provide telemetry and evidence.
- IaC through Terraform or AWS CDK/CloudFormation; CI/CD with signed artefacts, policy checks and progressive delivery.

## 4. Key non-negotiable design positions

1. A payment is not acknowledged as committed until its authoritative durable commit point has completed.
2. Every externally retryable payment command carries a bank-issued or client-provided idempotency key with defined uniqueness scope and retention.
3. A legal entity's data plane, keys, logs, backups, access policies and recovery copies must remain within its approved residency boundary unless Legal/Compliance approves otherwise.
4. No public access to databases, cluster control planes or administrative endpoints.
5. Multi-AZ is the baseline for AZ failure; cross-Region DR is a separate, tested capability.
6. No active-active writes to the same relational ledger without a proven conflict-free ownership/consensus model.
7. Application portability is achieved through open interfaces and deployable artefacts; not by pretending all managed data services are interchangeable.
8. Platform guardrails are centrally enforced; application risk and SLO ownership remains with product teams.

## 5. Executive risks requiring bank decisions

- Whether “near zero RPO” means zero data loss for every committed payment after a complete Region loss, or a defined maximum such as 1–5 seconds.
- Whether payment ledger and core banking systems remain authoritative on-premises during transition.
- Country-specific rules for data residency, cross-border replication, operational access and backup location.
- Approved AWS Regions and whether a third Region is available for MRSC witness placement for the relevant data class.
- Required payment throughput, peak burst, end-to-end latency, batch windows and scheme deadlines.
- Whether the bank's 99.99% target is end-to-end (including external schemes/on-premises dependencies) or limited to bank-controlled platform components.

These are decision gates, not assumptions to hide in implementation.
