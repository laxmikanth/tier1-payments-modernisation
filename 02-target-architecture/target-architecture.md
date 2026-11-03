# Target Architecture — AWS

## 1. Account and entity topology

Use AWS Organizations with Control Tower as the landing-zone orchestration layer. Separate accounts are the primary blast-radius, billing and policy boundary. An account is not, by itself, a complete legal or data-residency control; combine it with Region restriction, IAM, KMS, resource policies, network boundaries and application-level authorisation.

Recommended OU/account pattern:

- **Management account:** Organizations and billing administration only; no workload deployment.
- **Security OU:** Log Archive account (central immutable audit storage) and Security Tooling/Audit account (delegated administration for Security Hub, GuardDuty, Config aggregators and incident tooling).
- **Infrastructure OU:** Network account (Transit Gateway, Network Firewall, DNS, egress), Shared Services account (directory integration, artefact mirrors, CI/CD runners where appropriate), Observability account (central telemetry).
- **Workloads OU:** one production account per legal entity and major regulated workload boundary, per approved Region; separate non-production account(s). If a country regulator requires a separate AWS account or independent operational boundary, create it.
- **Sandbox OU:** isolated experimentation with low quotas, no production data and no default production connectivity.

Do not create an account for every microservice by default. Account boundaries should follow legal entity, data classification, operational ownership and blast-radius requirements. Use VPC/subnet/security-group boundaries inside accounts for service segmentation.

## 2. Network topology and hybrid connectivity

- Each workload VPC spans three AZs, with separate public edge, private application, private data and isolated management subnets.
- Public exposure is limited to CloudFront/WAF/Shield and approved internet-facing ALB/API endpoints. Workload nodes and databases have no public IPs.
- Transit Gateway in the Network account connects workload VPCs and on-premises networks.
- AWS Direct Connect uses separate physical locations, devices and preferably carriers; use BGP dynamic routing and capacity that survives one circuit loss. Site-to-Site VPN over the internet is a tested backup path, not an untested diagram line.
- Use Direct Connect Gateway where appropriate for multi-Region access, with route segmentation and prefix filtering.
- Route inter-VPC, north-south and selected east-west traffic through AWS Network Firewall inspection. Ensure symmetric routing and avoid bypass routes. AWS Network Firewall's Transit Gateway native attachment can simplify central inspection where supported and appropriate; validate regional/service availability and routing model.
- Use Route 53 Resolver inbound/outbound endpoints for hybrid DNS. Define split-horizon zones and prevent resolver loops.
- Use VPC endpoints/PrivateLink for AWS service access where supported; S3 and DynamoDB gateway endpoints where appropriate.
- Central egress VPC with NAT Gateway and controlled proxy/DNS filtering; deny direct internet egress from data subnets.
- Use IPAM for address planning; detect overlapping on-premises ranges before connectivity build.

## 3. Edge and application traffic flow

1. Client/channel reaches Route 53 and CloudFront (where suitable).
2. AWS WAF applies managed and bank-specific rules; Shield Advanced is considered for critical internet-facing endpoints.
3. API Gateway or ALB terminates TLS and applies request size, rate and protocol controls.
4. Authentication uses bank identity provider federation/OIDC; customer authentication remains in the bank's customer IAM pattern. Service-to-service identity uses workload roles and short-lived credentials.
5. Payment API validates schema, signature, entitlement, entity, limits and idempotency key.
6. Payment orchestrator executes domain state transitions and calls the authoritative ledger boundary.
7. Durable commit is recorded before success response; outbox/event publication occurs without a dual-write gap.
8. Downstream work is decoupled through SQS/SNS/EventBridge or MSK based on delivery/order/stream semantics.
9. Adapters connect to on-premises core banking, sanctions, clearing, fraud and reporting over private hybrid connectivity.
10. Telemetry and audit events are emitted with correlation ID, payment ID, entity ID and trace context; sensitive payload fields are masked.

## 4. Entity isolation model

- **Account boundary:** production account per entity or regulator-required boundary.
- **Region boundary:** workloads and data deployed only in approved Regions; SCPs and deployment policy deny unapproved Regions.
- **Key boundary:** entity-specific KMS customer-managed keys; distinct key administrators and key policies. Use multi-Region keys only where legally approved and operationally justified; they do not remove residency obligations.
- **Data boundary:** entity-specific database clusters/tables/buckets, encryption context, resource policies, backup vaults and retention configuration.
- **Network boundary:** separate VPCs and route tables; explicit Transit Gateway attachments and firewall rules; no implicit entity-to-entity routes.
- **Identity boundary:** role trust conditions include account, entity, environment and workload identity; no shared human admin role across all entities.
- **Application boundary:** every request carries authenticated entity context; server derives entity from trusted identity/entitlement, never trusts a client-supplied tenant header alone. Enforce row/object-level authorization and test cross-tenant access.
- **Operations boundary:** separate break-glass approval, audit and data access. Central platform operators do not receive standing access to regulated payloads.

## 5. Identity and privileged access

- Federate workforce identity through IAM Identity Center and the bank IdP; enforce phishing-resistant MFA for privileged roles where possible.
- Permission sets map to job functions; use short-lived sessions and approval/JIT workflow for production privilege.
- Use IAM roles for workloads; EKS Pod Identity or IRSA for pod-level access. Never distribute static AWS access keys to pods or developers.
- SCPs prevent disabling CloudTrail/Config/GuardDuty, public S3, unapproved Regions, weak encryption and uncontrolled root use. SCPs are guardrails, not permission grants.
- Root credentials are vaulted, MFA-protected and used only for exceptional account recovery; alert on every root activity.
- Break-glass is time-bound, ticketed, dual-approved and recorded; credentials are rotated after use.
- Use IAM Access Analyzer, credential reports and periodic entitlement recertification.

## 6. Data services and integration

- Aurora PostgreSQL for relational transactional workloads where ACID, joins and relational constraints are needed. Multi-AZ within Region for HA; Aurora Global Database for cross-Region DR only with its asynchronous RPO understood.
- DynamoDB for key-value access patterns such as idempotency state, payment workflow state or immutable per-payment aggregate when its single-item model fits. MRSC may provide zero-RPO item replication, but it has strict three-Region topology, cross-Region latency and no transaction APIs. Do not use it as a drop-in multi-row relational ledger.
- S3 for object storage, statements, exports and immutable archives; Block Public Access, bucket policies, KMS, versioning, Object Lock where applicable, lifecycle and access logging.
- SQS for durable work queues and buffering; FIFO only where ordering/deduplication requirements justify throughput trade-offs. SNS for fan-out; EventBridge for event routing; MSK for Kafka-compatible streaming and replay needs.
- Transactional outbox or CDC is required when a database commit must trigger an event. Consumers are idempotent and maintain inbox/deduplication state.
- Schema Registry (for MSK) or a governed schema repository; contract tests and compatibility rules.
- Use Secrets Manager for credentials and rotation; Parameter Store for non-secret configuration.

## 7. Observability and operations

- CloudWatch metrics, logs, alarms and dashboards; X-Ray or OpenTelemetry tracing with correlation IDs.
- CloudTrail organization trail to central log archive; AWS Config organization aggregator; Security Hub and GuardDuty delegated administration.
- Central SIEM ingestion with retention, access controls, immutable archive and evidence retrieval.
- Business telemetry: payment accepted, authorised, posted, settled, rejected, reversed, timed out and reconciled. Avoid logging PAN, credentials, full account numbers or unnecessary PII.
- SLOs: availability, successful payment completion, p95/p99 latency, queue age, duplicate rate, ledger mismatch, replication lag, failover time and dependency health.
- Runbooks cover degraded modes, queue replay, duplicate suppression, reconciliation, key compromise, region isolation and manual settlement controls.

## 8. Deployment architecture

- IaC modules for accounts, VPC, endpoints, IAM, KMS, EKS/ECS, databases, queues and alarms; pinned provider/module versions and policy tests.
- CI: SAST, SCA, secret scan, IaC scan, container scan, licence checks, SBOM, image signing and provenance attestation.
- CD: signed immutable artefacts; deploy to non-production, integration, performance, resilience and pre-production; progressive canary/blue-green; automated rollback on SLO breach.
- Separate pipeline identities from runtime identities; no long-lived cloud credentials in GitHub. Prefer GitHub OIDC federation with tightly scoped IAM role trust, or bank-managed CI runners.
- Admission policy verifies signed images, approved registries, non-root execution, resource requests/limits, seccomp and no privileged containers.
- Use GitOps for Kubernetes desired state if operationally mature; retain an auditable deployment trail.

## 9. Availability engineering

For 99.99% critical-journey availability, design the complete dependency chain, not only compute. Deploy stateless services across three AZs, use health-based load balancing, multi-AZ data, queue buffering, circuit breakers, bulkheads, bounded retries with jitter, timeouts and load shedding. Define dependency budgets for on-premises systems and payment schemes. An unavailable external scheme cannot be hidden by an AWS SLA; the journey may need an accepted pending state and later reconciliation.
