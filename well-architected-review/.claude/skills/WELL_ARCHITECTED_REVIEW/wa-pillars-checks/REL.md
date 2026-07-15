# Well-Architected Review — Reliability Pillar Checks
#
# Source:  AWS Well-Architected Framework, Reliability Pillar
#          https://docs.aws.amazon.com/wellarchitected/latest/framework/a-reliability.html
# Version: 2024 (WAF revision; check https://aws.amazon.com/architecture/well-architected
#          for the latest edition)
# Scope:   14 checks across REL 1–10 (Foundations, Change Management, Failure Management).
#          Checks are limited to what is verifiable via read-only AWS CLI evidence.
#          REL 3–5 (Workload Architecture — service dependencies, loose coupling, error
#          budgets) are excluded — they require architectural interviews and load testing
#          that cannot be inferred from infrastructure state alone.
# Format:  Each check entry: ID | question reference | best practice title | evaluation logic
#          Evidence file references use paths relative to <output-dir>/raw/.

---

## REL 1 — Foundations: Service Quotas

### REL01-BP01 — Aware of service quotas and constraints
**WAF Question:** REL 1. How do you manage Service Quotas and constraints?
**Evidence files:** `<REGION>/service_quotas_ec2.json` (all regions)

Read `service_quotas_ec2.json` for each region. Check
`RequestedQuotas` list:
- WARNING if the list is empty in all regions (no quota increase requests found,
  suggesting no proactive quota monitoring or management).
- PASS if quota change requests exist in at least one region (signals awareness
  and proactive management of service limits).
- SKIPPED if file is `_error`.

---

## REL 2 — Foundations: Network Topology

### REL02-BP01 — Use highly available network connectivity — multi-AZ subnets
**WAF Question:** REL 2. How do you plan your network topology?
**Evidence files:** `<REGION>/ec2_vpcs.json`, `<REGION>/ec2_subnets.json` (all regions)

For each region: read `ec2_vpcs.json` to list VPCs. Read `ec2_subnets.json` and
group subnets by `VpcId`. For each VPC, collect the set of distinct
`AvailabilityZone` values across its subnets:
- FAIL if any VPC has subnets in only one AZ (single point of failure at
  network layer).
- WARNING if any VPC has subnets in fewer than two AZs.
- PASS if every VPC has subnets spanning at least two distinct AZs.

### REL02-BP03 — Ensure IP subnet allocation accounts for expansion
**WAF Question:** REL 2. How do you plan your network topology?
**Evidence files:** `<REGION>/ec2_subnets.json` (all regions)

Read `ec2_subnets.json` for each region. For each subnet in `Subnets`:
- WARNING if `AvailableIpAddressCount` < 16 in any subnet (very little headroom
  for new resources; may block scaling events). Cite SubnetId and available count.
- PASS if all subnets have at least 16 available IP addresses.

### REL02-BP05 — Prefer hub-and-spoke topology for multi-VPC networks
**WAF Question:** REL 2. How do you plan your network topology?
**Evidence files:** `<REGION>/ec2_transit_gateways.json`, `<REGION>/ec2_vpc_peering.json`
  (all regions)

Read `ec2_vpcs.json` for VPC count per region. Read `ec2_transit_gateways.json`
and `ec2_vpc_peering.json`:
- If more than 3 VPCs exist and only VPC peering connections are present
  (no Transit Gateway): WARNING — a full-mesh peering model may become
  unmanageable; Transit Gateway is recommended.
- If Transit Gateways are present: PASS (hub-and-spoke model in use).
- If fewer than 3 VPCs: PASS (topology concern not yet applicable).
- Informational: note VPC count, TGW count, and peering connection count.

---

## REL 6 — Change Management: Monitoring

### REL06-BP01 — Monitor all components — CloudWatch alarms configured
**WAF Question:** REL 6. How do you monitor workload resources?
**Evidence files:** `<REGION>/cloudwatch_alarms.json` (all regions)

Read `cloudwatch_alarms.json` for each region. Count total alarms across all regions:
- WARNING if fewer than 5 alarms exist across all checked regions (suggests
  minimal or absent monitoring posture).
- PASS if alarms are present across regions.
- SKIPPED if all files are `_error`.

### REL06-BP02 — Define and calculate metrics — CloudWatch dashboards exist
**WAF Question:** REL 6. How do you monitor workload resources?
**Evidence files:** `<REGION>/cloudwatch_dashboards.json` (all regions)

Read `cloudwatch_dashboards.json` for each region. Check `DashboardEntries` list:
- WARNING if no dashboards exist in any region (no consolidated metric view).
- PASS if at least one dashboard exists.

### REL06-BP03 — Send notifications — SNS topics configured
**WAF Question:** REL 6. How do you monitor workload resources?
**Evidence files:** `<REGION>/sns_topics.json` (all regions)

Read `sns_topics.json` for each region. Check `Topics` list:
- WARNING if no SNS topics exist in any region (CloudWatch alarms may have no
  notification target — alerts go undelivered).
- PASS if at least one SNS topic exists.

---

## REL 7 — Change Management: Scaling

### REL07-BP01 — Use autoscaling — Auto Scaling Groups configured
**WAF Question:** REL 7. How do you design your workload to adapt to changes in demand?
**Evidence files:** `<REGION>/ec2_instances.json`, `<REGION>/autoscaling_groups.json`
  (all regions)

For each region: read `ec2_instances.json` to check if EC2 instances exist.
Read `autoscaling_groups.json`:
- WARNING if EC2 instances are present but `AutoScalingGroups` list is empty
  (instances running without autoscaling, vulnerable to demand spikes).
- PASS if Auto Scaling Groups are configured, or no EC2 instances are present.

### REL07-BP02 — Use load balancing — ALB/NLB present
**WAF Question:** REL 7. How do you design your workload to adapt to changes in demand?
**Evidence files:** `<REGION>/ec2_instances.json`, `<REGION>/elbv2_load_balancers.json`
  (all regions)

For each region: check if EC2 instances exist. Read `elbv2_load_balancers.json`:
- WARNING if EC2 instances are present but `LoadBalancers` list is empty
  (single-instance endpoints, no load distribution or health checking).
- PASS if load balancers are present, or no EC2 instances are running.

---

## REL 8 — Change Management: Deployments

### REL08-BP01 — Use deployment management — CI/CD pipeline configured
**WAF Question:** REL 8. How do you implement change?
**Evidence files:** `<REGION>/codepipeline_pipelines.json` (all regions)

Read `codepipeline_pipelines.json` for each region. Check `pipelines` list:
- WARNING if no pipelines found in any region (suggests manual deployments,
  which increase risk of human error during change events).
- PASS if at least one pipeline exists.

### REL08-BP02 — Test using a deployment pipeline — Infrastructure as Code in use
**WAF Question:** REL 8. How do you implement change?
**Evidence files:** `<REGION>/cfn_stacks.json` (all regions)

Read `cfn_stacks.json` for each region:
- WARNING if no CloudFormation stacks found in any region (IaC adoption absent;
  manual resource creation cannot be consistently reproduced or tested).
- PASS if stacks exist in at least one region.

---

## REL 9 — Failure Management: Backups

### REL09-BP01 — Back up data — RDS automated backups
**WAF Question:** REL 9. How do you back up data?
**Evidence files:** `<REGION>/rds_instances.json` (all regions)

For each region, read `rds_instances.json`. For each instance in `DBInstances`:
- FAIL if `BackupRetentionPeriod` = 0 (automated backups disabled). Cite
  DBInstanceIdentifier.
- WARNING if `BackupRetentionPeriod` is between 1 and 6 days (below the
  commonly recommended 7-day minimum).
- PASS if all instances have `BackupRetentionPeriod` ≥ 7, or no RDS instances exist.

### REL09-BP02 — Back up data — RDS Multi-AZ deployment
**WAF Question:** REL 9. How do you back up data?
**Evidence files:** `<REGION>/rds_instances.json` (all regions)

For each region, read `rds_instances.json`. For each instance in `DBInstances`:
- WARNING if any instance has `MultiAZ: false` (single-AZ DB; an AZ failure
  causes downtime). Cite DBInstanceIdentifier and engine.
- PASS if all instances are Multi-AZ, or no RDS instances exist.
Note: single-AZ may be intentional for non-production; flag for human review.

### REL09-BP03 — Back up data — S3 versioning enabled
**WAF Question:** REL 9. How do you back up data?
**Evidence files:** `s3_bucket_<NAME>.json` (all sampled buckets, `versioning` field)

For each per-bucket file, read the `versioning` section:
- WARNING if `Status` is absent, `"Suspended"`, or the section is `_error`.
  Cite bucket name.
- PASS if `Status: Enabled` for all checked buckets.

### REL09-BP04 — Back up data — DynamoDB Point-in-Time Recovery
**WAF Question:** REL 9. How do you back up data?
**Evidence files:** `<REGION>/dynamodb_tables.json` (all regions)

For each region: read `dynamodb_tables.json` to get table names. For each table,
the skill should issue:
```bash
$CLI_BASE dynamodb describe-continuous-backups \
  --table-name <TABLE_NAME> --region <REGION> --output json
```
Cap at 20 tables per region. For each result:
- FAIL if `ContinuousBackupsStatus` is `DISABLED` or `PointInTimeRecoveryStatus`
  is `DISABLED`. Cite table name.
- PASS if all checked tables have PITR `ENABLED`, or no DynamoDB tables exist.
- SKIPPED if the original `dynamodb_tables.json` is `_error`.

### REL09-BP05 — Back up data — AWS Backup plans configured
**WAF Question:** REL 9. How do you back up data?
**Evidence files:** `<REGION>/backup_plans.json` (all regions)

Read `backup_plans.json` for each region. Check `BackupPlansList`:
- WARNING if no backup plans exist in any region (no centralised backup
  governance; individual service backup settings may be inconsistent).
- PASS if at least one backup plan exists.

---

## REL 10 — Failure Management: Disaster Recovery

### REL10-BP01 — Deploy to multiple locations — multi-AZ resource spread
**WAF Question:** REL 10. How do you use fault isolation to protect your workload?
**Evidence files:** `<REGION>/ec2_instances.json`, `<REGION>/rds_instances.json`
  (all regions)

For each region: read `ec2_instances.json`. Collect the set of distinct
`Placement.AvailabilityZone` values across running instances:
- WARNING if all running instances are in a single AZ (entire compute layer
  lost in an AZ failure). Cite region and AZ.
- PASS if instances are spread across at least two distinct AZs, or no instances.

### REL10-BP04 — Use defined recovery strategies — RDS manual snapshots exist
**WAF Question:** REL 10. How do you use fault isolation to protect your workload?
**Evidence files:** `<REGION>/rds_snapshots.json` (all regions)

Read `rds_snapshots.json` for each region. Check `DBSnapshots` list:
- WARNING if no manual snapshots found and RDS instances exist (no user-initiated
  recovery point outside the automated backup window).
- PASS if manual snapshots exist, or no RDS instances are present.

### REL10-BP05 — Test DR strategies — S3 cross-region replication
**WAF Question:** REL 10. How do you use fault isolation to protect your workload?
**Evidence files:** `s3_bucket_<NAME>.json` (all sampled buckets, `replication` field)

For each per-bucket file, read the `replication` section:
- WARNING if no buckets have cross-region replication configured (no automated
  data copy to a secondary region for regional-failure scenarios).
- PASS if at least one bucket has replication rules targeting a different region.
- Informational: note count of buckets with and without replication.
