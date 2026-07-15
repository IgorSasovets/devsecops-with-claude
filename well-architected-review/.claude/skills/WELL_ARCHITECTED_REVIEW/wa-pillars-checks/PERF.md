# Well-Architected Review — Performance Efficiency Pillar Checks
#
# Source:  AWS Well-Architected Framework, Performance Efficiency Pillar
#          https://docs.aws.amazon.com/wellarchitected/latest/framework/a-performance-efficiency.html
# Version: 2024 (WAF revision; check https://aws.amazon.com/architecture/well-architected
#          for the latest edition)
# Scope:   9 checks across PERF 1–4 (Architecture Selection, Compute, Data Management,
#          Networking and Content Delivery).
#          Checks are limited to what is verifiable via read-only AWS CLI evidence.
#          PERF 5 (Process and Culture — benchmarking cadence, load testing practice)
#          is excluded — requires human interview and cannot be inferred from infrastructure
#          state alone.
# Format:  Each check entry: ID | question reference | best practice title | evaluation logic
#          Evidence file references use paths relative to <output-dir>/raw/.

---

## PERF 1 — Architecture Selection

### PERF01-BP01 — Use a data-driven approach — metrics and dashboards present
**WAF Question:** PERF 1. How do you select the best performing architecture?
**Evidence files:** `<REGION>/cloudwatch_alarms.json`, `<REGION>/cloudwatch_dashboards.json`
  (all regions)

This check uses the same evidence as REL06-BP01 and REL06-BP02. Reference those results:
- PASS if CloudWatch alarms exist (REL06-BP01 = PASS) AND dashboards exist
  (REL06-BP02 = PASS) — data is being collected and visualised for architecture decisions.
- WARNING if either alarms or dashboards are absent — architecture choices cannot
  be reliably validated or improved without telemetry.
- SKIPPED if both evidence files are `_error`.

---

## PERF 2 — Compute and Hardware

### PERF02-BP01 — Select the best compute option — serverless / Lambda adoption
**WAF Question:** PERF 2. How do you select and use compute resources efficiently?
**Evidence files:** `<REGION>/lambda_functions.json`, `<REGION>/ec2_instances.json`
  (all regions)

For each region: read `lambda_functions.json` and count entries in `Functions`.
Read `ec2_instances.json` and count running instances.
- WARNING if EC2 instances exist across all regions but zero Lambda functions found
  anywhere (serverless options may not have been evaluated for event-driven or
  short-lived compute workloads).
- PASS if Lambda functions are present in at least one region.
- Informational: report Lambda function count and EC2 instance count per region.

### PERF02-BP02 — Consider advanced compute — Graviton (arm64) instances
**WAF Question:** PERF 2. How do you select and use compute resources efficiently?
**Evidence files:** `<REGION>/ec2_instances.json` (all regions)

This check uses the same evidence as COST03-BP01. Reference that result:
- PASS if arm64 instances are present (Graviton evaluated and adopted).
- WARNING if all instances are x86_64 (Graviton not adopted; up to 40% better
  price/performance available for supported workloads).
- Informational: report instance type distribution.

### PERF02-BP03 — Evaluate capacity commitment models — Spot and Reserved usage
**WAF Question:** PERF 2. How do you select and use compute resources efficiently?
**Evidence files:** `<REGION>/ec2_spot_requests.json`,
  `<REGION>/ec2_reserved_instances.json` (all regions)

For each region: read `ec2_spot_requests.json` and check `SpotInstanceRequests`.
Read `ec2_reserved_instances.json` and check `ReservedInstances`.
- WARNING if no Spot or Reserved Instance requests exist in any region and
  EC2 instances are running (all capacity On-Demand — no capacity optimization
  for predictable or interruptible workloads).
- PASS if at least one active Spot request or Reserved Instance is found in
  any region.
- Informational: note Spot request count and Reserved Instance count.

---

## PERF 3 — Data Management

### PERF03-BP01 — Use purpose-built data stores — caching layer present
**WAF Question:** PERF 3. How do you store, manage, and access data?
**Evidence files:** `<REGION>/elasticache_clusters.json`,
  `<REGION>/rds_instances.json` (all regions)

For each region: read `elasticache_clusters.json` and check `CacheClusters`.
Read `rds_instances.json` to confirm whether RDS is in use:
- WARNING if RDS instances exist but no ElastiCache clusters found in any region
  (relational DB reads likely not cached; read load hits the primary DB directly).
- PASS if ElastiCache clusters are present.
- PASS if no RDS instances exist (caching concern not applicable).
- Informational: report cluster count and engine types (Redis vs Memcached).

### PERF03-BP02 — Use read replicas for read-heavy relational workloads
**WAF Question:** PERF 3. How do you store, manage, and access data?
**Evidence files:** `<REGION>/rds_instances.json` (all regions)

For each region, read `rds_instances.json`. For each instance in `DBInstances`,
inspect `ReadReplicaDBInstanceIdentifiers`:
- WARNING if any DB instance has an empty `ReadReplicaDBInstanceIdentifiers` list
  and the instance class is memory-optimised (db.r*) or larger than db.t*
  (suggests a sizeable DB with potentially high read load but no read scaling).
  Cite DBInstanceIdentifier and DBInstanceClass.
- PASS if at least one DB has read replicas configured, or no RDS instances exist.
Note: whether replicas are needed depends on workload read/write ratio — flag
for human review rather than issuing a FAIL.

---

## PERF 4 — Networking and Content Delivery

### PERF04-BP01 — Use edge caching — CloudFront distributions configured
**WAF Question:** PERF 4. How do you select and configure networking resources?
**Evidence files:** `cloudfront_distributions.json`

Read `cloudfront_distributions.json`. Check `DistributionList.Items`:
- WARNING if no distributions found (static assets or API responses not cached
  at the edge; all requests reach origin, increasing latency for global users).
- PASS if at least one distribution exists.
- SKIPPED if file is `_error`.

### PERF04-BP02 — Reduce network distance — VPC Endpoints for AWS services
**WAF Question:** PERF 4. How do you select and configure networking resources?
**Evidence files:** `<REGION>/ec2_vpc_endpoints.json` (all regions)

This check uses the same evidence as SEC03-BP07. Reference that result:
- PASS if VPC endpoints are present (traffic to S3, DynamoDB, or other AWS
  services stays on the AWS backbone, reducing latency and cost).
- WARNING if no endpoints found (AWS service traffic traverses the public
  internet, adding latency and egress costs).

### PERF04-BP03 — Optimise network performance — ENA enhanced networking on EC2
**WAF Question:** PERF 4. How do you select and configure networking resources?
**Evidence files:** `<REGION>/ec2_instances.json` (all regions)

For each region, read `ec2_instances.json`. For each running instance in
`Reservations[].Instances[]`:
- Identify instances with an `InstanceType` of medium or larger (skip nano/micro).
- Check `EnaSupport` field: `true` means Enhanced Networking (ENA) is enabled.
- WARNING if any medium-or-larger instance has `EnaSupport: false` (older instance
  type or configuration lacking higher bandwidth and lower latency networking).
  Cite InstanceId and InstanceType.
- PASS if all medium-or-larger instances have ENA enabled, or no qualifying
  instances exist.
