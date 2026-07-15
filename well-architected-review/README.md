# well-architected-review — Automated AWS Well-Architected Review

A Claude Code skill that automates evidence collection and analysis for the AWS
Well-Architected Framework (WAF). It runs exclusively read-only AWS CLI commands
against a live account, maps findings to WAF best practice IDs across four pillars,
and produces a structured report with PASS / FAIL / WARNING / SKIPPED verdicts and
actionable remediation guidance. No cloud resources are created, modified, or deleted.

**Skill:** `/well-architected-review`

**Pillars covered:** Security (SEC) · Reliability (REL) · Cost Optimization (COST) ·
Performance Efficiency (PERF)

**WAF checks implemented:** 30 SEC · 19 REL · 12 COST · 9 PERF — 70 total

**IAM prerequisites:** AWS managed policies `SecurityAudit` + `ReadOnlyAccess`
(or an equivalent read-only custom policy)

---

## Contents

- [Philosophy](#philosophy)
- [Benefits](#benefits)
- [Standards and checks covered](#standards-and-checks-covered)
- [Installation](#installation)
- [Usage guide](#usage-guide)
- [Output file reference](#output-file-reference)
- [FAQ](#faq)

---

## Philosophy

The AWS Well-Architected Framework is designed to be assessed through a combination
of human interviews and infrastructure inspection. This toolkit addresses only the
infrastructure inspection half — the checks where an AWS CLI read can produce a clear
verdict without needing to understand organisational processes, team structure, or
deployment intentions.

**Evidence before analysis.** All AWS CLI calls run first, storing raw JSON output
to disk before any interpretation begins. Analysis subagents then read from that
stable snapshot rather than issuing their own API calls. This separation has two
practical effects: if a subagent fails partway through, the raw evidence is preserved
and can be re-analysed; and it prevents the same API call from being issued multiple
times by different parts of the pipeline.

**Pillar analysis in parallel, not serial.** A monolithic single-pass review that
evaluates all 70 checks in one context window degrades at scale — early checks
consume context that later checks need. This toolkit fans out to one subagent per
pillar, each holding only the check definitions and raw evidence files relevant to
its domain. Concurrency is configurable: `--agents 1` gives sequential execution at
minimum token cost; `--agents 4` runs all four pillars simultaneously.

**Check logic lives outside the orchestrator.** Each pillar's evaluation rules are
defined in a dedicated reference file (`wa-pillars-checks/SEC.md`, `REL.md`, etc.)
rather than embedded in the main `SKILL.md`. The orchestrator never loads these files
into its own context — it passes them to subagents at fan-out time only. This means
adding a new check to the Security pillar requires editing only `SEC.md`; the
orchestration logic is untouched.

**Honest about scope.** WAF best practices that cannot be evaluated from infrastructure
state alone — organisational culture, runbook maturity, game day exercises, incident
response readiness — are listed explicitly in the report's Appendix as outside the scope
of automated review. The report tells the reader exactly what was and was not checked,
so it can be paired correctly with a human-led WAR engagement.

**No fabricated verdicts.** Subagents are instructed never to mark a check PASS based
on absence of evidence. Ambiguous or missing evidence produces WARNING or SKIPPED —
not a false PASS that would give a misleading score.

---

## Benefits

What a DevSecOps engineer can realistically expect from this toolkit:

**Rapid baseline.** Running the skill against an account gives a structured evidence
report in minutes rather than the days a manual WAR preparation typically takes.
Teams preparing for a formal AWS WAR engagement can use the output to identify and
close the most obvious gaps before the review begins.

**Consistent findings.** The same 70 checks run the same way every time. Findings
are tied to specific AWS CLI evidence fields — not to Claude's general knowledge of
what "good" looks like — which makes them auditable and reproducible.

**Actionable remediation guidance.** Every FAIL and WARNING finding includes a
specific recommendation grounded in the evidence that triggered it. The report orders
findings by severity (FAIL first, then WARNING) within each pillar, so the most
critical items are visible at the top.

**Repeatable over time.** Running the skill before and after a remediation effort
produces comparable reports. Because findings are keyed to stable WAF best practice
IDs (e.g. `SEC04-BP01`, `REL09-BP02`), engineers can track which specific checks
moved from FAIL to PASS.

**Honest scope boundary.** The report's Appendix lists every WAF area that was not
automatically assessed and explains why. Engineers know exactly what they still need
to cover in a human interview, rather than assuming the automated report is complete.

**Limitations to be aware of:**

- IAM policy semantic analysis is shallow. The skill flags customer-managed policies
  with broadly named identifiers but does not fetch and parse every policy document's
  Action/Resource fields. Deep IAM analysis requires human review or a dedicated tool.
- S3 per-bucket checks are capped at 50 buckets. Accounts with more than 50 buckets
  receive a representative sample, noted in the report.
- DynamoDB PITR checks are capped at 20 tables per region to avoid excessive API calls.
- The Operational Excellence and Sustainability pillars are not covered. Both require
  organisational context that cannot be inferred from infrastructure state.
- The skill assesses current state only. It cannot determine whether a control was
  in place previously, when it was removed, or why.

---

## Standards and checks covered

The skill implements checks derived from the
[AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
(2024 revision). WAF best practice IDs are used verbatim so findings can be
cross-referenced with AWS documentation and the AWS Well-Architected Tool directly.

### Security pillar — 30 checks

| WAF Question Area | Checks |
|---|---|
| SEC 1 — Security Foundations | SEC01-BP01, SEC01-BP02, SEC01-BP03, SEC01-BP04, SEC01-BP06 |
| SEC 2 — Authentication | SEC02-BP01, SEC02-BP02, SEC02-BP04, SEC02-BP05 |
| SEC 3 — Permissions | SEC03-BP01, SEC03-BP02, SEC03-BP04, SEC03-BP07 |
| SEC 4 — Detection | SEC04-BP01, SEC04-BP02, SEC04-BP02b, SEC04-BP03 |
| SEC 5 — Network Protection | SEC05-BP01, SEC05-BP02, SEC05-BP03, SEC05-BP04 |
| SEC 6 — Compute Protection | SEC06-BP01, SEC06-BP02, SEC06-BP03 |
| SEC 7 — Data Protection | SEC07-BP01, SEC07-BP02, SEC07-BP03, SEC07-BP04, SEC07-BP05, SEC07-BP07 |

**Excluded from Security:** SEC 8 (Application Security — requires code review
beyond infrastructure state) and SEC 9 (Incident Response process maturity —
requires human interview).

### Reliability pillar — 19 checks

| WAF Question Area | Checks |
|---|---|
| REL 1 — Service Quotas | REL01-BP01 |
| REL 2 — Network Topology | REL02-BP01, REL02-BP03, REL02-BP05 |
| REL 6 — Monitoring | REL06-BP01, REL06-BP02, REL06-BP03 |
| REL 7 — Scaling | REL07-BP01, REL07-BP02 |
| REL 8 — Deployments | REL08-BP01, REL08-BP02 |
| REL 9 — Backups | REL09-BP01, REL09-BP02, REL09-BP03, REL09-BP04, REL09-BP05 |
| REL 10 — Disaster Recovery | REL10-BP01, REL10-BP04, REL10-BP05 |

**Excluded from Reliability:** REL 3–5 (Workload Architecture — service dependency
analysis, loose coupling patterns, and error budget definitions cannot be determined
from infrastructure state alone and require architectural review).

### Cost Optimization pillar — 12 checks

| WAF Question Area | Checks |
|---|---|
| COST 1 — Financial Management | COST01-BP01, COST01-BP03 |
| COST 2 — Usage Awareness | COST02-BP01, COST02-BP03 |
| COST 3 — Resource Efficiency | COST03-BP01, COST03-BP02, COST03-BP03, COST03-BP04, COST03-BP05, COST03-BP06 |
| COST 4 — Demand Management | COST04-BP01 |
| COST 5 — Optimize Over Time | COST05-BP01 |

**Excluded from Cost Optimization:** Finance/engineering collaboration culture and
internal chargeback/showback processes (COST 1 partial) — these require organisational
interview. Cost anomaly detection configuration depth is limited to billing alarm
existence and cannot assess the sophistication of anomaly detection rules.

### Performance Efficiency pillar — 9 checks

| WAF Question Area | Checks |
|---|---|
| PERF 1 — Architecture Selection | PERF01-BP01 |
| PERF 2 — Compute and Hardware | PERF02-BP01, PERF02-BP02, PERF02-BP03 |
| PERF 3 — Data Management | PERF03-BP01, PERF03-BP02 |
| PERF 4 — Networking and Content Delivery | PERF04-BP01, PERF04-BP02, PERF04-BP03 |

**Excluded from Performance Efficiency:** PERF 5 (Process and Culture — benchmarking
cadence and load testing practice require human interview and cannot be inferred from
infrastructure state).

### Excluded pillars

**Operational Excellence (OPS):** All OPS checks are process and culture based
(runbooks, on-call practice, game day exercises, team responsibilities). None can
be evaluated from read-only infrastructure state.

**Sustainability (SUS):** Sustainability governance and carbon footprint review
require organisational context beyond what CLI commands can surface.

---

## Installation

### Prerequisites

- **Claude Code** installed and available in your `PATH`.
- **AWS CLI v2** installed and configured. Verify with:
  ```bash
  aws --version
  ```
- An AWS IAM identity (user or role) with both of the following AWS managed
  policies attached, or an equivalent read-only custom policy:
  - `arn:aws:iam::aws:policy/SecurityAudit`
  - `arn:aws:iam::aws:policy/ReadOnlyAccess`
- If using named profiles, confirm the profile is configured in `~/.aws/config`.

### Steps

1. Copy the `.claude/` directory from this toolkit folder into the root of your
   project (or into any directory where you intend to run Claude Code):
   ```bash
   cp -r well-architected-review/.claude/ /path/to/your-project/.claude/
   ```

2. Verify the skill and its reference files are in place:
   ```
   your-project/
   └── .claude/
       └── skills/
           └── WELL_ARCHITECTED_REVIEW/
               ├── SKILL.md
               └── wa-pillars-checks/
                   ├── SEC.md
                   ├── REL.md
                   ├── COST.md
                   └── PERF.md
   ```

3. Open Claude Code in your project directory:
   ```bash
   cd /path/to/your-project
   claude
   ```

4. Verify your AWS credentials before running:
   ```bash
   aws sts get-caller-identity
   # or, with a named profile:
   aws --profile my-audit-profile sts get-caller-identity
   ```

---

## Usage guide

### Command

```
/well-architected-review --regions <r1,r2,...> [options]
```

### Arguments

| Argument | Required | Description |
|---|---|---|
| `--regions <list>` | Yes | Comma-separated AWS region codes to check. For global services (IAM, S3, Organizations, CloudFront, Shield), only the first region is used as the API endpoint. Example: `us-east-1,eu-west-1,ap-southeast-1` |
| `--profile <name>` | No | AWS CLI named profile to use. Omit to use ambient environment credentials. |
| `--pillars <list>` | No | Comma-separated pillar codes to run: `sec`, `rel`, `cost`, `perf`. Defaults to all four. |
| `--skip-service <list>` | No | Comma-separated service tokens to exclude entirely. Valid tokens: `rds`, `elasticache`, `dynamodb`, `ec2`, `s3`, `iam`, `cloudtrail`, `guardduty`, `securityhub`, `waf`, `shield`, `inspector`, `lambda`, `ecr`, `cloudfront`, `autoscaling`, `backup`, `codepipeline`, `config`, `cloudwatch`, `ssm` |
| `--skip-check <list>` | No | Comma-separated WAF best practice IDs to skip. Example: `SEC02-BP05,REL09-BP02` |
| `--agents <n>` | No | Number of parallel subagents for pillar analysis. Default: `3`. Use `1` for sequential execution (cheapest). Max: `6`. |
| `--output-dir <path>` | No | Directory for output files. Defaults to `./war-output/`. |

### What the skill does

**Step 0 — Pre-flight.** Validates arguments, confirms AWS credentials using
`sts get-caller-identity`, prints the account ID and identity ARN for the user to
confirm, and creates the output directory.

**Step 1 — Evidence collection.** Runs approximately 35 global CLI commands and
approximately 30 per-region CLI commands for each region specified. All output is
saved as raw JSON under `<output-dir>/raw/`. No analysis occurs in this phase.

**Step 2 — Pillar analysis.** Spawns up to `--agents` subagents in parallel, each
assigned one pillar. Each subagent reads its pillar check file and the relevant raw
evidence files, then evaluates every check and returns a structured JSON result.

**Step 3 — Report generation.** Collates subagent results, computes per-pillar and
overall scores, and writes both output files.

**Step 4 — Hand back.** Prints a score table, the top FAIL and WARNING findings, and
file paths to the terminal.

### Example invocations

**Full review, two regions, named profile:**
```
/well-architected-review --regions us-east-1,eu-west-1 --profile audit-readonly
```

**Security pillar only, sequential (minimum token usage):**
```
/well-architected-review --regions us-east-1 --pillars sec --agents 1
```

**Full review, skip RDS and ElastiCache (no relational/cache layer in this workload):**
```
/well-architected-review --regions us-east-1,us-west-2 \
  --profile audit-readonly \
  --skip-service rds,elasticache
```

**Skip specific checks by BP ID (e.g. Shield Advanced — intentionally not subscribed):**
```
/well-architected-review --regions eu-west-1 \
  --profile audit-readonly \
  --skip-check SEC05-BP04,COST05-BP01
```

**All four regions, maximum parallelism, custom output directory:**
```
/well-architected-review \
  --regions us-east-1,us-west-2,eu-west-1,ap-southeast-1 \
  --profile audit-readonly \
  --agents 4 \
  --output-dir ./war-results-2025-07
```

---

## Output file reference

All output is written to `<output-dir>/` (default: `./war-output/`). The skill
never writes outside this directory.

### `WAR-REPORT.md`

**Produced by:** Step 3 (Report generation)

A human-readable Markdown report structured as follows:

- **Executive Summary table** — per-pillar counts of PASS, FAIL, WARNING, SKIPPED,
  and a percentage score (PASS / (PASS + FAIL + WARNING) × 100; SKIPPED excluded
  from the denominator).
- **Failed Checks section** — one block per FAIL finding, ordered SEC → REL → COST
  → PERF. Each block contains: check ID, best practice title, WAF question reference,
  the specific CLI evidence that triggered the finding, and a remediation step.
- **Warnings section** — same format as Failed Checks.
- **Passed Checks table** — ID, pillar, best practice title, evidence summary.
- **Skipped Checks table** — ID, pillar, best practice title, skip reason.
- **Appendix: Not Automatable** — lists every WAF area excluded from automated
  review with the reason for exclusion.

### `WAR-REPORT.json`

**Produced by:** Step 3 (Report generation)

A machine-readable JSON file intended for downstream tooling, ticketing system
integration, or trend tracking across runs. Contains:

- `report_metadata` — account ID, identity ARN, regions checked, pillars checked,
  skipped services, skipped checks, generation timestamp, tool version.
- `summary` — total check count, PASS/FAIL/WARNING/SKIPPED counts, overall pass
  rate percentage.
- `pillar_summaries` — same counts broken down per pillar.
- `findings` — array of all check results ordered FAIL → WARNING → PASS → SKIPPED,
  then by pillar within each group. Each finding includes: WAF best practice ID,
  pillar code, WAF question short title, best practice title, status, evidence
  (specific file and field cited), recommendation (null if PASS or SKIPPED), and
  skip reason (null unless SKIPPED).

### `raw/` directory

**Produced by:** Step 1 (Evidence collection)

Contains all raw AWS CLI JSON output. Global service output is stored directly
under `raw/` (e.g. `raw/iam_users.json`, `raw/s3_buckets.json`). Per-region output
is stored under `raw/<region>/` (e.g. `raw/us-east-1/ec2_instances.json`).
Per-bucket S3 checks are stored as `raw/s3_bucket_<bucket-name>.json`.

The `raw/` directory is retained after the skill completes. It can be used for
manual inspection, re-analysis, or debugging a finding. It is safe to delete once
the report has been reviewed.

---

## FAQ

**Is this a replacement for a formal AWS Well-Architected Review?**

No. A formal WAR engagement with an AWS Solutions Architect covers all six pillars,
including the Operational Excellence and Sustainability pillars that this toolkit
excludes entirely, and the process-maturity checks within the four pillars it does
cover. This toolkit is best used in two ways: as preparation material to close the
most obvious gaps before a formal engagement, and as a recurring automated baseline
between formal reviews.

**Do I need to run all four pillars at once?**

No. Use `--pillars sec` to run only the Security pillar, for example. Pillar results
are independent. Running a single pillar is common when remediating findings from a
previous run and wanting to confirm the specific pillar improved.

**What AWS services does the skill check?**

The skill issues read-only calls to: IAM, STS, AWS Organizations, AWS SSO/Identity
Center, IAM Access Analyzer, S3, S3 Control, EC2, VPC, ELB (v2), Auto Scaling, RDS,
ElastiCache, DynamoDB, Lambda, CloudTrail, GuardDuty, Security Hub, AWS Config,
CloudWatch, CloudWatch Logs, SNS, KMS, WAFv2, Shield, Inspector v2, ECR, SSM,
AWS Backup, CodePipeline, CloudFormation, Service Quotas, X-Ray, CloudFront,
AWS Budgets, Cost Explorer, Savings Plans.

**Can I exclude services that are not used in my workload?**

Yes. Use `--skip-service` with a comma-separated list of service tokens. For example,
if your workload uses no RDS or ElastiCache, pass `--skip-service rds,elasticache`
and all checks that depend on those services will be marked SKIPPED with the reason
"excluded by --skip-service". This avoids false WARNING findings for services
that are correctly absent from your architecture.

**Can I suppress specific checks?**

Yes. Use `--skip-check` with a comma-separated list of WAF best practice IDs
(e.g. `--skip-check SEC05-BP04`). This is useful for checks that are intentionally
not implemented — for example, if your organisation has decided not to subscribe to
Shield Advanced, passing `--skip-check SEC05-BP04` prevents the check from
appearing as a WARNING in every report.

**Does the skill modify anything in my AWS account?**

No. Every AWS CLI call issued by the skill uses a read-only verb: `describe`, `list`,
`get`, or `query`. The skill cannot create, modify, or delete any AWS resource.
The only writes the skill performs are to the local `<output-dir>/` directory on
the machine running Claude Code.

**Does the skill modify any files in my project?**

No. The only files written are the output report files and raw evidence files under
`<output-dir>/`. The skill never writes to any path outside that directory.

**What happens if a CLI call is denied due to insufficient permissions?**

The call returns an error, which the skill catches and stores as
`{"_error":"denied_or_not_enabled"}` in the raw evidence file. Any check that
depends on that evidence is marked SKIPPED with the reason "access denied or service
not enabled". The skill continues running — a single denied call does not abort the
review.

**The skill is slow with many regions. How can I speed it up?**

Use `--agents 4` to run all four pillars simultaneously during the analysis phase.
Note that the evidence collection phase (Step 1) is always sequential per-region —
this is intentional to avoid triggering API rate limits. For very large accounts,
consider limiting `--regions` to the regions where your primary workloads run.

**Why is the Operational Excellence pillar not covered?**

Every OPS pillar check concerns organisational process: how runbooks are maintained,
whether game days are conducted, how on-call responsibilities are assigned, and
whether teams have defined error budgets. None of these can be determined by
inspecting AWS resource configurations. Including them would require either
fabricating verdicts or returning SKIPPED for every check — neither is useful.
A human-led WAR engagement is the right tool for the OPS pillar.

**How do I update the checks when AWS updates the Well-Architected Framework?**

Each pillar's checks live in a single Markdown file (`wa-pillars-checks/SEC.md`,
etc.). To add, modify, or remove a check, edit the relevant pillar file. The
orchestration logic in `SKILL.md` does not need to change — it passes the pillar
file contents to the analysis subagent unchanged. Each pillar file begins with a
source and version comment block; update the version note when revising the checks.