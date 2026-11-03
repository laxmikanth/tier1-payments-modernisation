# Threat and Control Model

## 1. Threat-control matrix

| Threat / scenario | Preventive controls | Detective controls | Responsive controls | Owner / evidence |
|---|---|---|---|---|
| Cross-entity data access / tenant escape | Separate accounts, VPCs, keys and stores; trusted entity context; ABAC; row/object authorization; deny-by-default | CloudTrail data events where required, application access logs, cross-tenant tests, Access Analyzer | Revoke role, isolate account/resource, incident response and customer/regulator assessment | Product + Security; access tests and audit logs |
| Compromised privileged identity | IdP federation, phishing-resistant MFA, JIT access, least privilege, SCPs, no standing admin | GuardDuty, CloudTrail, anomalous role/session alerts, root-use alerts | Disable identity/session, rotate secrets, isolate account, preserve evidence | IAM team; access reviews and incident timeline |
| Data exfiltration | Private subnets/endpoints, egress allow-list, Network Firewall, KMS, S3 Block Public Access, DLP where applicable | GuardDuty, VPC Flow Logs, DNS logs, S3 data events, SIEM analytics | Revoke access, block route/endpoint, key rotation, legal/compliance escalation | Network/Security; flow and access evidence |
| Ransomware / destructive admin | Separate backup account, Vault Lock, immutable S3, least privilege, MFA/JIT, protected IaC | CloudTrail, Config drift, GuardDuty, backup deletion alerts | Isolate workloads, recover from clean immutable backup, rebuild from signed artefacts | Resilience + Security; restore test evidence |
| Payment replay / duplicate posting | Idempotency key, unique constraints, signed request, nonce/timestamp, sequence control | Duplicate rate, idempotency conflicts, ledger reconciliation | Quarantine, reverse with compensating journal, investigate caller | Payments Product; duplicate and reconciliation reports |
| API abuse / DDoS | WAF managed/custom rules, Shield, rate limits, quotas, authentication, payload limits | WAF/Shield metrics, API error/latency alarms, anomaly detection | Rate-limit/block, scale safe capacity, incident comms | Edge/API team; attack runbook |
| Supply-chain compromise | Pinned dependencies, SCA/SAST, SBOM, signed images, provenance, trusted registries, protected branches | ECR image scanning, code scanning, dependency alerts, admission verification | Quarantine image, revoke signing identity, roll back to known-good digest, investigate provenance | DevSecOps; SBOM and attestations |
| Vulnerable container / workload escape | Non-root, read-only FS, seccomp, no privileged mode, resource limits, network policies, patched base image | Runtime threat detection, EKS audit logs, Inspector findings | Isolate pod/node, rotate credentials, rebuild from clean image | Platform + Product; policy reports |
| Hybrid network compromise | Redundant private links, BGP filters, route segmentation, firewall inspection, mTLS | Flow logs, firewall alerts, BGP monitoring, synthetic probes | Withdraw routes, use VPN backup, isolate affected segment | Network Ops; route and failover evidence |
| Insider misuse of regulated data | Segregation of duties, dual approval, data masking, break-glass expiry, restricted support access | Data access audit, query monitoring, privileged session recording where available | Suspend access, preserve audit, HR/Legal/Compliance process | Data Owner + Security |
| Key misuse / deletion | Separate key admins, deletion waiting period, key policies, multi-person approval, no workload key admin | CloudTrail KMS events, Config, key-state alarms | Disable/re-enable under procedure, recover encrypted backups, rotate affected keys | Crypto team; key inventory and logs |
| Region split-brain | Single-writer ownership, fencing tokens, lease/epoch, quorum where applicable, failover approval | Writer identity/epoch telemetry, duplicate sequence alarms | Fence old writer, halt posting, reconcile before resuming | Data + SRE; DR exercise evidence |

## 2. Control-plane foundations

- AWS Organizations SCPs enforce Region allow-list, deny public S3, prevent disabling audit controls and restrict high-risk actions.
- AWS Control Tower controls and AWS Config conformance packs provide preventive/detective guardrails.
- CloudTrail organization trail is centralized into a dedicated Log Archive account with restricted delete and retention controls.
- GuardDuty, Security Hub and Inspector are centrally administered with delegated administrator accounts.
- IAM Access Analyzer identifies unintended resource sharing and external access.
- AWS Systems Manager provides controlled fleet access; avoid open SSH/RDP and unmanaged bastion hosts.
- AWS KMS key policies are separated from application roles; key administration is distinct from data administration.

## 3. Privileged access model

1. Workforce authenticates to bank IdP with MFA.
2. IAM Identity Center grants time-bound permission set based on approved group and ticket.
3. Production elevation requires approval by service owner and, for sensitive data/key operations, a second approver.
4. Session is short-lived, attributed to named individual and recorded in CloudTrail.
5. Break-glass credentials are stored in approved vault, tested periodically and alerted on every use.
6. Access is automatically revoked at expiry; access review is performed quarterly and after role changes.

## 4. Supply-chain security gates

- Source: branch protection, signed commits where required, mandatory review, secret scanning, SAST and licence policy.
- Build: isolated ephemeral runners, OIDC federation, dependency pinning, SBOM generation (SPDX/CycloneDX), provenance attestation.
- Artefact: immutable ECR digest, image signature, vulnerability scan, approved base image and retention.
- Deploy: verify signature/provenance, admission policy, IaC plan review, environment approval and canary.
- Runtime: least-privilege workload identity, no shell access by default, runtime threat monitoring and rapid revocation.
- Response: quarantine vulnerable digest, rebuild from clean dependency set, deploy known-good image and retain forensic evidence.

## 5. Residual risks

- Cloud provider control-plane compromise is outside the bank's direct control; mitigate through shared responsibility understanding, independent audit evidence, data minimisation and recovery planning.
- Zero-day vulnerabilities cannot be fully prevented; prioritise rapid detection, isolation and immutable rebuild.
- Human approval can be bypassed if emergency controls are poorly designed; test break-glass and segregation of duties.
- Central logging itself is a high-value target; isolate account, access and keys, and test log integrity.
