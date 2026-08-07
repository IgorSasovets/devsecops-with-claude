# Available Resources

A directory of all toolkits, skills, and commands available in this repository.
Each entry links to the toolkit's own `README.md` for full installation and usage
instructions. Entries are listed alphabetically by folder name.

---

## Toolkits

### `iac-security-review` — IaC Template Security Review Pipeline

A four-stage Claude Code pipeline for comprehensive security review of Terraform
and AWS CloudFormation templates. Covers project mapping, STRIDE threat modeling,
CIS AWS Foundations Benchmark v4.0.1 assessment, and finding validation with
CVSSv3.1 scoring.

**Skills:** `/iac-map` · `/iac-threat-model` · `/iac-audit` · `/iac-reassess`

**Standards covered:** CIS AWS Foundations Benchmark v4.0.1 (29 IaC-checkable
automated controls), STRIDE threat modeling, OWASP IaC Security Cheatsheet,
CVSSv3.1 with environmental scoring.

**Supports:** Terraform · AWS CloudFormation · Checkov · tfsec · Prowler

→ IaC Template Security Review Pipeline. See [`iac-security-review/README.md`](iac-security-review/README.md)

---

### `well-architected-review` — Automated AWS Well-Architected Review

A single-skill Claude Code toolkit that automates evidence collection and analysis
for four pillars of the AWS Well-Architected Framework using exclusively read-only
AWS CLI commands. It runs approximately 60 CLI calls per region, fans out parallel
subagent analysis per pillar, and produces a structured PASS / FAIL / WARNING /
SKIPPED report mapped to WAF best practice IDs with actionable remediation guidance.
No cloud resources are created, modified, or deleted.

**Skills:** `/well-architected-review`

**Standards covered:** AWS Well-Architected Framework (2024 revision) — 70 best
practice checks across Security (30 checks: SEC01–SEC07), Reliability (19 checks:
REL01–REL02, REL06–REL10), Cost Optimization (12 checks: COST01–COST05), and
Performance Efficiency (9 checks: PERF01–PERF04).

**Requires:** AWS managed policies `SecurityAudit` + `ReadOnlyAccess` (read-only;
no write permissions needed or used).

→ Automated AWS Well-Architected Review. See [`well-architected-review/README.md`](well-architected-review/README.md)

---

### `opengrep-rules-creator` — Opengrep Rules Creator Pipeline

A two-stage Claude Code pipeline that analyses application source code and
generates production-ready [Opengrep](https://github.com/opengrep/opengrep)
SAST rules tailored to the project's actual frameworks, entry points,
sanitizers, and vulnerability history. Stage 1 maps the codebase — tech stack,
data flows, dangerous sinks, validators, business logic, and dependencies —
and produces a structured recon report. Stage 2 reads that report and creates
one `.yaml` rule file per finding, with full metadata, test cases, and
`opengrep validate` syntax checking. Rules are generated one at a time with
operator approval, or in batch via `--autonomous` mode. A `tailored` mode
creates a single precise rule for a specific CVE or library method on demand.

**Skills:** `/opengrep-code-recon` · `/opengrep-rule-creator`

**Standards covered:** OWASP Top 10 Web (2021) — A01, A02, A03, A05, A06,
A07, A08, A09; OWASP API Security Top 10 (2023) — API1–API5, API8; OWASP
Mobile Top 10 (2024) — M1–M5, M9; CWE mappings on every rule; CVSSv3.1
scoring with complete vector on every rule.

**Supports:** JavaScript · TypeScript · Python · Java · Kotlin · Go · PHP ·
Swift · Dart · Opengrep CLI (optional — used for `opengrep validate` syntax
checking; falls back to static review when absent).

→ Opengrep Rules Creator Pipeline. See [`opengrep-rules-creator/README.md`](opengrep-rules-creator/README.md)

---

### `npm-audit-analysis` — npm Dependency Vulnerability Analysis

A single-skill Claude Code toolkit that ingests an `npm audit` JSON report (or
generates one by running `npm audit --json` itself), traces each vulnerable
dependency to actual usage in your source code, assesses real-world exploitability,
and produces a structured Markdown + JSON report. Findings are ranked by confirmed
exploitability rather than raw CVSS score. Per-finding temp files are written
progressively so analysis survives token-limit interruptions and can be resumed
across sessions. The skill never modifies project files or applies fixes — it is
read-only analysis only.

**Skills:** `/npm-audit-analysis`

**Standards covered:** CVSSv3.1 (scores read from npm advisory data), CWE
(identifiers extracted per finding), npm advisory database (all advisories
surfaced by `npm audit --json`), OWASP Top 10 A06:2021 — Vulnerable and Outdated
Components.

**Supports:** Any Node.js / npm project · npm v7+ · JavaScript · TypeScript ·
pre-generated audit JSON or live `npm audit` run · configurable skip lists for
dev-only dependencies and named packages.

→ npm Dependency Vulnerability Analysis. See [`npm-audit-analysis/README.md`](npm-audit-analysis/README.md)

---

*To add a new toolkit to this list, follow the contribution guide in [`README.md`](README.md)
and add an entry here in the same format, keeping entries in alphabetical order by folder name.*