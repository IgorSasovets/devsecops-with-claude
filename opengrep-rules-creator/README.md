# Opengrep Rules Creator

A two-stage Claude Code pipeline that analyses a project's source code and
produces production-ready [Opengrep](https://github.com/opengrep/opengrep)
rules tailored to that project's actual code patterns, frameworks, and
vulnerability history. The pipeline covers languages across web, backend, and
mobile ecosystems, and outputs one `.yaml` rule file per rule — ready to drop
into any CI pipeline that runs Opengrep.

---

## Contents

- [Philosophy](#philosophy)
- [Benefits](#benefits)
- [Standards and checks covered](#standards-and-checks-covered)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage guide](#usage-guide)
- [Output file reference](#output-file-reference)
- [FAQ](#faq)

---

## Philosophy

Most SAST rule sets are written generically — they match patterns across all
codebases regardless of what libraries a project actually uses, how its
validators are wired, or what its trust boundaries are. The result is a high
false-positive rate that erodes trust in findings and causes engineers to
switch scanners off.

This toolkit takes the opposite approach: **reconnaissance before rule
creation**. Stage 1 maps the project first — entry points, dangerous sinks,
sanitizers, validators, auth gates, business logic, and third-party packages —
before a single rule is written. Stage 2 reads that map and uses it as the
foundation for every pattern decision. Sanitizers found in Stage 1 become
`pattern-sanitizers` in Stage 2. Auth gates found in Stage 1 become taint
guards. Framework-specific input sources found in Stage 1 become `pattern-sources`.
The rules know about this project because they were built from it.

**Three tradeoffs this design makes explicit:**

**Staged over monolithic.** A single prompt that tries to map a codebase and
write rules simultaneously does neither well. Stage 1 produces a compact,
structured `CODE_RECON.md` that Stage 2 reads once — avoiding redundant file
reads and keeping each stage focused on one job.

**Operator-in-the-loop over fully autonomous.** Stage 2 presents one rule at
a time and waits for approval before proceeding. An autonomous mode exists for
operators who want batch output, but the default keeps a human in the decision
loop. This matters for rules that ship to CI: a rule written and approved by a
security engineer is more trustworthy than a batch of rules that was never
reviewed.

**Validation over raw output.** Every generated rule is validated with
`opengrep validate --config` before it is written to the index. A rule that
fails syntax validation is iterated on immediately rather than handed to the
operator as broken output.

---

## Benefits

A DevSecOps engineer using this toolkit should expect:

- **Rules that match your actual codebase.** Sources, sinks, and sanitizers
  are derived from the project under review — not from a generic template.
  This materially reduces false positives compared to off-the-shelf rulesets.

- **Coverage grounded in evidence.** Stage 2 proposes rules based on
  hot spots found in Stage 1, OWASP Top 10 gaps for the detected stack, and
  entries in `KNOWN_ISSUES.md` (your vulnerability history). Every proposed
  rule can be traced back to a signal in the codebase or your history.

- **Full metadata on every rule.** Each rule ships with CWE mapping, OWASP
  category, a complete CVSSv3.1 vector, references, and concrete fix guidance.
  The output is ready for a security team to act on, not just read.

- **Opengrep-specific optimisations.** Rules that require cross-function taint
  tracking are annotated with `--taint-intrafile` guidance. Higher-order
  function patterns and constructor-based taint flows (unique Opengrep
  capabilities) are considered during rule design.

- **Scales to project size.** Stage 1 auto-scales its analysis depth based
  on file count. Projects under 200 source files get a thorough deep pass;
  projects over 2 000 files fall back to a targeted grep-only pass. Token
  spend is proportional to project size, not fixed.

**Honest limitations:**

- The pipeline does **not** verify that rules detect real exploits — it
  validates rule syntax and structure only. Confirming detection accuracy
  against a live target requires running `opengrep scan` separately.
- Cross-file taint tracking (inter-procedural analysis across multiple files)
  is not yet fully supported by Opengrep. Rules requiring this will be noted
  but may have limited effectiveness until the feature matures upstream.
- Stage 1 recon is **not** a substitute for a full threat model. It surfaces
  patterns and hot spots; it does not reason about business context, trust
  boundaries, or deployment architecture. For that, use the
  `IAC_THREAT_MODEL` skill or the `threat-model` skill from Anthropic's
  reference harness.

---

## Standards and checks covered

### OWASP coverage

Stage 2 proposes rules against the following OWASP standards, filtered to
what is applicable to the detected stack:

| Standard | Categories covered |
|---|---|
| OWASP Top 10 Web (2021) | A01 Broken Access Control, A02 Cryptographic Failures, A03 Injection, A05 Security Misconfiguration, A06 Vulnerable Components, A07 Auth Failures, A08 Software Integrity Failures, A09 Logging Failures |
| OWASP API Security Top 10 (2023) | API1 BOLA, API2 Auth, API3 Object Property Exposure, API4 Resource Consumption, API5 Function Level Auth, API8 Security Misconfiguration |
| OWASP Mobile Top 10 (2024) | M1 Improper Credential Usage, M2 Inadequate Supply Chain, M3 Insecure Authentication, M4 Insufficient Input/Output Validation, M5 Insecure Communication, M9 Insecure Data Storage |

### CWE mapping

Every rule includes at least one CWE reference. Common mappings used:

CWE-78 (Command Injection), CWE-79 (XSS), CWE-89 (SQL Injection),
CWE-94 (Code Injection), CWE-116 (Encoding), CWE-200 (Exposure of Sensitive
Information), CWE-287 (Improper Auth), CWE-306 (Missing Auth), CWE-326
(Inadequate Encryption Strength), CWE-327 (Broken Crypto Algorithm),
CWE-338 (Insecure PRNG), CWE-400 (Uncontrolled Resource Consumption),
CWE-502 (Deserialization), CWE-601 (Open Redirect), CWE-611 (XML Injection),
CWE-918 (SSRF), CWE-1321 (Prototype Pollution).

### Explicitly out of scope

- **Runtime verification.** The pipeline does not run the application or
  generate proof-of-concept exploits. That requires a sandbox and is out of
  scope for a static rule creation workflow.
- **IaC files.** Terraform and CloudFormation templates are covered by the
  `iac-security-review` toolkit, not this one.
- **Binary analysis or compiled artefacts.** Opengrep operates on source code.
  Compiled outputs, Docker images, and dependency JARs are not scanned.
- **C / C++ / Rust.** Not in scope for Stage 1 detection. Rules for these
  languages can be added in tailored mode, but Stage 1 will not map them.

---

## Installation

### Prerequisites

- **Claude Code** — required to invoke the skills.
- **Opengrep CLI** — optional but strongly recommended for rule validation.
  Install via:
  ```bash
  # macOS
  brew install opengrep

  # Linux / CI
  pip install opengrep
  # or download the binary from https://github.com/opengrep/opengrep/releases
  ```
  Without the CLI, Stage 2 falls back to static self-review for validation.
  Rules are still produced; they are just not syntax-checked by the engine.

### Install the toolkit

Copy the `.claude/` directory from this toolkit into the root of the project
you want to analyse:

```bash
cp -r opengrep-rules-creator/.claude/ /path/to/your-project/.claude/
```

If your project already has a `.claude/` directory, merge the `skills/`
subdirectory only:

```bash
cp -r opengrep-rules-creator/.claude/skills/ /path/to/your-project/.claude/skills/
```

### Optional: install the config template

```bash
cp opengrep-rules-creator/.claude/opengrep-config.yml /path/to/your-project/.opengrep-config.yml
```

Edit `.opengrep-config.yml` to set your project name, language list,
exclusion paths, and known issues file path. See [Configuration](#configuration)
for all available fields.

---

## Configuration

The pipeline reads `.opengrep-config.yml` from the project root if it exists.
All fields are optional; defaults are documented inline.

```yaml
# .opengrep-config.yml
# Copy this file to your project root and fill in the values below.
# All fields are optional — the pipeline uses safe defaults when absent.

# project_context: Optional free-text description of the project, its
# deployment environment, and trust boundaries. The more detail provided
# here, the more precisely rules can account for your threat model.
# Example: "Node.js API serving authenticated mobile clients. Config files
# and admin routes are trusted; all /api/v1/* endpoints are public."
project_context: ""

# languages: Restrict recon to specific languages. When omitted, all
# supported languages are auto-detected from file extensions.
# Supported values: js, ts, python, java, kotlin, go, php, swift, dart
languages: []

# depth: Override auto-scaling for Stage 1. When omitted, depth is chosen
# automatically based on source file count.
# Values: fast | balanced | deep
depth: ""

# exclude_paths: Additional glob patterns to exclude from recon, on top of
# built-in exclusions (.git, node_modules, vendor, dist, build, etc.).
# Example:
#   - "legacy/**"
#   - "**/*.generated.ts"
exclude_paths: []

# known_issues_file: Path to a KNOWN_ISSUES.md file containing previously
# identified vulnerabilities. When present, Stage 2 uses this as its
# highest-priority rule source. Stage 2 will offer to create this file
# interactively if it is absent.
# Example: "docs/security/KNOWN_ISSUES.md"
known_issues_file: ""

# output_dir: Directory where CODE_RECON.md and all rule output is written.
# Relative to the project root. Default: opengrep-rules-creator
output_dir: "opengrep-rules-creator"
```

---

## Usage guide

### The two-stage workflow

```
Stage 1: /opengrep-code-recon <target-dir>
             ↓
         opengrep-rules-creator/CODE_RECON.md

Stage 2: /opengrep-rule-creator <target-dir>
             ↓
         opengrep-rules-creator/rules/<rule-id>.yaml  (one per rule)
         opengrep-rules-creator/tests/<rule-id>.<ext>
         opengrep-rules-creator/logs/<rule-id>-validate.log
         opengrep-rules-creator/RULES_INDEX.md
```

---

### Stage 1 — `/opengrep-code-recon`

Maps the project and produces `CODE_RECON.md`.

```
/opengrep-code-recon <target-dir> [--exclude <glob>] [--lang <id>] [--depth <fast|balanced|deep>] [--no-deps]
```

**Arguments:**

| Argument | Required | Description |
|---|---|---|
| `<target-dir>` | Yes | Project root to scan. Relative or absolute path. |
| `--exclude <glob>` | No | Additional exclusion glob. Repeatable. Stacks on top of built-ins and config exclusions. |
| `--lang <id>` | No | Restrict discovery to one language. Values: `js`, `ts`, `python`, `java`, `kotlin`, `go`, `php`, `swift`, `dart`. When omitted, all languages are auto-detected. |
| `--depth <level>` | No | Override depth auto-scaling. Values: `fast`, `balanced`, `deep`. Default: auto-selected by file count. |
| `--no-deps` | No | Skip the dependency inventory step. Faster for very large monorepos. |

**Example invocations:**

```bash
# Full auto-detect recon on a project
> /opengrep-code-recon .

# Python-only recon with extra exclusions
> /opengrep-code-recon ./backend --lang python --exclude "tests/**"

# Fast pass on a large monorepo, skip deps
> /opengrep-code-recon . --depth fast --no-deps

# Explicit deep pass on a small service
> /opengrep-code-recon ./services/auth-service --depth deep
```

**What Stage 1 produces:**

Stage 1 maps the following and writes a summary of each to `CODE_RECON.md`:

- Language and framework detection (JS/TS, Python, Java/Kotlin, Go, PHP, Swift, Dart)
- Dependency inventory with security-interesting package flagging
- HTTP / API entry points and user-input sources (taint sources)
- Dangerous sinks: SQL, command execution, template injection, file operations, deserialization, open redirect, XSS
- Validator and sanitizer mapping (critical for false-positive reduction in Stage 2)
- Auth/trust boundaries (used as taint guards in Stage 2 rules)
- Data processing functions: serialization, crypto, file upload handling
- Business logic hot spots: payment flows, access control, session management
- Configuration file inventory with hardcoded secret detection (file:line only — values are redacted)
- Database schemas, ORM models, and raw SQL query classification
- Security hot-spot aggregation with rule opportunity ranking

Stage 1 also writes a structured YAML metadata block at the bottom of
`CODE_RECON.md` that Stage 2 consumes directly — this block contains the
taint sources, sinks, and sanitizers extracted from your codebase, formatted
so Stage 2 can reference them without re-reading any source files.

---

### Stage 2 — `/opengrep-rule-creator`

Reads `CODE_RECON.md` and creates Opengrep rules.

```
/opengrep-rule-creator <target-dir> [--mode <interview|known-issues|auto|tailored>] [--rule <description>] [--autonomous] [--lang <lang>]
```

**Arguments:**

| Argument | Required | Description |
|---|---|---|
| `<target-dir>` | Yes | Project root. Must contain `opengrep-rules-creator/CODE_RECON.md`. |
| `--mode <mode>` | No | Input strategy. See table below. Defaults to auto-selection based on available files. |
| `--rule <description>` | No | For `--mode tailored` only: short description of the specific rule to create (e.g. `"lodash merge prototype pollution"` or `"SNYK-JS-LODASH-15869619"`). |
| `--autonomous` | No | Skip per-rule operator approval. Creates all proposed rules without pausing. Use deliberately. |
| `--lang <lang>` | No | Restrict rule output to one language. |

**Modes:**

| Mode | When to use | What it does |
|---|---|---|
| `interview` | First run, no vulnerability history | Leads operator through a structured coverage interview. Surfaces OWASP gaps and opengrep-rules community rules relevant to detected stack. |
| `known-issues` | Team has an existing KNOWN_ISSUES.md | Reads the file and creates rules from it. No interview. |
| `auto` | CODE_RECON.md hot spots are sufficient | Proposes rules directly from Stage 1 findings and detected stack. No interview. |
| `tailored` | Single CVE or library-specific rule needed | Creates one precise rule for a specific vulnerability, CVE, or library method. Use with `--rule`. |
| *(omitted)* | Default | If KNOWN_ISSUES.md exists: `known-issues` + `auto`. Otherwise: `interview` + `auto`. |

**Example invocations:**

```bash
# Default mode — reads KNOWN_ISSUES.md if present, then auto from recon
> /opengrep-rule-creator .

# Interactive interview to discover coverage areas
> /opengrep-rule-creator . --mode interview

# Fully autonomous — create all proposed rules without approval gates
> /opengrep-rule-creator . --autonomous

# Tailored rule for a specific CVE
> /opengrep-rule-creator . --mode tailored --rule "SNYK-JS-LODASH-15869619 prototype pollution via _.merge"

# Tailored rule from a description
> /opengrep-rule-creator . --mode tailored --rule "unparameterized SQL query in user search endpoint"

# Restrict to Python rules only, using known issues
> /opengrep-rule-creator . --mode known-issues --lang python
```

**Rule creation loop (non-autonomous):**

For each rule candidate, Stage 2:

1. Extracts the relevant entry points, sinks, and sanitizers from
   `CODE_RECON.md` for that rule's category.
2. Writes the rule YAML with full metadata (CWE, OWASP, CVSSv3.1 vector,
   references, fix guidance).
3. Writes a test file with all four test case types: two vulnerable variants
   that must trigger, and safe/sanitized variants that must not.
4. Validates the rule with `opengrep validate --config` (if CLI is available).
5. Presents the rule and test file to the operator for approval.
6. On approval, appends an entry to `RULES_INDEX.md` and proceeds to the
   next rule.

**KNOWN_ISSUES.md:**

If `KNOWN_ISSUES.md` does not exist and the operator chooses to create it,
Stage 2 conducts a short interview (five questions) and drafts the file.
The interview covers: vulnerability classes seen, specific CVEs or pentest
findings, risky third-party packages, business-logic issues, and deployment
context. The file is written to `opengrep-rules-creator/KNOWN_ISSUES.md`
and shown to the operator for confirmation before rule creation begins.

---

### Running generated rules

After Stage 2 completes, run all generated rules against your project:

```bash
# Basic scan
opengrep scan \
  --config opengrep-rules-creator/rules/ \
  --json \
  . \
  > opengrep-results.json

# With cross-function taint tracking (recommended for rules annotated with --taint-intrafile)
opengrep scan \
  --config opengrep-rules-creator/rules/ \
  --taint-intrafile \
  --json \
  . \
  > opengrep-results-deep.json

# SARIF output for GitHub Security tab or other SARIF consumers
opengrep scan \
  --config opengrep-rules-creator/rules/ \
  --taint-intrafile \
  --sarif \
  . \
  > opengrep.sarif
```

**GitHub Actions integration:**

```yaml
- name: Run Opengrep custom rules
  run: |
    opengrep scan \
      --config opengrep-rules-creator/rules/ \
      --taint-intrafile \
      --sarif \
      . > opengrep.sarif
  continue-on-error: true

- name: Upload SARIF results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: opengrep.sarif
```

---

## Output file reference

All output is written to `<project-root>/opengrep-rules-creator/` (or the
`output_dir` value from `.opengrep-config.yml`).

| File | Stage | Description |
|---|---|---|
| `CODE_RECON.md` | 1 | Full recon report: tech stack, entry points, sinks, validators, hot spots, metadata block for Stage 2. |
| `KNOWN_ISSUES.md` | 2 | Known vulnerability register. Created interactively if absent. Operator-editable. |
| `RULES_INDEX.md` | 2 | Running index of all created rules: ID, language, category, OWASP, severity, CVSSv3 score, mode, validation status, file path. |
| `rules/<rule-id>.yaml` | 2 | Individual Opengrep rule. One file per rule. Ready to use with `opengrep scan --config`. |
| `tests/<rule-id>.<ext>` | 2 | Test file for the rule. Contains `# ruleid:` (must-trigger) and `# ok:` (must-not-trigger) annotations. |
| `logs/<rule-id>-validate.log` | 2 | Raw output of `opengrep validate --config` for this rule. |
| `logs/<rule-id>-validate-tests.log` | 2 | Raw output of `opengrep validate` run against the test file. |
| `logs/validation-summary.log` | 2 | One-line-per-rule summary: rule ID, validate result, timestamp. |

The `opengrep-rules-creator/` directory is the only location this toolkit
writes to. No source files, config files, or other project files are modified.

---

## FAQ

**Do I have to run Stage 1 before Stage 2?**

Yes. Stage 2 reads `CODE_RECON.md` at startup and will prompt you to run
Stage 1 if the file is missing. In `tailored` mode, Stage 2 can produce a
single rule with minimal context from the operator if Stage 1 has not been
run — but the resulting rule will not benefit from project-specific sanitizer
and entry-point data, making false positives more likely.

**Can I run Stage 2 multiple times?**

Yes. Stage 2 appends to `RULES_INDEX.md` rather than overwriting it, so
running it again adds new rules without losing existing ones. `KNOWN_ISSUES.md`
is also preserved between runs — add new entries to it and re-run Stage 2 to
generate rules for them.

**What languages are supported?**

Stage 1 detects and maps: JavaScript, TypeScript, Python, Java, Kotlin, Go,
PHP, Swift, and Dart. Stage 2 generates rules for any language that Opengrep
supports — including the above plus C#, Ruby, and others — when rules are
requested in `tailored` mode. Note that Stage 1 recon will not map C# or Ruby
code, so tailored rules for those languages will be less project-specific.

**Does this toolkit modify any of my project files?**

No. Both stages are read-only with respect to your project. The only writes
are to the `opengrep-rules-creator/` output directory (or the configured
`output_dir`). No source files, IaC templates, config files, or pipeline
definitions are touched.

**Does Stage 1 log or transmit any secrets it finds?**

No. When Stage 1 detects a potential hardcoded credential, it records the
file path and line number only — never the credential value — in
`CODE_RECON.md`. The value never appears in any output file.

**What if `opengrep validate` is not available?**

Stage 2 detects CLI availability at startup and falls back to a structured
static self-review checklist if `opengrep validate` is not found. Rules are
still produced with the same structure and metadata — they are just not
syntax-checked by the engine before being written. Install the Opengrep CLI
to get syntax validation.

**Can I use the generated rules with Semgrep instead of Opengrep?**

Opengrep is a fork of Semgrep CE and maintains backward compatibility for
most rule features. Rules using standard `pattern`, `pattern-not`,
`pattern-sources`, `pattern-sinks`, and `pattern-sanitizers` syntax should
work with Semgrep. Rules that use the `--taint-intrafile` flag or
Opengrep-specific taint constructs (constructor tracking, higher-order
function taint) may not be fully supported by Semgrep.

**What is the `tailored` mode for?**

Tailored mode creates a single, precise rule for a specific vulnerability,
CVE, or library method — without running the full interview or proposing a
rule queue. Use it when a specific library issue (e.g. a Snyk advisory) or
a recently disclosed CVE needs a rule immediately, without the overhead of a
full pipeline run. The rule is still validated and written to the same output
directory as pipeline rules.

**Is Stage 2 fully autonomous by default?**

No. By default, Stage 2 presents one rule at a time and waits for operator
approval before creating the next. This keeps a human in the decision loop
for rules that will be deployed to CI. The `--autonomous` flag removes the
approval gate for operators who want to generate a batch of rules for review
later.

---

*This toolkit does not represent the views of Anthropic. It is an independent
community contribution to the DevSecOps with Claude repository. Claude and
Claude Code are products of Anthropic, PBC. Opengrep is an independent
open-source project maintained by the Opengrep consortium.*
