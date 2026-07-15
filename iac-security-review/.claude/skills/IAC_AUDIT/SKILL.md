---
name: iac-audit
description: >-
  Stage 3 of the IaC Security Review pipeline. Performs a security audit of
  IaC templates (Terraform .tf, CloudFormation .yaml/.json) using one or both
  of two paths: (A) threat-model-based — driven by THREAT_MODEL.md from Stage 2,
  (B) CIS-based — driven by CIS AWS Foundations Benchmark v4.0.1 controls.
  Fans out one sub-agent per template file for parallel analysis. Produces
  CIS_BASED_FINDINGS.md, THREAT_MODEL_BASED_FINDINGS.md, and/or OVERALL_FINDINGS.md
  with CVSSv3.1-scored findings in iac-review-results/. Requires PROJECT_MAP.md
  from Stage 1. THREAT_MODEL.md from Stage 2 is optional but strongly recommended.
  Use when asked to "audit IaC", "scan for security issues in Terraform/CloudFormation",
  or as Stage 3 of the IaC security review.
context: fork
argument-hint: "<target-dir> [--cis-only] [--threat-model-only] [--merge] [--no-fan-out]"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Task
  - Bash(find:*)
  - Bash(ls:*)
  - Bash(head:*)
  - Bash(grep:*)
  - Bash(cat:*)
  - Bash(wc:*)
---

# /iac-audit

Stage 3 of the IaC Security Review pipeline. Audits IaC templates for security
vulnerabilities using up to two parallel assessment paths and produces structured
findings files for Stage 4 reassessment.

**Assessment paths:**
- **Path A — Threat-model-based:** Uses threats in THREAT_MODEL.md to guide
  targeted analysis. Produces `THREAT_MODEL_BASED_FINDINGS.md`.
- **Path B — CIS-based:** Checks all IaC-checkable CIS AWS Benchmark v4.0.1
  Automated controls. Produces `CIS_BASED_FINDINGS.md`.
- Both paths run by default when THREAT_MODEL.md is present. Use `--cis-only`
  or `--threat-model-only` to restrict to one path.

**Sub-agent fan-out:** Unless `--no-fan-out`, spawns one sub-agent per template
file in parallel (capped at 10 concurrent). On projects with < 5 files, runs
sequentially.

---

## Arguments

- `<target-dir>` (required) — project root. Must contain `iac-review-results/PROJECT_MAP.md`.
- `--cis-only` — run Path B only (CIS benchmark checks).
- `--threat-model-only` — run Path A only (requires THREAT_MODEL.md).
- `--merge` — after both paths complete, produce `OVERALL_FINDINGS.md` (deduped,
  merged). Automatically enabled when both paths run.
- `--no-fan-out` — disable sub-agent parallelism; run sequentially. Use for
  small projects or debugging.

---

## Step 0 — Safety preamble

This skill performs **static analysis only**. It reads IaC templates. It does
not deploy, execute, or modify any infrastructure or target files.

---

## Step 1 — Load context

1. Read `<target-dir>/iac-review-results/PROJECT_MAP.md`. Extract:
   - Full file list (Terraform + CFN templates)
   - Resource type inventory (for CIS check scoping)
   - Security hot spots (seed findings for follow-up)
   - Environment context (for CVSSv3.1 environmental scoring)
   - `results_dir` path
   If PROJECT_MAP.md is absent, stop with an error: "Run `/iac-map <target-dir>` first."

2. Check for `<results-dir>/THREAT_MODEL.md`. If present:
   - Parse section 4 (threat table): extract all `Open` status threats.
   - Parse section 6 (open questions): note items flagged for auditor.
   - Parse section 8 (recommended mitigations): use as additional check prompts.
   Set `threat_model_available = true`.

3. Load environment context from PROJECT_MAP.md or config:
   - `tier` (production/staging/development)
   - `internet_facing` (true/false)
   - `data_classification`
   These are used in Step 5 for CVSSv3.1 environmental score adjustment.

4. Read `<target-dir>/.iac-review-config.yml` if present for any overrides.

5. Route based on flags and available inputs:
   ```
   --cis-only                   → run Path B only
   --threat-model-only          → run Path A only (error if no THREAT_MODEL.md)
   threat_model_available=true  → run both Path A and Path B
   threat_model_available=false → run Path B only (note absence to user)
   ```

---

## Step 2 — Scope files for review

From PROJECT_MAP.md, build the list of files to audit. Apply the same
exclusions as Stage 1. If `--cis-only`, filter further: only include templates
containing resource types relevant to CIS checks (IAM, S3, RDS, EFS, EC2, VPC,
CloudTrail, Config, KMS, Security Hub, Access Analyzer).

Tell the user:
- N files queued for audit
- Path(s) to run: CIS / Threat-model / Both
- Fan-out: N sub-agents (or sequential)

---

## Step 3 — Path A: Threat-model-based audit

*Skip if `--cis-only` or `threat_model_available = false`.*

For each threat in THREAT_MODEL.md section 4 with status `Open`:

Extract the threat's: STRIDE category, attack surface, asset, and linked
IaC root cause (from the threat catalog entries that seeded it).

Assign it to the most relevant template file(s) based on resource types.
Fan out sub-agents per file.

### Sub-agent brief — Threat-model path

```
You are performing an authorized static security review of IaC templates.
Your task: analyze the template for evidence of specific threats.

TARGET FILE: {file_path}
FRAMEWORK: {terraform | cloudformation}
RESOURCE TYPES PRESENT: {list from PROJECT_MAP}

THREAT MODEL CONTEXT:
{Paste relevant threat rows from THREAT_MODEL.md section 4}

OPEN QUESTIONS FROM THREAT MODEL:
{Paste section 6 items relevant to this file's resource types}

ENVIRONMENT CONTEXT:
  Tier: {tier}
  Internet-facing: {true/false}
  Data classification: {data_classification}

TASK: For each threat listed, check whether the template contains the
vulnerable pattern described. Reason from the code — do NOT execute anything.

For each finding, output exactly one block in this format:

<finding>
<path_id>TM-{file_index:02d}-{finding_index:02d}</path_id>
<file>{relative/path/to/file}</file>
<line>{line number or range}</line>
<threat_id>{threat ID from threat model, e.g. T-IAM-01}</threat_id>
<stride>{S|T|R|I|D|E}</stride>
<resource_type>{AWS resource type, e.g. aws_s3_bucket}</resource_type>
<resource_name>{Terraform resource name or CFN logical ID}</resource_name>
<title>{one-line summary}</title>
<description>{what the vulnerable pattern is; cite specific property names and values}</description>
<evidence>{exact line(s) from the template that demonstrate the issue}</evidence>
<cvss_base>{base score 0.0-10.0 using CVSSv3.1 AV/AC/PR/UI/S/C/I/A}</cvss_base>
<cvss_vector>{CVSS:3.1/AV:?/AC:?/PR:?/UI:?/S:?/C:?/I:?/A:?}</cvss_vector>
<recommendation>{specific remediation — property name and correct value}</recommendation>
<confidence>{0.0-1.0 — how certain you are this is a real issue}</confidence>
</finding>

If the file has NO issues matching the given threats, output a single:
<finding><path_id>TM-{n}-00</path_id><file>{file}</file><result>clean</result></finding>

IMPORTANT:
- Only report what you can directly observe in the template.
- Do not infer issues from file names alone.
- Cite the exact property/attribute that is the problem.
- Do not report on resources or services not present in this file.
```

---

## Step 4 — Path B: CIS-based audit

*Skip if `--threat-model-only`.*

Load CIS benchmark reference files from this skill's `cis-benchmarks/` directory.
Determine which sections are relevant based on resource types in PROJECT_MAP.md:

| Resource type found | CIS sections to load |
|--------------------|---------------------|
| aws_iam_*, AWS::IAM::* | cis-section1-iam.md |
| aws_s3_*, aws_db_*, aws_efs_*, AWS::S3::*, AWS::RDS::*, AWS::EFS::* | cis-section2-storage.md |
| aws_cloudtrail, aws_config_*, aws_flow_log, AWS::CloudTrail::*, AWS::Config::*, AWS::EC2::FlowLog | cis-section3-logging.md |
| aws_cloudwatch_*, aws_securityhub_*, AWS::CloudWatch::*, AWS::SecurityHub::* | cis-section4-5-monitoring-networking.md |
| aws_security_group*, aws_network_acl*, aws_vpc*, aws_instance*, aws_ebs_*, AWS::EC2::* | cis-section4-5-monitoring-networking.md |

Read only the relevant sections — skip the rest to save tokens.

Fan out one sub-agent per template file.

### Sub-agent brief — CIS path

```
You are performing an authorized static security review of IaC templates
against CIS AWS Foundations Benchmark v4.0.1 controls.

TARGET FILE: {file_path}
FRAMEWORK: {terraform | cloudformation}
RESOURCE TYPES PRESENT: {list from PROJECT_MAP}

CIS CONTROLS TO CHECK:
{Paste only the CIS control entries from the benchmark files that are
relevant to the resource types present in this file}

ENVIRONMENT CONTEXT:
  Tier: {tier}
  Internet-facing: {true/false}
  Data classification: {data_classification}

TASK: For each CIS control listed, check whether the template satisfies or
violates it. Use the IaC patterns in the control definition to guide your
analysis. Only report violations (FAIL). Skip controls where the relevant
resource type is not present in this file.

For each violation, output exactly one block:

<finding>
<path_id>CIS-{file_index:02d}-{finding_index:02d}</path_id>
<file>{relative/path/to/file}</file>
<line>{line number or range}</line>
<cis_id>{e.g. 2.2.3}</cis_id>
<cis_level>{L1|L2}</cis_level>
<cis_title>{control title}</cis_title>
<resource_type>{AWS resource type}</resource_type>
<resource_name>{Terraform name or CFN logical ID}</resource_name>
<title>{one-line summary of the violation}</title>
<description>{what is wrong and why it matters}</description>
<evidence>{exact line(s) from the template demonstrating the violation}</evidence>
<cvss_base>{base score 0.0-10.0 using CVSSv3.1}</cvss_base>
<cvss_vector>{CVSS:3.1/AV:?/AC:?/PR:?/UI:?/S:?/C:?/I:?/A:?}</cvss_vector>
<recommendation>{specific fix — property name and correct value}</recommendation>
<confidence>{0.0-1.0}</confidence>
</finding>

IMPORTANT:
- L1 controls are baseline; always report L1 failures.
- L2 controls are advanced; report L2 failures and mark the level.
- Only report controls relevant to resource types actually in this file.
- Cite the exact template line(s) that fail the control.
- A control check that PASSES should NOT appear in output.
```

---

## Step 5 — CVSSv3.1 Environmental Score Adjustment

After collecting all findings from both paths, apply environmental score
adjustments using the context from Step 1.

**Scoring approach:** Start from the sub-agent's base CVSS score and apply
modifiers. Do not inflate or deflate by more than 2.0 points from the base.

| Environmental Factor | Upward modifier | Downward modifier |
|---------------------|----------------|------------------|
| `tier = production` | +0.5 | — |
| `tier = development` | — | -1.0 |
| `internet_facing = true` | +0.5 on AV:N findings | — |
| `internet_facing = false` | — | -0.5 on AV:N findings |
| `data_classification = restricted` | +0.5 on C/I impact | — |
| `data_classification = public` | — | -0.5 on C impact |
| Compensating control noted in THREAT_MODEL.md | — | -0.5 to -1.0 |

**Final severity bands (CVSSv3.1):**
- Critical: 9.0–10.0
- High: 7.0–8.9
- Medium: 4.0–6.9
- Low: 0.1–3.9
- Informational: 0.0

Record both the base score and the adjusted score in findings.

---

## Step 6 — Collate and deduplicate

### Cross-path deduplication (for --merge)

A finding is a duplicate if it references the same `file` + `resource_name` +
same violated property. When a finding appears in both paths:
- Keep the entry with the higher adjusted CVSS score.
- In OVERALL_FINDINGS.md, note it was detected by both paths (add `detected_by: both`).
- Retain both the CIS ID and the threat model ID in the merged entry.

### Intra-path deduplication

Within a single path's results:
- Same `file` + `resource_name` + same property → keep one, note count.
- Different resources, same check → keep both (they are distinct findings).

### Stable ID assignment

After deduplication, assign stable IDs:
- CIS path: `CIS-001`, `CIS-002`, … (sorted by adjusted score desc, then file, then line)
- TM path: `TM-001`, `TM-002`, …
- Merged: `F-001`, `F-002`, … (interleaved, highest score first)

---

## Step 7 — Write findings files

Write to `<results-dir>/`:

### CIS_BASED_FINDINGS.md (Path B output)

```markdown
# CIS-Based Findings
Target: <target-dir>
Generated: <ISO8601>
CIS Benchmark: AWS Foundations v4.0.1
Files audited: N
Controls checked: N

## Summary
| Severity  | Count |
|-----------|-------|
| Critical  | N     |
| High      | N     |
| Medium    | N     |
| Low       | N     |
| Total     | N     |

## Findings

| ID | Severity | CIS | L | File | Resource | Title |
|----|----------|-----|---|------|----------|-------|

### CIS-001 — <title>
- **Severity:** <Critical|High|Medium|Low> (Adjusted: X.X | Base: X.X)
- **CVSS Vector:** <CVSS:3.1/...>
- **CIS Control:** [<id>] <title> (Level <1|2>)
- **File:** `<path>` line <N>
- **Resource:** `<type>.<name>`
- **Description:** <what is wrong>
- **Evidence:**
  ```
  <exact line(s) from template>
  ```
- **Recommendation:** <specific fix>
- **Confidence:** <0.0-1.0>
```

### THREAT_MODEL_BASED_FINDINGS.md (Path A output)

Same structure, replacing CIS fields with:
- **Threat:** `[<threat_id>]` — `<title>` | STRIDE: <category>

### OVERALL_FINDINGS.md (merged output, when --merge or both paths ran)

```markdown
# Overall Findings
<merged, deduped, sorted by adjusted severity>
...
| detected_by | both | cis | threat-model |
```

---

## Step 8 — Hand back

Tell the user:

1. Files audited: N Terraform + M CloudFormation.
2. Path(s) run: CIS / Threat-model / Both.
3. CIS findings: X (Crit/High/Med/Low split). Top 3 by severity.
4. Threat-model findings: X (split). Top 3 by severity. (if applicable)
5. Total unique findings after deduplication: X.
6. Files written: list paths.
7. Next step: `> /iac-reassess <target-dir>` — validate and challenge findings,
   reject false positives, produce final revised severity scores.

---

## Constraints

- **Never execute target code or make network calls to live infrastructure.**
- **Never modify template files.** Read-only.
- **Cap sub-agents at 10 concurrent.** If more than 10 files, batch in groups.
- **Do not fabricate line numbers.** Every cited line must be confirmed by Read or Grep.
- **Confidence threshold:** If a finding's `confidence < 0.4`, include it but
  mark it `[LOW CONFIDENCE]` in the title so Stage 4 can prioritize.
- **Minimum evidence:** Every finding must include at least one quoted line from
  the template as evidence. Findings without evidence will be rejected by Stage 4.
