---
name: iac-reassess
description: >-
  Stage 4 of the IaC Security Review pipeline. Challenges and validates findings
  from Stage 3 (/iac-audit). Re-reads the cited template lines to confirm each
  finding is real, reassesses CVSSv3.1 severity scores with full environmental
  context, and rejects findings with no practical exploitation path or genuine
  security impact. Produces CIS_FINDINGS_REVISITED.md, TM_FINDINGS_REVISITED.md,
  and/or OVERALL_FINDINGS_REVISITED.md in iac-review-results/. Use after /iac-audit
  completes, or when asked to "validate findings", "reassess severity", or
  "remove false positives from the IaC audit results".
context: fork
argument-hint: "<target-dir> [--cis-only] [--tm-only] [--overall-only]"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Task
  - Bash(grep:*)
  - Bash(head:*)
  - Bash(cat:*)
  - Bash(find:*)
---

# /iac-reassess

Stage 4 of the IaC Security Review pipeline. Validates Stage 3 findings through
static re-verification and principled severity reassessment. The goal is to
produce a final, high-confidence finding set that a security team can act on
without needing to manually triage false positives.

**What this stage does:**
1. Re-reads cited template lines to confirm each finding's evidence is accurate.
2. Checks for compensating controls elsewhere in the template set.
3. Challenges the exploitation path — is this finding actionable or theoretical?
4. Reassesses CVSSv3.1 scores with full environmental context.
5. Rejects findings that fail the exploitation test or have no real impact.
6. Produces revised findings files, clearly marking status for each finding.

**What this stage does NOT do:**
- It does not re-run a full audit (that is Stage 3's job).
- It does not make network calls to live infrastructure.
- It does not modify templates.

---

## Arguments

- `<target-dir>` (required) — project root. Must contain Stage 3 output in `iac-review-results/`.
- `--cis-only` — reassess `CIS_BASED_FINDINGS.md` only.
- `--tm-only` — reassess `THREAT_MODEL_BASED_FINDINGS.md` only.
- `--overall-only` — reassess `OVERALL_FINDINGS.md` only (use after merge).
- Default: reassess all available findings files.

---

## Step 0 — Safety preamble

This skill performs **static re-verification only**. It reads IaC templates and
the findings files produced by Stage 3. It does not execute, deploy, or probe
any infrastructure.

---

## Step 1 — Load context

1. Read `<results-dir>/CIS_BASED_FINDINGS.md` (if present and not excluded by flags).
2. Read `<results-dir>/THREAT_MODEL_BASED_FINDINGS.md` (if present).
3. Read `<results-dir>/OVERALL_FINDINGS.md` (if present).
4. Read `<results-dir>/PROJECT_MAP.md` for environment context and file list.
5. Read `<results-dir>/THREAT_MODEL.md` if present — section 4 `Mitigated` entries
   and section 3 trust boundaries are key inputs for Step 3.
6. Read `<target-dir>/.iac-review-config.yml` for environment context.

Tally: N-CIS findings, N-TM findings, N-Overall findings loaded for reassessment.

---

## Step 2 — Fan out reassessment sub-agents

Unless total findings < 10, spawn **one sub-agent per finding** in parallel,
capped at 10 concurrent. Each sub-agent receives one finding block and access
to the relevant template file(s).

For very small finding sets (< 10), run sequentially without sub-agents.

### Reassessment brief (per finding)

```
You are independently reassessing one security finding from an IaC audit.
Your job is to: (1) verify the evidence, (2) test the exploitation path,
(3) check for compensating controls, and (4) produce a final severity verdict.

FINDING:
{paste the full finding block}

TARGET FILE: {file_path}
You may Read this file and Grep within it.

ENVIRONMENT CONTEXT:
  Tier: {tier}
  Internet-facing: {true/false}
  Data classification: {data_classification}
  Compliance frameworks: {list}

STEP 1 — VERIFY EVIDENCE
Re-read the cited file at the cited line(s). Does the code actually contain
the pattern described? Answer: CONFIRMED | INCORRECT | PARTIALLY CORRECT
  - CONFIRMED: Code matches description exactly.
  - INCORRECT: Code does not match; evidence is wrong. → REJECT this finding.
  - PARTIALLY CORRECT: Issue exists but description overstates or understates it.
    Correct the description before proceeding.

STEP 2 — CHECK COMPENSATING CONTROLS
Search the same file and any related files (referenced modules, SGs attached
to the same VPC, policies attached to the same role) for controls that
mitigate this specific risk. Examples:
  - An S3 bucket with a broad policy but Block Public Access enabled → mitigated
  - A Security Group open on port 22 but the instance is in a private subnet → reduced risk
  - An unencrypted EBS volume that is never attached to any instance → informational only
  - A wildcard IAM policy scoped by an explicit permission boundary → partially mitigated
Record compensating controls found: NONE | PARTIAL | FULL

STEP 3 — TEST THE EXPLOITATION PATH
Apply Claude's judgment (guided by general principles) to challenge this finding:
Ask — "Is there a realistic, practical path by which an attacker could exploit
this issue to cause meaningful impact?" Consider:
  - Does exploitation require physical access, console access, or permissions
    the attacker is unlikely to have?
  - Is the vulnerable resource actually deployed into an environment where the
    attack surface is reachable?
  - Does the finding describe a violation of best practice with no concrete
    exploit story (pure informational / defense-in-depth gap)?
  - Is the impact limited to a non-sensitive resource with no upstream blast radius?

Score the exploitation path:
  EXPLOITABLE    — clear, practical exploit path; real impact
  CONDITIONAL    — exploitable only under specific conditions; note them
  THEORETICAL    — no practical exploit path; issue is real but impact negligible
  NOT_EXPLOITABLE — finding is incorrect or fully mitigated

STEP 4 — SEVERITY REASSESSMENT
Starting from the Stage 3 CVSSv3.1 base score, apply adjustments:

Environmental adjustments (already applied in Stage 3 — verify they are correct):
  production + internet_facing = most severe interpretation of AV and scope
  development + no internet = reduce AV, consider scope

Compensating control adjustments:
  FULL compensation: reduce score by 2.0 (but not below Informational)
  PARTIAL compensation: reduce score by 0.5–1.0
  NONE: no reduction

Exploitation path adjustment:
  EXPLOITABLE: no change to score
  CONDITIONAL: reduce by 0.5; note condition in description
  THEORETICAL: reduce by 1.5; downgrade to Informational if base < 4.0
  NOT_EXPLOITABLE: REJECT

Final severity: Critical(9-10) | High(7-8.9) | Medium(4-6.9) | Low(0.1-3.9) | Informational(0)

STEP 5 — VERDICT

Output exactly one block:

<verdict>
<id>{original finding ID}</id>
<status>CONFIRMED | REVISED | REJECTED | INFORMATIONAL</status>
<evidence_check>{CONFIRMED | INCORRECT | PARTIALLY_CORRECT}</evidence_check>
<compensating_controls>{NONE | PARTIAL | FULL} — {brief description or "none found"}</compensating_controls>
<exploitation_path>{EXPLOITABLE | CONDITIONAL | THEORETICAL | NOT_EXPLOITABLE}</exploitation_path>
<exploitation_notes>{specific conditions or why path is limited}</exploitation_notes>
<original_severity>{Critical|High|Medium|Low}</original_severity>
<original_score>{X.X}</original_score>
<revised_severity>{Critical|High|Medium|Low|Informational}</revised_severity>
<revised_score>{X.X}</revised_score>
<revised_cvss_vector>{CVSS:3.1/...}</revised_cvss_vector>
<score_change_rationale>{one sentence explaining why score changed, or "No change"}</score_change_rationale>
<revised_description>{corrected or confirmed description; include conditions if CONDITIONAL}</revised_description>
<recommendation>{confirmed or refined remediation}</recommendation>
</verdict>

REJECTION criteria (output REJECTED if ANY of these apply):
  - Evidence check: INCORRECT (code does not match the finding description)
  - Compensating controls: FULL (issue is completely mitigated elsewhere)
  - Exploitation path: NOT_EXPLOITABLE
  - The finding is a duplicate of another finding on the same resource with the
    same root cause (keep the higher-scoring one)
  - The finding is purely a best-practice gap with no concrete impact in this
    specific deployment context

Do NOT reject because:
  - The finding is unlikely to be exploited (THEORETICAL is still kept as Informational)
  - The remediation is difficult or costly
  - The resource is in a dev environment (adjust score, don't reject)
```

---

## Step 3 — Collate verdicts

Collect all `<verdict>` blocks. Tally:

| Status | Count |
|--------|-------|
| CONFIRMED (no change) | N |
| REVISED (severity changed) | N |
| INFORMATIONAL (downgraded) | N |
| REJECTED | N |

For REJECTED findings: record them in a dedicated section of the output file
with the rejection reason — they should not simply disappear (audit trail).

For REVISED findings: note the original vs. revised score and the delta.

Re-sort all non-rejected findings by revised score (desc).
Reassign stable IDs: `CIS-R-001`, `TM-R-001`, or `F-R-001` for merged.

---

## Step 4 — Write revisited findings files

Write to `<results-dir>/`:

### Structure for each output file

```markdown
# [CIS|TM|Overall] Findings — Revisited
Target: <target-dir>
Stage 3 source: <source file(s)>
Reassessed: <ISO8601>
Assessor: /iac-reassess

## Reassessment summary

| Original total | Confirmed | Revised | Informational | Rejected |
|---------------|-----------|---------|---------------|---------|
| N             | N         | N       | N             | N       |

Severity distribution (after reassessment):
| Severity      | Before | After |
|---------------|--------|-------|
| Critical      | N      | N     |
| High          | N      | N     |
| Medium        | N      | N     |
| Low           | N      | N     |
| Informational | —      | N     |

## Validated findings

### [F-R-001] <title> ✅ CONFIRMED | ⚠️ REVISED | ℹ️ INFORMATIONAL

- **Status:** CONFIRMED | REVISED | INFORMATIONAL
- **Severity:** <Revised> (<Original> originally)
- **Score:** <Revised X.X> (Base X.X → Adjusted X.X) 
- **CVSS Vector:** <CVSS:3.1/...>
- **CIS / Threat ID:** <ID>
- **File:** `<path>` line <N>
- **Resource:** `<type>.<name>`
- **Evidence:**
  ```
  <cited template lines>
  ```
- **Exploitation path:** EXPLOITABLE | CONDITIONAL | THEORETICAL
- **Conditions / Notes:** <if CONDITIONAL, describe required conditions>
- **Compensating controls:** <NONE | what was found>
- **Score change rationale:** <why score changed, or "No change">
- **Description:** <revised or confirmed description>
- **Recommendation:** <confirmed or refined remediation>

---

## Rejected findings

<These were identified in Stage 3 but do not represent actionable security issues
in this specific deployment context.>

### [REJECTED] <original ID> — <title>

- **Original severity:** <severity and score>
- **Rejection reason:** <INCORRECT_EVIDENCE | FULLY_MITIGATED | NOT_EXPLOITABLE | DUPLICATE>
- **Detail:** <one-sentence explanation>

---

## Recommendations prioritized

Final ordered action list based on revised severity:

| Priority | Finding ID | Severity | Title | Effort |
|----------|-----------|----------|-------|--------|
| 1        | F-R-001   | Critical | ...   | Low    |
```

Effort estimate: Low = config change in 1 file | Medium = multiple files or module change | High = architectural change.

---

## Step 5 — Write OVERALL_FINDINGS_REVISITED.md

If reassessing both CIS and TM paths, produce a merged final file after writing
the individual revisited files. This is the final deliverable for the review:

```markdown
# Overall Findings — Final Assessment
<merge of CIS_FINDINGS_REVISITED.md + TM_FINDINGS_REVISITED.md>
<deduped on same resource + same property + same root cause>
<highest severity kept for duplicates>
<findings from both paths noted with detected_by: both|cis|threat-model>
```

---

## Step 6 — Hand back

Tell the user:

1. Findings reviewed: N total (N CIS, N TM, N merged as applicable).
2. Final status breakdown: Confirmed / Revised / Informational / Rejected.
3. Severity shift summary: how many went up / stayed same / went down.
4. Top 5 validated findings by revised severity (ID, severity, one-line title).
5. Rejected findings: N — reasons summary.
6. Files written: list all paths.
7. Review complete. Suggest next actions:
   - Share `OVERALL_FINDINGS_REVISITED.md` with remediation team.
   - Track findings in your issue tracker; use the Recommendation field for tickets.
   - Re-run `/iac-map` + `/iac-audit` + `/iac-reassess` after remediation to validate fixes.

---

## Constraints

- **Never reject a finding solely because it is unlikely to be exploited.** Downgrade
  to Informational instead. Rejection is reserved for evidence failures, full mitigations,
  and proven non-exploitability.
- **Audit trail is mandatory.** Rejected findings must appear in the Rejected section
  with reasons. Do not silently drop findings.
- **Do not add new findings.** Stage 4 validates Stage 3 output — it does not conduct
  new discovery. If a new issue is noticed during re-reading, add a note at the bottom
  of the file under "Observations for follow-up" but do not assign it a finding ID.
- **Do not modify template files.**
- **Score floor.** No finding's revised score may be raised above 10.0 or lowered
  below 0.0. Informational findings score exactly 0.0.
