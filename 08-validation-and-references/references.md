# References

The following AWS documentation and guidance informed the design. Check the live documentation for current service limits, Region availability, feature restrictions and pricing before implementation.

1. AWS Well-Architected Framework — https://docs.aws.amazon.com/wellarchitected/
2. AWS Well-Architected Framework (six pillars) — https://wa.aws.amazon.com/index.en.html
3. AWS Control Tower multi-account strategy — https://docs.aws.amazon.com/controltower/latest/userguide/aws-multi-account-landing-zone.html
4. AWS Control Tower landing zone planning — https://docs.aws.amazon.com/controltower/latest/userguide/planning-your-deployment.html
5. AWS Control Tower overview and controls — https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html
6. Amazon Aurora Global Database disaster recovery, switchover and failover — https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html
7. Amazon Aurora availability and durability FAQ — https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html
8. DynamoDB Global Tables: how it works and consistency modes — https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html
9. DynamoDB Global Tables design best practices — https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-global-table-design.html
10. AWS Direct Connect resiliency recommendations — https://aws.amazon.com/directconnect/resiliency-recommendation/
11. AWS Network Firewall deployment architecture guidance — https://aws.github.io/aws-security-services-best-practices/guides/network-firewall/deployment-architecture/docs/
12. AWS Backup logically air-gapped vault announcement and supported services/Regions — https://aws.amazon.com/about-aws/whats-new/2024/08/general-availability-aws-backup-logically-air-gapped-vault/
13. AWS Security Hub documentation — https://docs.aws.amazon.com/securityhub/
14. Amazon GuardDuty documentation — https://docs.aws.amazon.com/guardduty/
15. AWS IAM Identity Center documentation — https://docs.aws.amazon.com/singlesignon/
16. Amazon EKS best practices guide — https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html
17. Amazon ECS documentation — https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
18. AWS Lambda best practices — https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html
19. AWS Prescriptive Guidance — https://docs.aws.amazon.com/prescriptive-guidance/

## Source interpretation notes

- AWS Control Tower is used as landing-zone orchestration over Organizations and related services; account design remains a bank governance decision.
- AWS Direct Connect resiliency guidance recommends multiple locations, redundant devices/providers and dynamic routing for critical production workloads.
- Aurora Global Database's cross-Region replication is asynchronous. Planned switchover and unplanned failover have different RPO characteristics.
- DynamoDB MRSC synchronously replicates item changes to another Region before success and supports RPO zero, but requires exactly three Regions and does not support transaction APIs. This distinction is central to the ledger ADR.
- Network Firewall centralized inspection must be routed symmetrically and validated for the selected Transit Gateway attachment pattern.
- Backup immutability, logically air-gapped vault availability and supported resource types vary by Region and service.

## Citation note

AWS service documentation is authoritative for service behaviour but does not replace legal advice, regulator interpretation, bank policy, formal service commitments or workload-specific testing.
