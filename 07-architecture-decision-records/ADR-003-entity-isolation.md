# ADR-003: Legal-entity isolation through account, Region, key and data boundaries

- **Status:** Proposed
- **Decision:** Use separate AWS production accounts per legal entity / regulatory boundary and approved Region, with entity-specific VPCs, KMS keys, databases, backup vaults and IAM roles. Shared network and security services are centrally operated but do not host regulated application payloads by default.

## Alternatives

- **Shared account / shared database with tenant_id:** cheapest and simplest, but higher blast radius and greater risk of cross-tenant access or misconfiguration.
- **Shared account / separate VPC and database:** improved network/data separation but account-level IAM, quotas, audit and billing boundaries remain shared.
- **Separate accounts per entity:** stronger isolation, billing and governance boundary; increases account/network operations and cross-account deployment complexity.

## Rationale

The scenario explicitly requires entity isolation and regulated data residency. AWS accounts are a resource isolation and blast-radius boundary and support independent billing and policy. Account separation is combined with Region restrictions, entity KMS keys, resource policies and application authorisation; it is not treated as sufficient alone.

## Consequences

- Account vending and policy automation are mandatory.
- Central platform team operates shared services with delegated, audited access.
- Cross-entity analytics requires explicit data owner approval, minimisation and controlled export.
- Cost allocation and quotas are clearer.
- Multi-account global tables require special setup and support MREC only; do not assume MRSC can span entity accounts.

## Review triggers

A regulator requires stronger isolation, an entity's data classification changes, operational overhead becomes disproportionate, or AWS account-level service constraints change.
