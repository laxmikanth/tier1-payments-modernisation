# ADR-001: Use a portfolio of AWS application runtimes

- **Status:** Proposed
- **Date:** 3 November 2026
- **Decision owners:** Enterprise Architecture, Cloud Platform, Payments Product Engineering
- **Context:** Teams disagree whether the bank should standardise on EKS, ECS, Lambda or EC2.
- **Decision:** Adopt EKS for workloads that genuinely need Kubernetes capabilities; ECS/Fargate as the default for standard container services; Lambda for bounded event-driven tasks; EC2 by exception for legacy/specialised workloads.

## Drivers

- 99.99% critical journey availability and controlled recovery.
- Minimise operational complexity and avoid unnecessary Kubernetes platform burden.
- Preserve application-level portability and repeatable delivery.
- Support diverse workload shapes, regulatory controls and team maturity.
- Keep product teams accountable for service SLOs while central platform provides paved roads.

## Options considered

### Option A — EKS for every service
**Advantages:** Kubernetes API and ecosystem, consistent scheduling model, rich policy/service-mesh options, portability across conformant Kubernetes environments.

**Disadvantages:** Cluster lifecycle, add-ons, node capacity, networking, admission, upgrades and skills create a material platform operating model. Kubernetes does not make databases, IAM, storage, ingress or event services portable. It can become a central bottleneck.

**Disposition:** Rejected as universal mandate. EKS is approved where complexity is justified.

### Option B — ECS/Fargate for every container service
**Advantages:** Simpler AWS-native orchestration, less control-plane/node management, good fit for conventional APIs and workers.

**Disadvantages:** ECS task/service definitions and ecosystem are AWS-specific; less compatible with Kubernetes tooling and policies; some advanced scheduling and platform extensions are less portable.

**Disposition:** Accepted as default for standard container workloads, not as universal mandate.

### Option C — Lambda for all compute
**Advantages:** Minimal server management, event-driven scale and pay-per-use economics.

**Disadvantages:** Runtime, execution duration, concurrency, cold-start, packaging and event semantics constrain suitability. Long-running orchestration, sustained consumers and specialised networking may be better in containers.

**Disposition:** Rejected as universal runtime; approved for bounded event-driven functions.

### Option D — EC2 for all workloads
**Advantages:** Maximum OS/runtime control and compatibility with legacy/vendor software.

**Disadvantages:** Patching, scaling, host hardening, capacity, image management and recovery increase toil and attack surface.

**Disposition:** Rejected as default; approved only for documented constraints.

## Consequences

- Platform team maintains two paved roads (ECS and EKS) plus Lambda patterns; EC2 exception catalogue.
- Product teams select runtime with an ADR-lite template and NFR evidence.
- All workloads externalise state and use standard API/event contracts.
- OCI images, SBOM, signed artefacts, OTel instrumentation and IaC are mandatory baseline.
- EKS cost and operational toil are measured; teams must demonstrate why Kubernetes is needed.
- Portability is tested at application and artefact boundaries, not claimed from runtime selection alone.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Too many runtimes fragment operations | Limit approved patterns; standardise pipelines, logging, identity and runbooks |
| ECS lock-in | Keep business logic in OCI container; isolate ECS service definitions in IaC module |
| EKS upgrade/security drift | Supported-version policy, automated conformance, upgrade rehearsals |
| Lambda retry duplicates | Idempotent handler, DLQ, reserved concurrency, event replay tests |
| EC2 exception becomes permanent | Owner, compensating controls and dated exit plan |

## Review triggers

Review if the platform team's support capacity is exceeded, runtime choice causes SLO failure, portability test fails, AWS pricing changes materially, or a new workload requires a capability not met by current patterns.
