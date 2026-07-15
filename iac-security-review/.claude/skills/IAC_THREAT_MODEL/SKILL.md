---
name: iac-threat-model
description: >-
  Stage 2 of the IaC Security Review pipeline. Optional but recommended.
  Builds a STRIDE-based threat model for the target infrastructure by either
  (a) interviewing the user using an adaptive four-question framework seeded
  from PROJECT_MAP.md, or (b) autonomously generating a threat model from
  templates and built-in threat catalogs when the user cannot provide context.
  Ingests and transforms existing scanner reports into KNOWN_ISSUES.md.
  Writes THREAT_MODEL.md and optionally KNOWN_ISSUES.md to iac-review-results/.
  Output feeds /iac-audit for threat-model-based assessment path.
  Use when asked to "threat model", "build a threat model for IaC", or
  "what threats apply to this infrastructure".
context: fork
argument-hint: "<target-dir> [--autonomous] [--known-issues <file>] [--seed <THREAT_MODEL.md>]"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Bash(find:*)
  - Bash(ls:*)
  - Bash(head:*)
  - Bash(grep:*)
  - Bash(cat:*)
  - AskUserQuestion
  - Task
---

# /iac-threat-model

Stage 2 of the IaC Security Review pipeline. Produces `THREAT_MODEL.md` using
the STRIDE methodology, seeded from the project's IaC templates and context
gathered from the user or built-in threat catalogs.

**This stage is optional.** `/iac-audit` can run without it, falling back to
CIS-based assessment. When available, the threat model makes Stage 3 significantly
more targeted and relevant.

---

## Arguments

- `<target-dir>` (required) — project root. Same directory used with `/iac-map`.
- `--autonomous` — skip the interview; generate threat model from templates and
  catalogs only. Use when no one familiar with the project is available.
- `--known-issues <file>` — path to an existing scanner report to ingest.
  Overrides `known_issues_file` in `.iac-review-config.yml`.
- `--seed <THREAT_MODEL.md>` — start the interview from an existing draft
  threat model (e.g. produced by `--autonomous` in an earlier run).

---

## Step 0 — Safety preamble

This skill performs **static analysis only**. It reads IaC templates and config
files. It does not execute, deploy, or probe any infrastructure.

Confirm and state:
1. Target directory exists and is a local checkout you can read.
2. You will not execute any code or make network requests to live infrastructure.

---

## Step 1 — Load context

1. Read `<target-dir>/.iac-review-config.yml` if present. Extract:
   - `environment` block (tier, internet_facing, data_classification, compliance_frameworks)
   - `known_issues_file` path
   - `output.results_dir`
2. Read `<target-dir>/iac-review-results/PROJECT_MAP.md` if present.
   Extract: resource type inventory, security hot spots, recommended focus areas.
   If absent, note it and proceed (skip references to project map data).
3. Read the four threat catalog files from this skill's directory:
   - `threats-list/aws-threat-catalog.md`
   - `threats-list/terraform-threats.md`
   - `threats-list/cfn-threats.md`
   - `threats-list/iac-owasp.md`
   Keep these in working memory — they are the threat vocabulary for this session.

---

## Step 2 — Route to mode

```
--autonomous flag set?
  └─ YES → go to AUTONOMOUS MODE (Step 4)
  └─ NO  → is --seed provided?
               └─ YES → read seed file → go to INTERVIEW MODE with draft pre-loaded (Step 3)
               └─ NO  → ask the user:
                          "Is someone familiar with this project's architecture or
                           security requirements available to answer questions in
                           this session?"
                          YES → go to INTERVIEW MODE (Step 3)
                          NO  → go to AUTONOMOUS MODE (Step 4)
```

---

## Step 3 — INTERVIEW MODE

Use an adaptive four-question framework. The four questions are the backbone;
the number of follow-up questions per topic adapts to the complexity of the
answers and the signals from the template scan.

Before starting the interview, do a targeted scan of IaC templates to seed
each question with concrete observations. Use Grep — do not read files in full.

```bash
# Identify top resource types to make Q1 concrete
grep -rh "^resource \"\|^\s*Type: AWS::" <target-dir> | sort | uniq -c | sort -rn | head -20

# Surface specific hot spots to ask about
grep -rn "0\.0\.0\.0/0\|publicly_accessible\s*=\s*true\|PubliclyAccessible\s*:\s*true" <target-dir>
grep -rn "Principal.*\*\|\"Action\".*\*" <target-dir>
```

### Q1 — What are we building?

**Ask about:** System purpose, user types, data flows, integration points,
external dependencies. Tailor based on resource types found in templates.

Example prompts (adapt to actual resources found):
- "What does this system do, and who are its users?"
- "I see [N] RDS instances — what kind of data do they store?"
- "This infrastructure has internet-facing load balancers — what services do they expose?"
- "Are there integrations with external systems (payment processors, third-party APIs)?"
- "What AWS accounts / environments does this deploy into?"

Fill threat model sections: **Assets**, **Entry points**, **Trust boundaries**.

### Q2 — What could go wrong?

**Ask about:** Threat actors relevant to this system, attack scenarios, known
concern areas. Map answers to STRIDE categories and threat catalog entries.

Example prompts:
- "Who are the likely threat actors — opportunistic attackers, targeted nation-state, insider threat, or a combination?"
- "What data or functionality, if compromised, would cause the most business impact?"
- "Are there known abuse patterns or regulatory concerns for this type of system?"
- "I noticed [specific hot spot from scan] — is this intentional? What's the risk context?"

Fill threat model section: **Threat actors**, **STRIDE threat table** (initial rows).

### Q3 — What are we doing about it?

**Ask about:** Existing controls, compensating controls, accepted risks. This
prevents false positives in Stage 3 by recording known mitigations.

Example prompts:
- "Are there WAF rules, SCPs, or GuardDuty detections that compensate for the
  open security group rules I found?"
- "Is encryption handled at the application layer for any resources where IaC
  encryption is disabled?"
- "Are there known accepted risks or documented exceptions?"
- "What compliance frameworks govern this system?"

Fill threat model section: **Controls**, **Accepted risks**, **Open questions**.

### Q4 — How do we know we got it right?

**Ask about:** Coverage gaps, validation mechanisms, residual concerns.

Example prompts:
- "Are there threat categories you're particularly concerned we might miss?"
- "Has this system had a previous security assessment? What were the main findings?"
- "Are there areas of the codebase you consider higher risk that I should focus on?"

This question also surfaces the `known_issues_file` if not already provided:
- "Do you have an existing scanner report (Checkov, tfsec, Prowler) I should
  incorporate as context?"

Fill threat model section: **Open questions for Stage 3**.

### Interview output

After all four questions, synthesize answers into `THREAT_MODEL.md` (Step 6).
If the user provided a known-issues file at any point, proceed to Step 5 before
writing the threat model.

---

## Step 4 — AUTONOMOUS MODE

Generate the threat model by reading templates and applying the threat catalogs.
Fan out with sub-agents for parallel analysis of large projects.

### 4a — Template scan

Read the top-level structure of each IaC file identified in PROJECT_MAP.md
(or discovered via Glob if PROJECT_MAP.md is absent). Use targeted reads:
- Read files under 100 lines in full.
- For larger files, use `head -100` + targeted `grep` passes.

Extract: resource types, IAM policies and trust relationships, network config
(SGs, NACLs, VPC settings), encryption settings, logging config, external
exposure indicators.

### 4b — Threat catalog matching

For each resource type found, match against the threat catalogs:
- Check `aws-threat-catalog.md` for service-specific threats (T-IAM-*, T-S3-*, etc.)
- Check `terraform-threats.md` for Terraform-framework risks (TF-*)
- Check `cfn-threats.md` for CloudFormation-framework risks (CFN-*)
- Check `iac-owasp.md` for cross-cutting categories (OWASP-1 through OWASP-8)

Filter: only include threats where the template contains the relevant resource
type or anti-pattern. Discard inapplicable threats.

### 4c — STRIDE gap-fill

After catalog matching, check each STRIDE category has at least one threat:
- **S**poofing — IAM trust, STS, API keys
- **T**ampering — state, template integrity, data modification
- **R**epudiation — logging, audit trail
- **I**nformation Disclosure — encryption, IAM, network exposure
- **D**enial of Service — availability, resource quotas
- **E**levation of Privilege — IAM escalation, role abuse

Add inferred threats for any uncovered STRIDE category based on resource types
present. Mark inferred threats with `[INFERRED]` in the source column.

---

## Step 5 — Known issues ingestion (if file provided)

Run this step if `--known-issues <file>` is set, config file has `known_issues_file`,
or the user mentioned an existing scanner report during the interview.

### Supported formats

Detect format automatically:
- **Checkov JSON** — look for `"check_id"`, `"check_type"`, `"results"` keys
- **tfsec JSON** — look for `"results"` array with `"rule_id"`, `"severity"` keys
- **tfsec SARIF** — look for `"$schema"` with `sarif`, `"runs"` array
- **Prowler JSON** — look for `"StatusExtended"`, `"CheckID"` keys
- **Prowler CSV** — look for `CHECK-ID,SEVERITY,SERVICE` header
- **Markdown / plain text** — treat as free-form findings; extract by looking for
  severity keywords (HIGH, MEDIUM, LOW, CRITICAL) and resource references

### Transformation rules

For each finding, extract and normalize:

```markdown
| Field | Source field |
|-------|-------------|
| Title | check_id / rule_id / first line of finding |
| CVSS Score | Map severity: CRITICAL→9.0, HIGH→7.5, MEDIUM→5.0, LOW→2.5 (approximate base; no env modifier here) |
| Description | message / description / first sentence of finding text |
| Root Cause | resource / file / short rationale |
| Status | PASS/FAIL/WARN if available |
```

**Deduplication:** Group identical check IDs across multiple files into one entry.
**Limit:** If more than 50 findings, summarize the top 20 by severity and note the total count.

Write the transformed output to `<results-dir>/KNOWN_ISSUES.md`:

```markdown
# Known Issues
Source: <file path and format>
Processed: <ISO8601 timestamp>
Total findings: <N> | Shown: <N>

## Summary
| Severity | Count |
|----------|-------|
| Critical | N |
| High     | N |
| Medium   | N |
| Low      | N |

## Findings

### [KI-001] <Title>
- **CVSS (approximate):** <score>
- **Severity:** <CRITICAL|HIGH|MEDIUM|LOW>
- **Description:** <normalized description>
- **Root Cause:** <resource / IaC pattern causing the issue>
- **Original ID:** <check_id or rule_id>
```

---

## Step 6 — Write THREAT_MODEL.md

Write `<results-dir>/THREAT_MODEL.md` using the schema below.
Read the schema immediately before writing — do not rely on memory of it.

```markdown
# Threat Model
Project: <name from config or directory>
Generated: <ISO8601 timestamp>
Mode: <interview | autonomous | interview-seeded>
Provider: AWS
IaC frameworks: <terraform | cloudformation | both>

## 1. System context
<2-4 sentence description of what the system does, who uses it, and its
deployment environment. From interview answers or autonomous inference.>

## 2. Assets
| Asset | Sensitivity | Description |
|-------|-------------|-------------|
| <e.g. customer PII in RDS> | High | <what it is and why it matters> |

## 3. Entry points and trust boundaries
| Entry Point | Protocol | Trust Level | Notes |
|------------|----------|-------------|-------|

## 4. Threat table (STRIDE)
| ID | STRIDE | Threat | Actor | Surface | Asset | Likelihood (1-5) | Impact (1-5) | L×I | Controls | Status |
|----|--------|--------|-------|---------|-------|-----------------|-------------|-----|----------|--------|

Likelihood scale: 1=Rare, 2=Unlikely, 3=Possible, 4=Likely, 5=Almost certain
Impact scale: 1=Negligible, 2=Minor, 3=Moderate, 4=Significant, 5=Critical
Status: Open | Accepted | Mitigated

## 5. Deprioritized threats
<Threats considered but excluded, with brief rationale>

## 6. Open questions for Stage 3
<Items that could not be resolved from templates or interview — flag these
for the auditor to investigate>

## 7. Known issues
<"See KNOWN_ISSUES.md" if generated, or "None provided">

## 8. Recommended mitigations (top 5)
| Priority | Threat IDs | Recommendation |
|----------|-----------|----------------|

## 9. Compliance notes
<Relevant observations if compliance_frameworks are set in config>
```

---

## Step 7 — Hand back

Tell the user:

1. Mode used and rationale.
2. Threat table: N threats total (S/T/R/I/D/E breakdown).
3. Top 5 threats by L×I score (id, one-line description, score).
4. Known issues: N findings ingested → KNOWN_ISSUES.md (or "not provided").
5. Open questions for Stage 3: list them.
6. Files written: paths to THREAT_MODEL.md and KNOWN_ISSUES.md.
7. Next step: `> /iac-audit <target-dir>` — threat-model-based audit path will
   be selected automatically because THREAT_MODEL.md is now present.

---

## Constraints

- **Do not execute or deploy anything.** Read only.
- **Do not fabricate resource details.** Only reference resources confirmed by
  Grep/Read. Mark inferred threats clearly with `[INFERRED]`.
- **Cap interview at 4 main questions + 3 follow-ups per question.**
  Do not interrogate indefinitely. Unresolved items become open questions.
- **Known issues ingestion is a transform, not a full re-analysis.**
  Reproduce the finding accurately; do not add new CVSS scores from scratch
  without acknowledging the approximation.
