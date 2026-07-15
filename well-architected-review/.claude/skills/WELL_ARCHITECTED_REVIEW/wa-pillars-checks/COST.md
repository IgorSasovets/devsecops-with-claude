# Well-Architected Review — Cost Optimization Pillar Checks
#
# Source:  AWS Well-Architected Framework, Cost Optimization Pillar
#          https://docs.aws.amazon.com/wellarchitected/latest/framework/a-cost-optimization.html
# Version: 2024 (WAF revision; check https://aws.amazon.com/architecture/well-architected
#          for the latest edition)
# Scope:   10 checks across COST 1–5 (Financial Management, Usage Awareness, Resource
#          Efficiency, Demand Management, Optimization Over Time).
#          Checks are limited to what is verifiable via read-only AWS CLI evidence.
#          COST 1 partial (finance/engineering collaboration culture, showback/chargeback
#          processes) is excluded — requires organisational interview.
# Format:  Each check entry: ID | question reference | best practice title | evaluation logic
#          Evidence file references use paths relative to <output-dir>/raw/.

---

## COST 1 — Practice Cloud Financial Management

### COST01-BP01 — Establish a cost optimization function — Budgets configured
**WAF Question:** COST 1. How do you implement cloud financial management?
**Evidence files:** `budgets.json`

Read `budgets.json`. Check `Budgets` list:
- FAIL if list is empty (no budgets defined — no cost guardrails or alerts).
- WARNING if fewer than 2 budgets exist (minimal coverage; typically want at
  least one for total account spend and one for per-service or per-environment).
- PASS if 2 or more budgets are configured.
- SKIPPED if file is `_error`.

### COST01-BP03 — Establish budgets and forecasts — cost allocation tags active
**WAF Question:** COST 1. How do you implement cloud financial management?
**Evidence files:** `cost_allocation_tags.json`

Read `cost_allocation_tags.json`. Check `CostAllocationTags` list:
- WARNING if the list is empty (no cost allocation tags activated — spend cannot
  be attributed to teams, projects, or environments in Cost Explorer).
- PASS if at least one tag is active.
- SKIPPED if file is `_error`.

---

## COST 2 — Expenditure and Usage Awareness

### COST02-BP01 — Govern usage — SCPs limit unused or unapproved services
**WAF Question:** COST 2. How do you govern usage?
**Evidence files:** `org_scps.json`

Read `org_scps.json`. Check `Policies` list:
- WARNING if the list is empty or file is `_error` (no Service Control Policies
  configured — unrestricted service usage allowed organisation-wide).
- PASS if at least one SCP exists (governance framework in place).
Note: SCP content is not evaluated here — existence is the signal.

### COST02-BP03 — Monitor cost proactively — CloudWatch billing alarms
**WAF Question:** COST 2. How do you govern usage?
**Evidence files:** `<REGION>/cloudwatch_alarms.json` (check us-east-1 specifically,
  as billing metrics are only published in us-east-1)

Read `cloudwatch_alarms.json` for `us-east-1` (billing alarms are only available
in this region). Search `MetricAlarms` for any alarm with `Namespace: AWS/Billing`:
- WARNING if no billing alarms found (unexpected cost spikes go undetected).
- PASS if at least one billing alarm is configured.
- SKIPPED if `cloudwatch_alarms.json` for us-east-1 is `_error`.

---

## COST 3 — Cost-Effective Resources (Right-Sizing and Waste)

### COST03-BP01 — Select the best compute option — Graviton (arm64) usage
**WAF Question:** COST 3. How do you evaluate cost when selecting services?
**Evidence files:** `<REGION>/ec2_instances.json` (all regions)

For each region, read `ec2_instances.json`. Inspect `Architecture` field
for each instance in `Reservations[].Instances[]`:
- WARNING if all running instances are `x86_64` and none are `arm64`
  (Graviton instances not adopted; typically 20% cheaper for equivalent
  workloads with up to 40% better price/performance).
- PASS if at least one `arm64` (Graviton) instance is running.
- Informational: report total x86_64 vs arm64 instance counts.

### COST03-BP02 — Eliminate orphaned resources — unattached EBS volumes
**WAF Question:** COST 3. How do you evaluate cost when selecting services?
**Evidence files:** `<REGION>/ec2_volumes.json` (all regions)

For each region, read `ec2_volumes.json`. For each volume in `Volumes`:
- FAIL if any volume has `State: available` (unattached — costs incurred
  with no workload benefit). List each VolumeId, SizeGiB, and region.
- PASS if all volumes are attached (`State: in-use`), or no volumes exist.

### COST03-BP03 — Eliminate orphaned resources — unassociated Elastic IPs
**WAF Question:** COST 3. How do you evaluate cost when selecting services?
**Evidence files:** `<REGION>/ec2_eips.json` (all regions)

For each region, read `ec2_eips.json`. For each address in `Addresses`:
- FAIL if any EIP has no `AssociationId` field (unassociated — AWS charges
  for unattached EIPs). List each PublicIp and region.
- PASS if all EIPs are associated, or no EIPs exist.

### COST03-BP04 — Identify idle load balancers
**WAF Question:** COST 3. How do you evaluate cost when selecting services?
**Evidence files:** `<REGION>/elbv2_load_balancers.json`,
  `<REGION>/elbv2_target_groups.json` (all regions)

For each region: read `elbv2_load_balancers.json` to get all load balancers.
Read `elbv2_target_groups.json` and group target groups by `LoadBalancerArn`.
- WARNING if any load balancer has no associated target groups (paying for an
  LB with no registered targets). Cite LoadBalancerArn and Name.
- PASS if all load balancers have at least one target group, or no load balancers exist.
Note: cannot determine if targets are healthy from list commands alone — flag
load balancers without target groups only.

### COST03-BP05 — Optimise storage tiers — S3 lifecycle policies configured
**WAF Question:** COST 3. How do you evaluate cost when selecting services?
**Evidence files:** `s3_bucket_<NAME>.json` (all sampled buckets, `lifecycle` field)

For each per-bucket file, read the `lifecycle` section:
- Count buckets where `lifecycle` is `_error` or has no `Rules` entries.
- WARNING if more than 30% of checked buckets have no lifecycle configuration
  (data accumulating in S3 Standard with no transition to cheaper storage classes).
- PASS if lifecycle rules are configured on at least 70% of checked buckets.

### COST03-BP06 — Identify aged EBS snapshots
**WAF Question:** COST 3. How do you evaluate cost when selecting services?
**Evidence files:** `<REGION>/ec2_snapshots.json` (all regions)

For each region, read `ec2_snapshots.json`. For each snapshot in `Snapshots`:
- WARNING if any snapshot has a `StartTime` older than 180 days and its
  `Description` does not follow a recognisable retention naming convention
  (e.g. "Automated by AWS Backup"). Cite SnapshotId, StartTime, and VolumeSize.
- PASS if no snapshots older than 180 days exist, or all aged snapshots appear
  intentional (backup convention naming).
- Informational: report total snapshot count and oldest snapshot date.

---

## COST 4 — Manage Demand and Supply Resources

### COST04-BP01 — Use automatic scaling — ASGs configured
**WAF Question:** COST 4. How do you manage demand and supply resources?
**Evidence files:** `<REGION>/autoscaling_groups.json` (all regions)

This check shares evidence with REL07-BP01. Reference the REL07-BP01 result:
- If REL07-BP01 = PASS → COST04-BP01 = PASS (scaling configured prevents
  over-provisioning for demand).
- If REL07-BP01 = WARNING → COST04-BP01 = WARNING (fixed-size fleet likely
  over-provisioned for off-peak periods).
- If REL07-BP01 = SKIPPED → COST04-BP01 = SKIPPED.

---

## COST 5 — Optimize Over Time

### COST05-BP01 — Adopt commitment-based pricing — Savings Plans or Reserved Instances
**WAF Question:** COST 5. How do you evaluate new services?
**Evidence files:** `savings_plans.json`, `<REGION>/ec2_reserved_instances.json` (all regions),
  `<REGION>/ec2_instances.json` (all regions)

Read `savings_plans.json` to check `savingsPlans` list.
Read `ec2_reserved_instances.json` per region to check `ReservedInstances` list.
Read `ec2_instances.json` per region to establish if EC2 instances are running.

- WARNING if running EC2 instances exist but `savingsPlans` is empty AND
  all `ReservedInstances` lists are empty (all capacity On-Demand — no
  commitment pricing, typically 20–40% more expensive than equivalent RIs or
  Savings Plans for steady-state workloads).
- PASS if at least one active Savings Plan or Reserved Instance is found.
- PASS if no EC2 instances are running (commitment not applicable).
- Informational: note count of active Savings Plans and Reserved Instances by region.
