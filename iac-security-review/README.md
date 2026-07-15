# IaC Security Review — Claude Code Skill Suite

A four-stage pipeline for comprehensive security review of Infrastructure as Code (IaC) templates using Claude Code. Covers Terraform (`.tf`, `.tfvars`) and AWS CloudFormation (`.yaml`, `.json`, `.template`) with threat modeling, CIS benchmark assessment, and automated finding validation.

> **Current scope:** AWS provider (v1). Designed for extension to GCP and Azure.

---

## Contents

- [Overview](#overview)
- [Why a staged pipeline?](#why-a-staged-pipeline)
- [Standards and checks](#standards-and-checks)
- [Pipeline flow](#pipeline-flow)
- [Installation](#installation)
- [Usage guide](#usage-guide)
- [FAQ](#faq)

---

## Overview

This suite gives DevSecOps engineers a structured, token-efficient way to audit IaC templates — without relying on a single monolithic prompt that reads everything, hallucinates patterns, and returns a wall of unranked findings.

The pipeline consists of four Claude Code skills, each invoked as a slash command:

| Stage | Command | Purpose | Output |
|-------|---------|---------|--------|
| 1 | `/iac-map` | Discover and map all IaC files; extract resource inventory; flag hot spots | `PROJECT_MAP.md` |
| 2 | `/iac-threat-model` | Build a STRIDE threat model via interview or autonomous analysis | `THREAT_MODEL.md`, `KNOWN_ISSUES.md` |
| 3 | `/iac-audit` | Audit templates against threat model and/or CIS benchmark controls | `CIS_BASED_FINDINGS.md`, `TM_BASED_FINDINGS.md`, `OVERALL_FINDINGS.md` |
| 4 | `/iac-reassess` | Validate findings, reject false positives, reassess severity scores | `*_REVISITED.md` |

All outputs land in a single `iac-review-results/` directory in your project root.

---

## Why a staged pipeline?

Most LLM-based security tools fall into the same trap: they read every file, run a generic "find security issues" prompt, and return a long list of findings with no prioritisation and a high false-positive rate. This pipeline is designed differently, following the same principle used in professional red-team engagements: **understand before you test, and validate before you report.**

### The problem with "scan everything at once"

When a single prompt reads dozens of template files simultaneously several things go wrong:

**Context saturation.** Large Terraform repos can have hundreds of `.tf` files. Loading them all at once pushes earlier files toward the edges of the context window where recall degrades. The model is effectively reading the last file it loaded with full attention and everything before it with partial attention. Critical findings in large modules get missed.

**Generic findings.** Without knowing what the infrastructure is for, who uses it, and what data it handles, every finding gets the same generic severity. An open security group in a development VPC with no sensitive data scores identically to the same misconfiguration in a PCI-scoped production environment. This creates noise that erodes trust in the tool.

**No exploitation validation.** A static pattern match on `cidr_blocks = ["0.0.0.0/0"]` generates a finding regardless of whether that security group is attached to a public-facing instance, a private Lambda function, or a test resource that never gets deployed. Without a second pass to challenge the finding in context, the remediation list becomes a guessing game.

**Token waste.** Reading vendor directories, test fixtures, and generated Terraform cache folders wastes significant tokens on content that will never contain real issues.

### How staging fixes each problem

**Stage 1 solves context saturation and token waste.** The mapper uses `grep` and `glob` before reading anything, builds a file inventory in a single lightweight pass, and produces a structured map that all later stages use as their index. Downstream stages read only the files they need, in the order they need them. Test directories, `.terraform` caches, and vendor paths are excluded before a single byte of template content is loaded.

**Stage 2 solves generic findings.** The threat model transforms infrastructure context — whether gathered through a five-minute interview or derived autonomously from the templates — into a ranked threat table. Stage 3 then checks for those specific threats in those specific templates, not every possible pattern against every possible file. A threat-model-guided audit on a 200-file repo reads perhaps 20 files deeply rather than skimming 200 files shallowly.

**Stage 4 solves false positives.** Every finding from Stage 3 goes through a four-step challenge: verify the evidence is real, check for compensating controls elsewhere in the template set, test whether a realistic exploitation path exists, and reassess the CVSS score with environmental context. Findings that fail the challenge are moved to a clearly labelled rejected section with reasons — not silently dropped, but not cluttering the action list either.

**The result is a prioritised, validated finding set** where every item in the final `OVERALL_FINDINGS_REVISITED.md` has been confirmed in the template source, has no compensating control that neutralises it, and has a realistic exploitation story. A security engineer can work through that list top-to-bottom without first spending an afternoon triaging false positives.

---

## Standards and checks

### CIS AWS Foundations Benchmark v4.0.1 (December 2024)

The CIS benchmark reference files were created based on the official CIS guidelines and contain only controls that can be reviewed in a scope of Automated assessment type and checkable directly via IaC template analysis (no live AWS API calls required). Controls that require runtime state — credential age, active key counts, MFA enrollment — are excluded.

**29 IaC-checkable controls across 5 domains:**

| Domain | Controls | Key checks |
|--------|----------|-----------|
| **Section 1 — Identity and Access Management** | 8 | No root access keys, password policy length and reuse, no wildcard `*:*` admin policies, IAM instance roles, IAM Access Analyzer enabled, support role present, permissions via groups only |
| **Section 2 — Storage** | 6 | S3 HTTPS-only bucket policy, S3 Block Public Access (all 4 settings), RDS encryption at rest, RDS not publicly accessible, RDS auto minor version upgrade, EFS encryption |
| **Section 3 — Logging** | 7 | CloudTrail multi-region with global events, log file validation, AWS Config enabled, S3 access logging on CloudTrail bucket, CloudTrail KMS encryption, KMS key rotation, VPC Flow Logs on all VPCs |
| **Section 4 — Monitoring** | 2 | Unauthorized API call alarms (CloudWatch metric filters), Security Hub enabled |
| **Section 5 — Networking** | 6 | EBS encryption by default, NACLs not open to `0.0.0.0/0` on admin ports, Security Groups not open to `0.0.0.0/0` or `::/0` on admin ports, default Security Group restricts all traffic, IMDSv2 enforced |

Each control entry in the benchmark files includes the CIS level (L1/L2), a description, the rationale, and concrete Terraform and CloudFormation code examples showing both passing and failing configurations.

### Threat catalogs

Stage 2 uses four built-in threat reference files, covering 104 specific threat entries across three layers:

| Catalog | Entries | Scope |
|---------|---------|-------|
| `aws-threat-catalog.md` | 47 | AWS service-level threats (IAM, S3, EC2, networking, compute, data, supply chain) mapped to STRIDE |
| `terraform-threats.md` | 26 | Terraform framework-specific risks: state security, provider pinning, credential handling, IAM misconfig patterns, network defaults |
| `cfn-threats.md` | 31 | CloudFormation-specific risks: parameter handling, stack policies, nested template security, deletion/update policies, encryption gaps |
| `iac-owasp.md` | 8 categories | OWASP IaC Security Cheatsheet distilled: insecure defaults, hardcoded secrets, access control, missing encryption, logging gaps, outdated components, network config, pipeline supply chain |

### CVSSv3.1 scoring with environmental modifiers

All findings are scored using CVSSv3.1. Stage 3 produces base scores; Stage 4 applies environmental modifiers from your `.iac-review-config.yml`:

- **Production tier** → +0.5 adjustment
- **Internet-facing** → +0.5 on network-vector findings
- **Restricted data classification** → +0.5 on confidentiality/integrity impact
- **Compensating controls found** → −0.5 to −2.0 depending on strength
- **Theoretical exploitation path** → −1.5, minimum Informational

---

## Pipeline flow

### Full pipeline (recommended)

```
Your project
     │
     ▼
┌─────────────────────────────────────────────────┐
│  Stage 1: /iac-map                              │
│                                                 │
│  • Glob/Grep discovery (never reads in full)    │
│  • Classifies Terraform modules vs. roots       │
│  • Identifies CloudFormation templates          │
│  • Flags hot spots (hardcoded secrets,          │
│    public exposure, IAM wildcards)              │
│  • Builds resource type inventory               │
│  • Applies exclusions from config               │
└──────────────────┬──────────────────────────────┘
                   │ PROJECT_MAP.md
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 2: /iac-threat-model          [OPTIONAL] │
│                                                 │
│  Interview mode: 4-question STRIDE framework    │
│  Autonomous mode: catalog-based auto-generation │
│                                                 │
│  • Ingests existing scanner reports             │
│    (Checkov, tfsec, Prowler) → KNOWN_ISSUES.md  │
│  • Produces ranked threat table (L×I scored)   │
└──────────────────┬──────────────────────────────┘
                   │ THREAT_MODEL.md
                   │ KNOWN_ISSUES.md (optional)
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 3: /iac-audit                            │
│                                                 │
│  Path A — Threat-model-based:                   │
│    One sub-agent per template file (parallel)   │
│    Checks for threats from THREAT_MODEL.md      │
│    → TM_BASED_FINDINGS.md                       │
│                                                 │
│  Path B — CIS benchmark-based:                  │
│    One sub-agent per template file (parallel)   │
│    Checks 29 IaC-checkable CIS controls         │
│    → CIS_BASED_FINDINGS.md                      │
│                                                 │
│  Both paths merge, deduplicate, score with      │
│  CVSSv3.1 + environmental modifiers             │
│    → OVERALL_FINDINGS.md                        │
└──────────────────┬──────────────────────────────┘
                   │ Findings files
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 4: /iac-reassess                         │
│                                                 │
│  Per-finding validation (parallel sub-agents):  │
│    1. Verify evidence in template source        │
│    2. Check for compensating controls           │
│    3. Test exploitation path realism            │
│    4. Reassess CVSSv3.1 with full env context   │
│                                                 │
│  Status per finding:                            │
│    ✅ CONFIRMED  — validated as-is              │
│    ⚠️  REVISED   — severity adjusted            │
│    ℹ️  INFORMATIONAL — real but theoretical     │
│    ❌ REJECTED   — false positive (with reason) │
│                                                 │
│    → OVERALL_FINDINGS_REVISITED.md (final)      │
└─────────────────────────────────────────────────┘
```

### Alternate: CIS-only fast path (no threat model)

Use this when you need a quick baseline check and do not have time for a threat modeling session.

```
/iac-map .
/iac-audit . --cis-only --merge
/iac-reassess . --overall-only
```

### Alternate: Threat-model-only path

Use this when you already have CIS compliance covered by another tool (e.g. Prowler running in CI) and want architecture-specific threat hunting.

```
/iac-map .
/iac-threat-model . --autonomous    # or omit --autonomous for interview
/iac-audit . --threat-model-only
/iac-reassess . --tm-only
```

### Iterative path (recommended for large teams)

Run the autonomous threat model first to generate a draft, then refine it in an interview session before the audit.

```
/iac-map .
/iac-threat-model . --autonomous                           # Generate draft
/iac-threat-model . --seed iac-review-results/THREAT_MODEL.md  # Refine with owner
/iac-audit . --merge
/iac-reassess . --overall-only
```

---

## Installation

### Prerequisites

- Claude Code (latest version with `Task` sub-agent support)
- Project containing Terraform (`.tf`) or CloudFormation (`.yaml`/`.json`) files

### Setup

**1. Copy the `.claude/` directory into your project:**

```bash
# From this repository root:
cp -r iac-security-review/.claude/ /path/to/your-project/.claude/
```

This installs the four skills and the config template. The folder structure will be:

```
your-project/
└── .claude/
    ├── iac-review-config.yml          ← edit this
    └── skills/
        ├── IAC_MAP/SKILL.md
        ├── IAC_THREAT_MODEL/
        │   ├── SKILL.md
        │   └── threats-list/
        │       ├── aws-threat-catalog.md
        │       ├── terraform-threats.md
        │       ├── cfn-threats.md
        │       └── iac-owasp.md
        ├── IAC_AUDIT/
        │   ├── SKILL.md
        │   └── cis-benchmarks/
        │       ├── cis-section1-iam.md
        │       ├── cis-section2-storage.md
        │       ├── cis-section3-logging.md
        │       └── cis-section4-5-monitoring-networking.md
        └── IAC_REASSESS/SKILL.md
```

**2. Edit the config file:**

Open `.claude/iac-review-config.yml` and fill in your project's values:

```yaml
project_name: "my-aws-platform"
provider: aws
iac_type: terraform           # terraform | cloudformation | both

environment:
  tier: "production"          # production | staging | development
  internet_facing: true
  data_classification: "confidential"
  compliance_frameworks:
    - "SOC2"
    - "PCI-DSS"

# Paste existing scanner output here to include in the threat model:
known_issues_file: "reports/checkov-latest.json"
```

**3. Add exclusion paths for your project's structure:**

The default config already excludes `.terraform/`, `vendor/`, `test*/`, `examples/`, etc. Add any project-specific paths you want skipped:

```yaml
exclude_paths:
  - "infrastructure/legacy/**"   # not in scope for this review
  - "modules/third-party/**"     # vendor modules, not our code
```

**4. Commit `.claude/` to version control** so the entire team uses the same skills and config. The `iac-review-results/` directory that gets created during reviews should be added to `.gitignore` or committed selectively — it contains findings that may be sensitive.

---

## Usage guide

### Running the full pipeline

Open Claude Code in your project root and run each command in sequence. Each stage runs in an isolated sub-agent context (`context: fork`) so its verbose internal processing does not fill your main conversation window.

```
/iac-map .
```
Wait for the `PROJECT_MAP.md` summary. Review the hot spots and recommended focus areas. If the file inventory looks wrong (too many files included, or important modules missing), adjust `exclude_paths` in the config and re-run before proceeding.

```
/iac-threat-model .
```
Claude will ask whether someone familiar with the project is available for an interview. If yes, it walks through four questions and builds a threat model from your answers. If not — or if you pass `--autonomous` — it generates the threat model from the templates and built-in catalogs. The interview takes 10–20 minutes and significantly improves Stage 3 signal.

To ingest an existing scanner report at the same time:
```
/iac-threat-model . --known-issues reports/checkov-results.json
```

```
/iac-audit . --merge
```
Both assessment paths run in parallel sub-agents. You will see progress messages as each template file is processed. The `--merge` flag triggers deduplication and produces `OVERALL_FINDINGS.md` after both paths complete. On a large project with 30+ template files this may take a few minutes.

```
/iac-reassess . --overall-only
```
Each finding in `OVERALL_FINDINGS.md` is passed to an independent sub-agent for validation. The final `OVERALL_FINDINGS_REVISITED.md` is your primary deliverable — take this to your remediation team.

---

### Command reference

#### `/iac-map <target-dir> [options]`

| Option | Description |
|--------|-------------|
| `--exclude <glob>` | Add an exclusion path on top of config defaults. Repeatable. |
| `--no-inventory` | Skip per-file resource counting. Faster on very large repos. |

#### `/iac-threat-model <target-dir> [options]`

| Option | Description |
|--------|-------------|
| `--autonomous` | Skip the interview. Generate threat model from templates and catalogs only. |
| `--known-issues <file>` | Path to an existing scanner report (Checkov JSON, tfsec JSON/SARIF, Prowler JSON/CSV, or Markdown). Overrides `known_issues_file` in config. |
| `--seed <file>` | Start the interview from an existing `THREAT_MODEL.md` draft. Useful for iterative refinement. |

#### `/iac-audit <target-dir> [options]`

| Option | Description |
|--------|-------------|
| `--cis-only` | Run CIS benchmark checks only (Path B). |
| `--threat-model-only` | Run threat-model-based checks only (Path A). Requires `THREAT_MODEL.md`. |
| `--merge` | Produce `OVERALL_FINDINGS.md` after both paths complete. Enabled automatically when both paths run. |
| `--no-fan-out` | Disable parallel sub-agents. Run sequentially. Use for very small projects or debugging. |

#### `/iac-reassess <target-dir> [options]`

| Option | Description |
|--------|-------------|
| `--cis-only` | Reassess `CIS_BASED_FINDINGS.md` only. |
| `--tm-only` | Reassess `THREAT_MODEL_BASED_FINDINGS.md` only. |
| `--overall-only` | Reassess `OVERALL_FINDINGS.md` only. Use this after a `--merge` audit run. |

---

### Output files reference

All files are written to `iac-review-results/` in your project root.

| File | Stage | Description |
|------|-------|-------------|
| `PROJECT_MAP.md` | 1 | File inventory, resource types, hot spots, focus area recommendations |
| `THREAT_MODEL.md` | 2 | STRIDE threat table with likelihood × impact scores, assets, entry points, recommended mitigations |
| `KNOWN_ISSUES.md` | 2 | Transformed scanner report: Title, CVSS (approximate), Description, Root Cause |
| `CIS_BASED_FINDINGS.md` | 3 | CIS benchmark violations with CVSSv3.1 base + adjusted scores |
| `TM_BASED_FINDINGS.md` | 3 | Threat-model-guided findings with threat ID references |
| `OVERALL_FINDINGS.md` | 3 | Merged and deduplicated findings from both audit paths |
| `CIS_FINDINGS_REVISITED.md` | 4 | Validated CIS findings with exploitation assessment |
| `TM_FINDINGS_REVISITED.md` | 4 | Validated TM findings with exploitation assessment |
| `OVERALL_FINDINGS_REVISITED.md` | 4 | **Final deliverable.** Prioritised, validated findings ready for remediation |

---

## FAQ

**Do I have to run all four stages every time?**

No. Stage 1 (`/iac-map`) is always required because all later stages depend on `PROJECT_MAP.md`. Stage 2 is optional — skipping it means Stage 3 runs CIS-only, which is still valuable. Stages 3 and 4 are designed to run together, but you can run Stage 3 alone if you want the raw findings without validation.

The minimum viable run is:
```
/iac-map .
/iac-audit . --cis-only
```

---

**Why does Stage 2 ask me questions instead of just reading the templates?**

Templates tell you what resources exist, but they do not tell you what data those resources handle, who the threat actors are, or what compensating controls exist outside the IaC (WAF rules, GuardDuty detections, network appliances). A threat model built from an interview produces findings that reflect actual business risk rather than generic pattern matches. The 10–20 minutes spent in the interview typically saves hours of false-positive triage later.

If you genuinely cannot do an interview, `--autonomous` generates a reasonable threat model from the templates alone. It will be less accurate for business-logic threats but perfectly adequate for infrastructure-level issues.

---

**What formats does Stage 2 accept for existing scanner reports?**

The `--known-issues` option (and `known_issues_file` in the config) accepts:
- **Checkov JSON** — from `checkov -d . -o json`
- **tfsec JSON** — from `tfsec . --format json`
- **tfsec SARIF** — from `tfsec . --format sarif`
- **Prowler JSON** — from `prowler aws -M json`
- **Prowler CSV** — from `prowler aws -M csv`
- **Markdown or plain text** — free-form notes, previous audit reports

The skill auto-detects the format and normalises all findings into a consistent structure (Title, CVSS score, Description, Root Cause) stored in `KNOWN_ISSUES.md`.

---

**The audit spawned many sub-agents. Is that normal?**

Yes. Stage 3 spawns one sub-agent per template file, capped at 10 concurrent. This fan-out is intentional: it allows all files to be reviewed in parallel rather than sequentially, which reduces total wall-clock time significantly on large repos. Stage 4 spawns one sub-agent per finding for independent validation.

If you need to disable parallelism (for debugging, or on very small projects), pass `--no-fan-out` to Stage 3. Stage 4 automatically runs sequentially when there are fewer than 10 findings.

---

**How does the suite handle large monorepos with hundreds of template files?**

Stage 1 caps the resource inventory at 200 files (samples the largest 50 plus 150 random) and notes the cap. The hot-spot Grep passes run across all files regardless of the cap.

Stage 3 processes files in batches of 10 concurrent sub-agents. A repo with 80 template files runs in 8 sequential batches. Each sub-agent reads only its assigned file deeply, so the total token cost scales with the number of unique resource types across the repo rather than its raw file count.

For very large repos, consider running Stage 3 twice with `--cis-only` and `--threat-model-only` in separate sessions, then manually combining the results, rather than running both paths at once.

---

**Why are some CIS controls missing from the benchmark files?**

Three categories of controls were deliberately excluded:

1. **Manual assessment controls** — controls that require human review of account settings or processes (e.g. 1.1 security contact details, 2.1.3 data classification) cannot be checked by reading templates.

2. **Runtime-state controls** — controls that require live AWS API calls to evaluate (e.g. 1.12 credential age, 1.13 active access key count, 1.14 key rotation date) are not assessable from static IaC. These require a runtime tool like Prowler or the AWS console.

3. **CloudWatch metric filter controls** (Section 4, controls 4.2–4.15) — these are Manual assessment type in the CIS document. Their IaC patterns are included as supplementary reference in the monitoring/networking benchmark file, but they are not included in the automated check pass.

If you need coverage of the excluded controls, run Prowler or AWS Security Hub alongside this suite — their outputs can be fed into Stage 2 via `--known-issues`.

---

**Will this tool modify my Terraform or CloudFormation files?**

Never. All four skills are strictly read-only with respect to your infrastructure templates. The only files written are the output Markdown files in `iac-review-results/`. This is enforced at the skill level via `allowed-tools` — write access is scoped to the results directory only.

---

**How do I update the CIS benchmark when a new version is released?**

The CIS benchmark controls are stored as plain Markdown files in `.claude/skills/IAC_AUDIT/cis-benchmarks/`. To update:

1. Download the new CIS AWS Foundations Benchmark PDF.
2. Identify which controls changed, were added, or were removed.
3. Edit the relevant section file (`cis-section1-iam.md`, etc.) following the existing format: Level, Assessment type, IaC-checkable check, Terraform pattern (pass), CFN pattern (pass).
4. Add new controls; remove deprecated ones.
5. Commit the updated files — all team members pick up the changes automatically on next use.

---

**How do I add custom security checks?**

You have two options:

**Option A — Extend the threat catalogs.** Add entries to `threats-list/aws-threat-catalog.md` (or `terraform-threats.md`, `cfn-threats.md`) following the existing table format. Stage 2 reads these files at runtime, so new threats are automatically incorporated into future threat models and Stage 3 audits.

**Option B — Add a custom CIS-style check file.** Create a new file in `.claude/skills/IAC_AUDIT/cis-benchmarks/` (e.g. `custom-org-checks.md`) and include your organisation's internal security requirements in the same format as the CIS files. Then reference it from the audit skill by adding a note in `IAC_AUDIT/SKILL.md` pointing to the custom file.

---

**Can I use this in a CI/CD pipeline?**

The skills are designed for interactive Claude Code sessions, not headless CI execution. For CI, consider running a traditional scanner (Checkov, tfsec, Trivy) on every pull request and feeding those results into this pipeline's Stage 2 (`--known-issues`) for deeper human-review cycles.

A practical pattern is: **automated scanner in CI for every PR + this pipeline for quarterly reviews and major architecture changes**.

---

**Can this replace a penetration test or manual security review?**

No. This suite performs static analysis of IaC templates only. It does not:

- Test running infrastructure (no network probes, no API calls to your AWS account)
- Detect runtime misconfigurations not present in the IaC templates
- Find application-layer vulnerabilities in code deployed by the infrastructure
- Identify misconfigurations introduced by manual changes to the AWS console after deployment (IaC drift)

Think of it as a highly capable static linter with security expertise — it significantly reduces the effort required for a human security reviewer and catches a large class of IaC-level issues, but it is a complement to, not a replacement for, penetration testing and runtime security tooling.