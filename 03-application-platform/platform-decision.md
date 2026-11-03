# Application Platform Decision — EKS vs ECS vs Lambda vs EC2

## 1. Decision summary

Adopt a **portfolio platform** rather than mandate one runtime:

- **EKS:** complex domain services requiring Kubernetes ecosystem, advanced scheduling, service mesh or a consistent platform across multiple teams. Use managed control plane, private endpoint, managed node groups or Fargate selectively, multi-AZ nodes, Pod Identity/IRSA and policy enforcement.
- **ECS with Fargate:** default for straightforward containerised APIs, adapters and background workers where Kubernetes features are not needed. Lower cluster-management burden and AWS-integrated deployment.
- **Lambda:** event-driven, short-lived, bursty and independently scalable functions such as notification, lightweight validation, file/event processing and scheduled reconciliation triggers. Not the default for latency-sensitive long-running orchestration or workloads requiring specialised runtime control.
- **EC2:** constrained legacy, vendor appliance, specialised kernel/driver, licensed software or stateful workload that cannot be reasonably moved to managed containers. Treat as an exception with patching, AMI, backup and lifecycle ownership.

## 2. Comparison

| Dimension | EKS | ECS/Fargate | Lambda | EC2 |
|---|---|---|---|---|
| Operating model | Managed Kubernetes control plane; customer manages workload and node/platform configuration | AWS-native orchestrator; Fargate removes host management | Fully managed function runtime | Customer manages OS, agents, patching and capacity |
| Best fit | Complex microservices platform, ecosystem/tooling needs | Standard container services and workers | Event-driven bounded functions | Legacy/vendor/specialised workloads |
| Portability | High at Kubernetes API level, but add-ons, IAM, ingress, storage and networking remain provider-specific | OCI image portable, ECS task/service definitions AWS-specific | Low runtime portability; function code can be portable | OS/image portable with substantial infrastructure coupling |
| Scaling | HPA/Karpenter/Cluster Autoscaler patterns; tune carefully | Service autoscaling; Fargate capacity managed by AWS | Event/request-driven scaling; concurrency quotas and downstream protection required | ASG/manual scaling; capacity planning burden |
| Failure modes | Cluster/add-on/node/pod/AZ dependencies; control-plane managed by AWS | Service scheduler/Fargate/AZ capacity and task health | Invocation errors, throttling, concurrency exhaustion, timeout | Host, AZ, OS, agent and patch failures |
| Operations | Highest platform skill and governance demand | Moderate; fewer infrastructure concerns | Low infrastructure operations, more event/runtime constraints | Highest OS operations |
| State | Keep state external; persistent volumes only for justified cases | Keep state external | Stateless execution; external state | Can host state but increases recovery burden |
| Cost profile | Efficient at sustained utilisation; cluster/platform overhead | Pay for task resources/time; convenient, may cost more at sustained load | Pay per request/duration; cost and concurrency need modelling | Potentially efficient with commitments but idle capacity and ops cost |
| Banking controls | Strong policy possible; admission, network, secrets and workload identity must be designed | IAM task roles, security groups, image controls and deployment policy | Execution role, VPC access, concurrency and event-source controls | Full OS hardening and patch compliance burden |

## 3. Representative component mapping

| Component | Preferred runtime | Rationale / guardrail |
|---|---|---|
| Payment API / orchestration | ECS/Fargate or EKS | Choose EKS only if platform standardisation, advanced traffic policy or ecosystem justifies it; keep API stateless |
| High-throughput payment workers | ECS/Fargate or EKS | Long-running consumers, controlled concurrency, back-pressure and predictable scaling |
| On-premises integration adapter | ECS/Fargate | Containerised adapter with private network, mTLS and circuit breakers |
| Risk model inference service | EKS/EC2 depending accelerator/runtime | Dedicated capacity, latency and model-serving requirements may justify EKS/EC2 |
| Notifications | Lambda | Event-driven, bounded and retry-safe |
| File ingestion / statement transform | Lambda for small files; ECS for large/long-running | Avoid Lambda timeout/memory limits; use S3 event + SQS buffering |
| Reconciliation batch | ECS task / AWS Batch | Long-running controlled batch with checkpointing; Lambda only for small partitioned jobs |
| Legacy vendor package | EC2 | Time-box exception and define exit plan |

## 4. Application and API portability

Portability is a set of measurable properties:

1. **Domain independence:** business rules live in application code, not in a proprietary workflow engine or cloud-specific orchestration construct.
2. **Container portability:** OCI images, standard Linux base images, non-root process, externalised configuration and health endpoints.
3. **API contracts:** OpenAPI for synchronous APIs; AsyncAPI/CloudEvents-style envelope for events; semantic versioning and consumer-driven contract tests.
4. **Data portability:** documented logical schema, export/import process, standard SQL where suitable, explicit data ownership and tested restore/export. Do not pretend DynamoDB data models are interchangeable with relational SQL.
5. **Infrastructure portability:** Terraform modules or well-isolated IaC modules; separate cloud-specific modules from reusable workload modules.
6. **Observability portability:** OpenTelemetry instrumentation and structured logs; backend exporters can vary.
7. **Identity abstraction:** OIDC/OAuth2 and workload identity; avoid proprietary application assumptions about long-lived cloud credentials.
8. **Delivery portability:** signed artefacts, SBOM, provenance, immutable versioning and environment promotion independent of runtime.
9. **Resilience patterns:** idempotency, retries, circuit breakers, bulkheads and outbox are implemented in domain/application patterns rather than relying exclusively on one vendor service.
10. **Exit evidence:** a periodic portability exercise deploys a representative service to an alternate conformant Kubernetes/container runtime and exports/restores a representative dataset.

## 5. What not to abstract

Do not create a lowest-common-denominator abstraction over every AWS service. Use AWS-native managed services where justified, but isolate them behind domain-owned adapters and contracts. The abstraction should protect business logic and provide a replacement seam, not reimplement cloud services poorly.

## 6. Operational guardrails

- EKS: private API endpoint; supported version policy; managed node groups; Pod Identity/IRSA; network policies; admission policy; image signing; resource quotas; PDBs; multi-AZ node placement; upgrade rehearsal.
- ECS: task roles, private subnets, deployment circuit breaker, health checks, autoscaling, capacity provider strategy and immutable images.
- Lambda: reserved concurrency for critical functions, DLQ/destinations, idempotent handlers, bounded retries, alarms for throttles/errors and VPC attachment only when needed.
- EC2: golden AMI, Systems Manager, patch baselines, EDR, IMDSv2, encrypted EBS, ASG health checks and documented decommission date.

## 7. Review triggers

Reassess runtime when a service's p99 latency, deployment frequency, operational toil, cost per transaction, scaling behaviour, security posture or portability test fails its target. Runtime choice is not permanent; migration requires compatibility tests and an approved change plan.
