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

*To add a new toolkit to this list, follow the contribution guide in [`README.md`](README.md)
and add an entry here in the same format, keeping entries in alphabetical order by folder name.*