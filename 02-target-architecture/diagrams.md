# Architecture Diagrams

All diagrams use the same names as the written design. Render in GitHub using Mermaid support. For a formal submission, export SVG/PNG from Mermaid CLI and check readability at normal zoom.

## A. Context diagram

```mermaid
flowchart LR
  C[Customers / Channels] --> EDGE[Route 53 / CloudFront / WAF]
  EDGE --> API[Payments API]
  API --> PAY[Payment Domain Services]
  PAY --> LEDGER[Authoritative Ledger Boundary]
  PAY <--> INT[Integration Adapters]
  INT <--> DC[On-premises Core Banking]
  INT <--> SCHEME[Clearing / Payment Schemes]
  INT <--> RISK[Sanctions / Fraud / AML]
  PAY --> EVT[Durable Events]
  EVT --> DOWN[Notifications / Reporting / Reconciliation]
  PAY --> OBS[Observability & Audit]
  LEDGER --> OBS
  subgraph Governance
    ORG[AWS Organizations / Control Tower]
    SEC[Security, Audit & Guardrails]
    ORG --> SEC
  end
  SEC -.governs.-> EDGE
  SEC -.governs.-> PAY
  SEC -.governs.-> LEDGER
```

## B. Logical architecture

```mermaid
flowchart TB
  CH[Channels / Partner APIs] --> WAF[AWS WAF + Shield]
  WAF --> IN[API Gateway / ALB]
  IN --> AUTH[Authentication, Entitlement & Entity Context]
  AUTH --> PAPI[Payment API]
  PAPI --> ORCH[Payment Orchestrator]
  ORCH --> VALID[Validation / Limits / Risk]
  VALID --> LEDGER[Ledger Adapter]
  LEDGER --> DB[(Aurora PostgreSQL - Regional ACID)]
  ORCH --> IDEM[(Idempotency / Workflow State)]
  ORCH --> OUTBOX[(Transactional Outbox)]
  OUTBOX --> PUB[Outbox Publisher / CDC]
  PUB --> BUS[EventBridge / SNS / SQS / MSK]
  BUS --> ADAPT[Core / Scheme / Fraud Adapters]
  ADAPT --> HYB[Transit Gateway + Network Firewall]
  HYB --> ONPREM[On-premises systems]
  BUS --> RECON[Reconciliation / Reporting]
  PAPI --> TEL[Metrics / Logs / Traces]
  ORCH --> TEL
  DB --> TEL
```

## C. Deployment and trust-boundary diagram

```mermaid
flowchart TB
  subgraph ORG["AWS Organization — central governance boundary"]
    CT[Control Tower / Organizations / SCPs]
    subgraph SEC["Security OU"]
      LOG[Log Archive Account: CloudTrail, Config, immutable S3]
      TOOLS[Security Tooling: GuardDuty, Security Hub, SIEM]
    end
    subgraph INFRA["Infrastructure OU"]
      NET[Network Account: Transit Gateway, Network Firewall, DNS, egress]
      SHARED[Shared Services: CI/CD, artefact mirror, directory integration]
    end
    subgraph PROD["Workloads OU — Entity A / approved Region"]
      subgraph VPC["Entity A Production VPC — three AZs"]
        subgraph AZ1["AZ-a"]
          EDGE1[Private ingress / ALB]
          APP1[EKS or ECS services]
          DATA1[(Aurora writer / replica)]
        end
        subgraph AZ2["AZ-b"]
          APP2[EKS or ECS services]
          DATA2[(Aurora storage / replica)]
        end
        subgraph AZ3["AZ-c"]
          APP3[EKS or ECS services]
          DATA3[(Aurora storage / replica)]
        end
        SQS[SQS / EventBridge]
        KMS[KMS Entity A key]
      end
    end
    CT -.guardrails.-> SEC
    CT -.guardrails.-> INFRA
    CT -.guardrails.-> PROD
    VPC <--> NET
    APP1 --> KMS
    DATA1 --> KMS
    SEC -.audit / detect.-> PROD
  end
  ONPREM[On-premises data centres] <-->|Dual Direct Connect + VPN backup| NET
  REGION[Approved secondary Region / separate entity account] <-->|Approved DR replication only| PROD
```

## D. Regional DR diagram

```mermaid
flowchart LR
  subgraph R1["Primary approved Region"]
    G1[Route 53 / Global Accelerator policy]
    A1[Ingress + WAF]
    S1[Multi-AZ application services]
    DB1[(Aurora primary cluster)]
    DDB1[(DynamoDB MRSC replica - suitable item records)]
    Q1[SQS / EventBridge]
    G1 --> A1 --> S1
    S1 --> DB1
    S1 --> DDB1
    S1 --> Q1
  end
  subgraph R2["Secondary approved Region"]
    A2[Standby ingress]
    S2[Warm application capacity]
    DB2[(Aurora Global DB secondary - async)]
    DDB2[(DynamoDB MRSC replica)]
    Q2[Recovery queues / replay]
    A2 --> S2
    S2 --> DB2
    S2 --> DDB2
    S2 --> Q2
  end
  subgraph R3["Approved witness Region"]
    W[MRSC witness or third replica]
  end
  DB1 -.asynchronous physical replication.-> DB2
  DDB1 <-->|synchronous item-level quorum| DDB2
  DDB1 <-->|quorum participation| W
  USERS[Clients / partners] --> G1
  G1 -.health-based traffic shift after approval.-> A2
  ONP[On-premises core / clearing] <-->|Redundant hybrid routes| R1
  ONP <-->|Pre-approved recovery routes| R2
```

## E. Migration roadmap

```mermaid
flowchart LR
  D[0 Discover & classify] --> F[1 Landing zone & connectivity]
  F --> P[2 Platform and observability]
  P --> N[3 Non-critical pilot]
  N --> C[4 Coexistence and domain migration]
  C --> H[5 Critical payment cutover]
  H --> O[6 Optimise and decommission]
  C -.rollback to legacy route.-> C
  H -.rollback only before irreversible ledger authority switch.-> C
```

## Diagram notes

- Entity A is illustrative. Replicate the account/VPC/key/data pattern for each legal entity and approved Region; do not imply all entities share the same data plane.
- Region R3 is only used for DynamoDB MRSC quorum if it is in the permitted MRSC Region set and approved for the data class. A witness is not an application serving Region.
- Aurora Global Database is asynchronous cross-Region replication. The diagram intentionally does not imply zero RPO for Aurora.
- Exact service placement, ingress pattern, Region, number of AZs and route tables require implementation-level validation.
