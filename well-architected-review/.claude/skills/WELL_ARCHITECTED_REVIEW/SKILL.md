---
name: well-architected-review
description: >-
  Automated AWS Well-Architected Review (WAR) using read-only AWS CLI commands.
  Collects evidence across Security, Reliability, Cost Optimization, and
  Performance Efficiency pillars. Fans out parallel subagent analysis per pillar,
  maps findings to WAF best practice IDs, and writes WAR-REPORT.md + WAR-REPORT.json.
  Requires SecurityAudit + ReadOnlyAccess IAM policies (or equivalent).
  Use when asked to "run a well-architected review", "WAR check", "audit my AWS account",
  or "check AWS best practices".
context: fork
argument-hint: "--regions <r1,r2,...> [--profile <name>] [--pillars <sec,rel,cost,perf>] [--skip-service <svc1,svc2>] [--skip-check <BP-ID1,BP-ID2>] [--agents <1-6>] [--output-dir <path>]"
allowed-tools:
  - Read
  - Write
  - Bash(aws:*)
  - Bash(echo:*)
  - Bash(date:*)
  - Bash(mkdir:*)
  - Bash(cat:*)
  - Bash(jq:*)
  - Task
---

# /well-architected-review

Automated evidence collection and analysis for the AWS Well-Architected Framework.
Runs **read-only** AWS CLI calls only — never creates, modifies, or deletes resources.

**Required IAM permissions:** AWS managed policies `SecurityAudit` + `ReadOnlyAccess`
(or equivalent read-only custom policy). The skill will fail gracefully on any denied call.

**Pillars covered:** Security (SEC), Reliability (REL), Cost Optimization (COST),
Performance Efficiency (PERF).

**Not automatable (out of scope):** Operational culture, runbooks, team structure,
game day exercises, process maturity — these require human interview and are flagged
in the Appendix section of the report.

---

## Token Efficiency Contract

This skill separates evidence collection from analysis deliberately — no file content
is loaded into the main context beyond what is needed to orchestrate subagents.

| Phase | What is read | What is skipped |
|---|---|---|
| Pre-flight | `sts get-caller-identity` output only | Nothing else |
| Step 1 (Collection) | CLI JSON written to `raw/` on disk — not loaded into context | Raw files are consumed only by subagents in Step 2 |
| Step 2 (Analysis) | Each subagent reads **only its pillar check file** (`wa-pillars-checks/<PILLAR>.md`) plus the raw evidence files it needs for its specific checks | Subagents do not read other pillars' check files or raw files outside their scope |
| Step 3 (Report) | Subagent JSON result blocks only (structured, compact) | Never re-reads raw CLI output at report generation time |

Pillar check files in `wa-pillars-checks/` are **not loaded at skill startup**.
They are passed to subagents at fan-out time only, keeping the orchestrator context lean.

---

## Arguments

- `--regions <r1,r2,...>` **(required)** — comma-separated AWS region codes to check.
  Example: `--regions us-east-1,eu-west-1`. For global services (IAM, S3, CloudFront,
  Organizations, Budgets) only the first listed region is used as the API endpoint.
- `--profile <name>` *(optional)* — AWS CLI named profile. Defaults to ambient
  environment credentials if omitted.
- `--pillars <list>` *(optional)* — comma-separated subset of pillars: `sec`, `rel`,
  `cost`, `perf`. Defaults to all four if omitted.
- `--skip-service <list>` *(optional)* — comma-separated service tokens to skip entirely.
  Valid tokens: `rds`, `elasticache`, `dynamodb`, `ec2`, `s3`, `iam`, `cloudtrail`,
  `guardduty`, `securityhub`, `waf`, `shield`, `inspector`, `lambda`, `ecr`,
  `cloudfront`, `autoscaling`, `backup`, `codepipeline`, `config`, `cloudwatch`, `ssm`.
- `--skip-check <list>` *(optional)* — comma-separated WAF best practice IDs to skip.
  Example: `--skip-check SEC02-BP05,REL09-BP02`.
- `--agents <n>` *(optional)* — number of parallel subagents for pillar analysis.
  Default: `3`. Set to `1` for sequential (cheapest). Max: `6`.
- `--output-dir <path>` *(optional)* — directory to write output files.
  Defaults to `./war-output/`.

---

## Step 0 — Pre-flight

1. Parse and validate all arguments. Abort with a clear error if `--regions` is missing
   or contains an invalid format.

2. Build the AWS CLI base command: if `--profile` was given, every CLI call in this
   skill MUST include `--profile <name>`. Store as `CLI_BASE="aws --profile <name>"`
   (or `CLI_BASE="aws"` if no profile). All subsequent Bash calls in Steps 1–2 must
   use `$CLI_BASE` as the prefix — never call `aws` directly.

3. Confirm credentials work:
   ```bash
   $CLI_BASE sts get-caller-identity --output json
   ```
   If this fails, stop immediately with a clear error. Print the Account ID,
   User/Role ARN, and User ID so the user can confirm the correct identity.
   Store `ACCOUNT_ID` from this output — it is required for several global API calls.

4. Set `GLOBAL_REGION` = first region in the `--regions` list. This is the endpoint
   used for all account-global services (IAM, S3 Control, Organizations, Budgets,
   Shield, CloudFront).

5. Create the output directory and raw evidence subdirectory:
   ```bash
   mkdir -p <output-dir>/raw
   ```

6. Resolve the skill's own directory path. The pillar check files are located at:
   ```
   <skill-dir>/wa-pillars-checks/SEC.md
   <skill-dir>/wa-pillars-checks/REL.md
   <skill-dir>/wa-pillars-checks/COST.md
   <skill-dir>/wa-pillars-checks/PERF.md
   ```
   Confirm these files exist before proceeding. If a requested pillar's check file
   is missing, skip that pillar and warn the user.

7. Tell the user:
   - Account ID and identity confirmed
   - Regions to be checked
   - Pillars to be run (and any skipped due to missing check files)
   - Agent concurrency count
   - Skipped services and check IDs (if any)
   - Output path

---

## Step 1 — Data Collection

**Collect all raw evidence before any analysis begins.**

Run all CLI commands below, storing results as JSON files under `<output-dir>/raw/`.
This cleanly separates evidence gathering from interpretation and ensures subagents
work from a stable snapshot rather than issuing their own API calls.

Wrap every call to handle errors gracefully:
```bash
$CLI_BASE <command> --output json 2>/dev/null \
  || echo '{"_error":"denied_or_not_enabled"}'
```
A response containing `"_error"` means the service is unavailable or access was
denied — subagents will treat those checks as `SKIPPED`.

Skip collection for any service whose token appears in `--skip-service`.

### 1a — Global / Account-Level (run once using GLOBAL_REGION endpoint)

```bash
# Identity & Account
$CLI_BASE iam get-account-summary --output json \
  > <output-dir>/raw/iam_account_summary.json
$CLI_BASE iam get-account-password-policy --output json \
  > <output-dir>/raw/iam_password_policy.json

# IAM Users, Roles, Policies
$CLI_BASE iam list-users --output json \
  > <output-dir>/raw/iam_users.json
$CLI_BASE iam list-roles --max-items 500 --output json \
  > <output-dir>/raw/iam_roles.json
$CLI_BASE iam list-policies --scope Local --output json \
  > <output-dir>/raw/iam_customer_policies.json
$CLI_BASE iam list-virtual-mfa-devices --output json \
  > <output-dir>/raw/iam_mfa_devices.json
$CLI_BASE accessanalyzer list-analyzers --output json \
  > <output-dir>/raw/iam_access_analyzer.json
$CLI_BASE sso-admin list-instances --output json \
  > <output-dir>/raw/iam_sso_instances.json

# Organizations & SCPs
$CLI_BASE organizations describe-organization --output json \
  > <output-dir>/raw/org_info.json
$CLI_BASE organizations list-policies \
  --filter SERVICE_CONTROL_POLICY --output json \
  > <output-dir>/raw/org_scps.json

# S3 Global
$CLI_BASE s3api list-buckets --output json \
  > <output-dir>/raw/s3_buckets.json
$CLI_BASE s3control get-public-access-block \
  --account-id $ACCOUNT_ID --output json \
  > <output-dir>/raw/s3_account_public_access.json

# Cost & Billing
$CLI_BASE budgets describe-budgets \
  --account-id $ACCOUNT_ID --output json \
  > <output-dir>/raw/budgets.json
$CLI_BASE ce list-cost-allocation-tags --output json \
  > <output-dir>/raw/cost_allocation_tags.json
$CLI_BASE savingsplans describe-savings-plans --output json \
  > <output-dir>/raw/savings_plans.json

# CloudFront (global)
$CLI_BASE cloudfront list-distributions --output json \
  > <output-dir>/raw/cloudfront_distributions.json

# WAFv2 CloudFront scope (must use us-east-1)
$CLI_BASE wafv2 list-web-acls \
  --scope CLOUDFRONT --region us-east-1 --output json \
  > <output-dir>/raw/wafv2_cloudfront_acls.json

# Shield (must use us-east-1)
$CLI_BASE shield describe-subscription \
  --region us-east-1 --output json \
  > <output-dir>/raw/shield_subscription.json
```

**Per-bucket S3 checks:** Iterate over buckets from `s3_buckets.json`.
Cap at 50 buckets — if more exist, process the first 50 and note the cap in the report.
For each bucket, collect into `<output-dir>/raw/s3_bucket_<BUCKET_NAME>.json`:

```bash
{
  "encryption":    $(... s3api get-bucket-encryption --bucket <B>   2>/dev/null || echo '{"_error":"denied_or_not_enabled"}'),
  "versioning":    $(... s3api get-bucket-versioning  --bucket <B>   2>/dev/null || echo '{"_error":"denied_or_not_enabled"}'),
  "logging":       $(... s3api get-bucket-logging     --bucket <B>   2>/dev/null || echo '{"_error":"denied_or_not_enabled"}'),
  "public_access": $(... s3api get-public-access-block --bucket <B>  2>/dev/null || echo '{"_error":"denied_or_not_enabled"}'),
  "lifecycle":     $(... s3api get-bucket-lifecycle-configuration --bucket <B> 2>/dev/null || echo '{"_error":"denied_or_not_enabled"}'),
  "replication":   $(... s3api get-bucket-replication --bucket <B>   2>/dev/null || echo '{"_error":"denied_or_not_enabled"}'),
  "policy":        $(... s3api get-bucket-policy      --bucket <B>   2>/dev/null || echo '{"_error":"denied_or_not_enabled"}')
}
```

### 1b — Per-Region Collection (loop over each region in `--regions`)

For each `REGION`, save to `<output-dir>/raw/<REGION>/`:

```bash
mkdir -p <output-dir>/raw/<REGION>

# EC2 & Networking
$CLI_BASE ec2 describe-instances         --region <REGION> --output json > ec2_instances.json
$CLI_BASE ec2 describe-security-groups   --region <REGION> --output json > ec2_security_groups.json
$CLI_BASE ec2 describe-vpcs              --region <REGION> --output json > ec2_vpcs.json
$CLI_BASE ec2 describe-subnets           --region <REGION> --output json > ec2_subnets.json
$CLI_BASE ec2 describe-network-acls      --region <REGION> --output json > ec2_nacls.json
$CLI_BASE ec2 describe-flow-logs         --region <REGION> --output json > ec2_flow_logs.json
$CLI_BASE ec2 describe-volumes           --region <REGION> --output json > ec2_volumes.json
$CLI_BASE ec2 describe-snapshots         --owner-ids self --region <REGION> --output json > ec2_snapshots.json
$CLI_BASE ec2 describe-addresses         --region <REGION> --output json > ec2_eips.json
$CLI_BASE ec2 describe-reserved-instances --region <REGION> --output json > ec2_reserved_instances.json
$CLI_BASE ec2 describe-spot-instance-requests --region <REGION> --output json > ec2_spot_requests.json
$CLI_BASE ec2 describe-vpc-peering-connections --region <REGION> --output json > ec2_vpc_peering.json
$CLI_BASE ec2 describe-transit-gateways  --region <REGION> --output json > ec2_transit_gateways.json
$CLI_BASE ec2 describe-vpc-endpoints     --region <REGION> --output json > ec2_vpc_endpoints.json

# Load Balancers & Auto Scaling
$CLI_BASE elbv2 describe-load-balancers  --region <REGION> --output json > elbv2_load_balancers.json
$CLI_BASE elbv2 describe-target-groups   --region <REGION> --output json > elbv2_target_groups.json
$CLI_BASE autoscaling describe-auto-scaling-groups --region <REGION> --output json > autoscaling_groups.json

# RDS
$CLI_BASE rds describe-db-instances      --region <REGION> --output json > rds_instances.json
$CLI_BASE rds describe-db-snapshots      --snapshot-type manual --region <REGION> --output json > rds_snapshots.json

# ElastiCache
$CLI_BASE elasticache describe-cache-clusters --region <REGION> --output json > elasticache_clusters.json

# DynamoDB
$CLI_BASE dynamodb list-tables           --region <REGION> --output json > dynamodb_tables.json

# Lambda
$CLI_BASE lambda list-functions          --region <REGION> --output json > lambda_functions.json

# CloudTrail
$CLI_BASE cloudtrail describe-trails     --include-shadow-trails --region <REGION> --output json > cloudtrail_trails.json

# GuardDuty
$CLI_BASE guardduty list-detectors       --region <REGION> --output json > guardduty_detectors.json

# Security Hub
$CLI_BASE securityhub describe-hub       --region <REGION> --output json > securityhub_hub.json
$CLI_BASE securityhub get-enabled-standards --region <REGION> --output json > securityhub_standards.json

# AWS Config
$CLI_BASE configservice describe-configuration-recorders --region <REGION> --output json > config_recorders.json
$CLI_BASE configservice describe-delivery-channels       --region <REGION> --output json > config_delivery_channels.json
$CLI_BASE configservice describe-config-rules            --region <REGION> --output json > config_rules.json

# CloudWatch
$CLI_BASE cloudwatch describe-alarms     --region <REGION> --output json > cloudwatch_alarms.json
$CLI_BASE cloudwatch list-dashboards     --region <REGION> --output json > cloudwatch_dashboards.json
$CLI_BASE logs describe-log-groups       --region <REGION> --output json > cwlogs_log_groups.json

# SNS
$CLI_BASE sns list-topics                --region <REGION> --output json > sns_topics.json

# KMS
$CLI_BASE kms list-keys                  --region <REGION> --output json > kms_keys.json

# WAFv2 Regional
$CLI_BASE wafv2 list-web-acls --scope REGIONAL --region <REGION> --output json > wafv2_regional_acls.json

# Inspector
$CLI_BASE inspector2 list-account-permissions --region <REGION> --output json > inspector2_permissions.json

# ECR
$CLI_BASE ecr describe-repositories      --region <REGION> --output json > ecr_repositories.json

# SSM
$CLI_BASE ssm describe-instance-information --region <REGION> --output json > ssm_managed_instances.json

# AWS Backup
$CLI_BASE backup list-backup-plans        --region <REGION> --output json > backup_plans.json

# CodePipeline
$CLI_BASE codepipeline list-pipelines    --region <REGION> --output json > codepipeline_pipelines.json

# CloudFormation
$CLI_BASE cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --region <REGION> --output json > cfn_stacks.json

# Service Quotas
$CLI_BASE service-quotas list-requested-service-quota-changes-by-service \
  --service-code ec2 --region <REGION> --output json > service_quotas_ec2.json

# X-Ray
$CLI_BASE xray get-sampling-rules        --region <REGION> --output json > xray_sampling.json
```

After all collection is complete, tell the user:
- Total CLI commands executed
- Commands that returned `_error` (access denied or service not enabled)
- Confirmation that analysis is ready to begin

---

## Step 2 — Pillar Analysis (Parallel Subagents)

Spawn one subagent per requested pillar, up to `--agents` concurrency.
If `--agents 1`, run pillars sequentially: SEC → REL → COST → PERF.
If `--agents > 1`, start that many in parallel and queue remaining pillars
until a slot frees.

Each subagent receives:
- Path to `<output-dir>/raw/`
- List of regions checked
- List of skipped services (`--skip-service`)
- List of skipped check IDs (`--skip-check`)
- Full contents of its pillar check file (read from `wa-pillars-checks/<PILLAR>.md`)

### Subagent Brief Template

Prepend this header to the pillar check file contents when spawning each subagent:

```
You are an AWS Well-Architected Review analyst. Your job is to read raw AWS CLI
evidence files and evaluate each best practice check against that evidence.

EVIDENCE LOCATION: <output-dir>/raw/
REGIONS CHECKED: <regions list>
SKIPPED SERVICES: <list or "none">
SKIPPED CHECK IDs: <list or "none">

RULES:
- Read evidence files before evaluating any check. Do not guess at contents.
- NEVER invent findings. Every FAIL or WARNING must cite the specific file
  and field (e.g. "ec2_security_groups.json — GroupId sg-abc has inbound
  0.0.0.0/0 on port 22").
- If a file contains {"_error":"denied_or_not_enabled"}, mark all checks
  depending on that file as SKIPPED with reason "access denied or service
  not enabled".
- If a check covers a service in SKIPPED SERVICES, mark it SKIPPED:
  "excluded by --skip-service".
- If a check ID is in SKIPPED CHECK IDs, mark it SKIPPED:
  "excluded by --skip-check".
- Status definitions — apply strictly:
    PASS    — evidence clearly satisfies the best practice
    FAIL    — evidence clearly violates the best practice
    WARNING — partially met, ambiguous, or human review needed
    SKIPPED — service excluded, access denied, or check ID excluded
- Do not mark a check PASS based on absence of evidence. Ambiguous = WARNING.

OUTPUT — emit exactly one JSON block, nothing else:

{
  "pillar": "<PILLAR_CODE>",
  "checks": [
    {
      "id": "<WAF-BP-ID>",
      "question": "<WAF question short title>",
      "best_practice": "<best practice title>",
      "status": "PASS|FAIL|WARNING|SKIPPED",
      "evidence": "<specific finding — file + field cited>",
      "recommendation": "<remediation step if FAIL or WARNING, else null>",
      "skip_reason": "<reason if SKIPPED, else null>"
    }
  ],
  "summary": {
    "pass": 0,
    "fail": 0,
    "warning": 0,
    "skipped": 0
  }
}

PILLAR CHECK INSTRUCTIONS:
<insert full contents of wa-pillars-checks/<PILLAR>.md here>
```

---

## Step 3 — Report Generation

After all subagents return results, collate and write both output files.

**Compute totals:**
- Roll up PASS/FAIL/WARNING/SKIPPED counts per pillar and overall
- Pass rate = PASS / (PASS + FAIL + WARNING) × 100 (SKIPPEDs excluded from denominator)
- Order findings: FAIL first, then WARNING, then PASS, then SKIPPED
- Within each status group, order by pillar (SEC → REL → COST → PERF)

### WAR-REPORT.json

Write to `<output-dir>/WAR-REPORT.json`:

```json
{
  "report_metadata": {
    "generated_at": "<ISO8601 timestamp>",
    "account_id": "<from sts get-caller-identity>",
    "identity_arn": "<from sts get-caller-identity>",
    "regions_checked": ["<region1>"],
    "pillars_checked": ["SEC", "REL", "COST", "PERF"],
    "skipped_services": [],
    "skipped_checks": [],
    "tool_version": "well-architected-review/1.0"
  },
  "summary": {
    "total_checks": 0,
    "pass": 0,
    "fail": 0,
    "warning": 0,
    "skipped": 0,
    "pass_rate_pct": 0.0
  },
  "pillar_summaries": [
    {
      "pillar": "SEC",
      "label": "Security",
      "pass": 0, "fail": 0, "warning": 0, "skipped": 0
    }
  ],
  "findings": [
    {
      "id": "<WAF-BP-ID>",
      "pillar": "<PILLAR_CODE>",
      "question": "<WAF question short title>",
      "best_practice": "<best practice title>",
      "status": "PASS|FAIL|WARNING|SKIPPED",
      "evidence": "<specific finding or evidence cited>",
      "recommendation": "<remediation step or null>",
      "skip_reason": "<reason or null>"
    }
  ]
}
```

### WAR-REPORT.md

Write to `<output-dir>/WAR-REPORT.md`:

```markdown
# AWS Well-Architected Review Report

**Account:** <account_id>
**Identity:** <identity_arn>
**Regions Checked:** <regions>
**Pillars Checked:** Security · Reliability · Cost Optimization · Performance Efficiency
**Generated:** <timestamp>

---

## Executive Summary

| Pillar | PASS | FAIL | WARNING | SKIPPED | Score |
|---|---|---|---|---|---|
| 🔐 Security | N | N | N | N | N% |
| ⚡ Reliability | N | N | N | N | N% |
| 💰 Cost Optimization | N | N | N | N | N% |
| 📈 Performance Efficiency | N | N | N | N | N% |
| **TOTAL** | **N** | **N** | **N** | **N** | **N%** |

> Score = PASS / (PASS + FAIL + WARNING) × 100. SKIPPED checks excluded from score.

---

## ❌ Failed Checks (Action Required)

### [<id>] <best_practice>
**Pillar:** <pillar> | **Question:** <question>
**Evidence:** <evidence>
**Recommendation:** <recommendation>

---

## ⚠️ Warnings (Review Recommended)

[Same format as Failed Checks above]

---

## ✅ Passed Checks

| ID | Pillar | Best Practice | Evidence Summary |
|---|---|---|---|

---

## ⏭️ Skipped Checks

| ID | Pillar | Best Practice | Skip Reason |
|---|---|---|---|

---

## Appendix: Not Automatable

The following WAF areas require human interview and are outside the scope
of this automated review:

- **OPS pillar (all):** Runbooks, on-call practices, game day exercises,
  team responsibilities, culture
- **SEC 9 (Incident Response):** IR plan maturity, tabletop exercises,
  forensics capability
- **REL 3–5 (Workload Architecture):** Service dependency analysis,
  chaos engineering, error budget definition
- **PERF 5 (Process & Culture):** Benchmarking cadence, load testing practice
- **SUS pillar (all):** Sustainability governance, carbon footprint review
- **COST 1 partial:** Finance/engineering collaboration culture

A full Well-Architected Review with an AWS Solutions Architect is recommended
to cover these areas alongside this automated evidence report.
```

---

## Step 4 — Hand Back

After both files are written, print to the user:

1. **Score card** — compact table (pillar | PASS | FAIL | WARNING | SKIPPED | score %)
2. **Top 5 FAIL findings** — one line each: `[<id>] <best_practice> → <recommendation>`
3. **Top 3 WARNING findings** — one line each
4. **File paths** for `WAR-REPORT.md` and `WAR-REPORT.json`
5. **Next steps:**
   - Address FAIL items first (ordered by priority in the report)
   - Schedule a full WAR engagement with an AWS SA to cover unautomated pillars
   - Re-run after remediation to track improvement

---

## Constraints

- **Never run mutating AWS CLI commands.** Only `describe`, `list`, `get`, `query`
  verbs are permitted. If any instruction could be misread as mutating, skip it
  and log a warning.
- **Never store credentials in output files.** Reports must never contain AWS secret
  access keys, session tokens, or account-specific values beyond Account ID.
- **Graceful degradation on every CLI error.** A `{"_error":"..."}` response from
  any collection command must never stop the skill. Log it and mark dependent
  checks as SKIPPED.
- **S3 bucket cap.** If the account has more than 50 S3 buckets, process the first
  50 and note in the report: "S3 checks based on a sample of 50/N buckets."
- **No fabricated evidence.** Subagents must never invent findings. Every FAIL or
  WARNING requires a cited file and field from the raw evidence.
- **No PASS from absence.** If evidence is missing or ambiguous, the result is
  WARNING or SKIPPED — never PASS.
- **`--skip-service` and `--skip-check` are global.** They override pillar check
  file instructions for any matching check.
- **Writes only to `<output-dir>/`.** The skill never writes to the user's project
  files, IaC templates, or any path outside the designated output directory.
