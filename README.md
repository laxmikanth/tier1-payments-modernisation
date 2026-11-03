# Tier 1 Payments Platform Modernisation — AWS Reference Architecture

**Selected cloud: Amazon Web Services (AWS)**  
**Assessment version:** 1.0  
**Prepared:** 3 November 2026  
**Status:** Architecture proposal for assessment; validate against bank-specific regulatory, workload and regional facts before production adoption.

## Executive position

Modernise the payments platform through a governed AWS multi-account landing zone, country/entity-aligned workload boundaries, multi-AZ regional execution, hybrid connectivity, and an explicitly tiered data strategy. Use managed services where they reduce undifferentiated operations, but keep domain logic, APIs, event schemas, deployment artefacts and infrastructure definitions portable.

The design targets **99.99% availability for critical journeys** and **30-minute RTO**. Near-zero RPO is treated as a data-class-specific requirement, not a blanket claim: DynamoDB Global Tables multi-Region strong consistency (MRSC) provides zero-RPO item replication within its supported three-Region topology, but MRSC does **not** support DynamoDB transaction APIs. Aurora Global Database replication is asynchronous; AWS documents typical RPO in seconds for unplanned regional failover. Therefore the relational ledger's cross-Region zero-RPO claim is not made without an additional synchronous commit protocol and proof. See `04-data-and-dr/`.

## Repository map

| Path | Contents |
|---|---|
| `01-executive-architecture/` | Context, principles, quality attributes, assumptions, governance |
| `02-target-architecture/` | Logical/deployment design, trust boundaries, networking, identity, entity isolation |
| `03-application-platform/` | EKS/ECS/Lambda/EC2 comparison and portability model |
| `04-data-and-dr/` | Transaction integrity, replication, failover, backup, cyber recovery and tests |
| `05-threat-and-control-model/` | Threat/control matrix and privileged-access controls |
| `06-migration-roadmap/` | Migration waves, coexistence, rollback, exit criteria and risk register |
| `07-architecture-decision-records/` | Decision records including the contentious database/DR choice |
| `08-validation-and-references/` | Validation plan, known limitations, references and AI disclosure |
| `diagrams/` | Mermaid source for context, logical, deployment, DR and migration diagrams |

## How to use

1. Review assumptions and replace placeholders with approved country, entity, data-classification, transaction-volume, latency and regulatory requirements.
2. Render Mermaid diagrams in GitHub (or Mermaid CLI) and verify labels remain readable.
3. Review each ADR with Architecture Review Board, Security, Data, Operations, Risk and country compliance owners.
4. Turn the validation plan into executable non-functional tests and evidence.
5. Commit the repository to a bank-approved GitHub organization. Do not place credentials, production data, secrets, internal IP ranges or sensitive architecture details in a public repository.

## Suggested GitHub publication

```bash
git init
git add .
git commit -m "Initial AWS Tier 1 payments architecture assessment"
git branch -M main
git remote add origin https://github.com/<organisation>/tier1-payments-modernisation.git
git push -u origin main
```

Use a **private repository** unless the bank explicitly authorises public disclosure. Enable branch protection, code scanning and secret scanning where available.

## Scope boundary

This is a target-state architecture and assessment response, not a production-ready low-level design. Region selection, service availability, quotas, pricing, data residency, payment scheme rules, PCI DSS scope, retention periods, key ownership, cross-border transfer rules and actual workload characteristics must be confirmed with the bank. No service-level objective is guaranteed merely by selecting a service.
