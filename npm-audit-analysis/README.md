# npm Audit Analysis

Enriched npm dependency vulnerability analysis for Node.js projects. This toolkit
ingests an `npm audit` JSON report (or generates one), traces each vulnerable
dependency to actual usage in your source code, assesses real-world exploitability,
and produces a structured Markdown + JSON report — so you know which vulnerabilities
are worth fixing urgently and which are noise.

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

`npm audit` tells you that a vulnerability exists. It does not tell you whether
your application actually uses the vulnerable code path, or whether an external
attacker can reach it. The result is that raw audit output routinely flags
vulnerabilities in packages that are never imported, imported but only via
dev tooling, or imported but the vulnerable method is never called — none of
which represent real risk in production.

This toolkit solves that gap with three design decisions:

**Source tracing before severity judgement.** Before assigning any exploitability
verdict, the skill searches for actual import statements and vulnerable method
calls in your source code. A CRITICAL advisory against a package your application
never imports is not a CRITICAL risk — it is a false positive that wastes triage
time and creates alert fatigue. The skill surfaces this distinction explicitly in
the report.

**Adaptive depth, bounded cost.** A quick grep pass runs for every finding at
every severity level. A deeper call-chain trace, which follows usage back toward
route handlers and entry points, runs only for HIGH and CRITICAL findings where
the investment is justified. The deep pass is capped at five grep calls per
finding so token cost stays predictable even on large codebases.

**Progressive persistence.** Analysis results are written to disk after each
individual finding, not only at the end. If a session hits a token limit or is
interrupted, every completed finding is already saved. The next invocation
resumes from where it left off. No work is lost.

---

## Benefits

A DevSecOps engineer using this toolkit can expect:

- **Triage focus.** The report ranks findings by exploitability, not raw CVSS
  score. Findings confirmed as reachable from a public entry point appear first.
  Findings in unused or dev-only packages appear last or are filtered out.

- **Evidence-based verdicts.** Every exploitability assessment cites the specific
  file and line where the vulnerable package is imported and where the vulnerable
  method is called. "Not exploitable" is backed by negative grep evidence, not
  assumption.

- **Actionable fix commands.** The report distinguishes safe fixes (`npm audit fix`)
  from major-version bumps (`npm audit fix --force`) and manual-only cases, with
  the specific package and target version named for each.

- **Resilience on large projects.** The per-finding temp file approach means that
  a project with 50 audit findings can be analysed across multiple sessions if
  needed, without re-processing completed findings.

**Scope and limitations:**

- This skill performs static analysis only. It reads source files and reasons about
  them — it does not execute your application, does not send requests to a running
  server, and cannot confirm dynamic exploitability (e.g. vulnerabilities triggered
  only by specific runtime conditions or feature flags).
- Call-chain depth is bounded. Very long or dynamic call chains (e.g. calls through
  plugin registries, dynamic `require()`, heavily abstracted middleware) may be
  marked as `"Call chain indeterminate"` and flagged for manual review.
- The skill analyses the version of packages currently installed in `node_modules`.
  If `node_modules` is out of sync with `package-lock.json`, results may not
  reflect the actual installed state. Run `npm install` before invoking the skill.

---

## Standards and checks covered

| Standard | Coverage |
|----------|----------|
| CVSSv3.1 | CVSS scores are read from the npm advisory data and included in the report. The skill does not re-calculate CVSS scores. |
| CWE | CWE identifiers are extracted from advisory data and included per finding. |
| npm advisory database | All advisories surfaced by `npm audit --json` against the npm registry. Private registries are supported if `npm audit` is configured to use them. |
| OWASP A06:2021 (Vulnerable and Outdated Components) | This toolkit directly implements the analysis workflow for this OWASP Top 10 category. |

**Explicitly excluded:**

- License compliance analysis — use a dedicated licence checker (e.g. `license-checker`) for this.
- Supply-chain integrity verification (signature checking, provenance) — use `npm audit signatures` separately.
- Runtime vulnerability detection — this toolkit is static analysis only.
- Vulnerabilities in non-npm ecosystems (Python, Go, Java, etc.) — this toolkit is Node.js / npm only.

---

## Installation

### Prerequisites

- Claude Code installed and configured.
- Node.js project with `package.json`, `package-lock.json`, and `node_modules/` present.
  If `node_modules` is absent, run `npm install` first.

### Step 1 — Copy the toolkit into your project

```bash
cp -r npm-audit-analysis/.claude/ /path/to/your-project/.claude/
```

This places the skill at `.claude/skills/NPM_AUDIT_ANALYSIS/SKILL.md` inside
your project, which is exactly where Claude Code expects it.

### Step 2 — (Optional) Add a project config file

Copy the config template to your project root and edit it:

```bash
cp npm-audit-analysis/.npm-audit-skill.json /path/to/your-project/.npm-audit-skill.json
```

Open the file and adjust the values. All fields are optional — defaults are
used for any key that is absent. See [Configuration](#configuration) below.

### Step 3 — Open Claude Code

```bash
cd /path/to/your-project
claude
```

The skill is now available as `/npm-audit-analysis`.

---

## Usage guide

### Basic invocation — let Claude run the audit

```bash
/npm-audit-analysis
```

Runs `npm audit --json` in the current directory, analyses all findings, and
writes the report to `./vulnerable-dependencies-analysis/`.

### Provide a pre-generated report

```bash
/npm-audit-analysis --report ./audit.json
```

Skips the `npm audit` run and uses the provided file instead. Useful in CI
environments where the audit is run as a separate pipeline step, or when you
want to analyse a snapshot from a specific point in time.

To generate the report file:

```bash
npm audit --json > audit.json
```

### Specify a project directory

```bash
/npm-audit-analysis /path/to/my-project
```

Useful when running Claude Code from a different directory than the project root.

### Skip dev-only dependencies

```bash
/npm-audit-analysis --skip-dev
```

Excludes any vulnerability that is introduced exclusively through
`devDependencies`. Appropriate when you are confident your build pipeline
does not ship dev packages to production.

### Skip specific packages

```bash
/npm-audit-analysis --skip legacy-stub,internal-shim
```

Excludes named packages from analysis entirely. Use for packages that have
already been triaged externally, internal forks known to be safe, or packages
you are actively planning to replace.

### Combine flags

```bash
/npm-audit-analysis /path/to/project --report ./audit.json --skip-dev --skip legacy-stub
```

### Ignore the config file

```bash
/npm-audit-analysis --no-config
```

Ignores `.npm-audit-skill.json` and uses defaults (or CLI flags only).

### Resume an interrupted session

Re-invoke with the same arguments. The skill detects existing `temp-DA-*.md`
files in the output directory and resumes from the first finding that is not
yet complete:

```bash
/npm-audit-analysis
# Output: "Resuming analysis from DA-006 — found 5 completed temp files."
```

---

## Configuration

The `.npm-audit-skill.json` file in your project root controls default behaviour.
CLI flags always override config file values.

```jsonc
{
  // Set to true to skip vulnerabilities only reachable through devDependencies.
  // Useful when you are confident devDependencies are never deployed to production.
  // CLI override: --skip-dev
  "skipDev": false,

  // List of package names to exclude from analysis entirely.
  // Use for internal stubs, known-safe forks, or packages already triaged externally.
  // CLI override: --skip pkg1,pkg2
  "skipPackages": [],

  // Output directory for temp files and final reports.
  // Relative to the project root. Created automatically if it does not exist.
  // Default: "vulnerable-dependencies-analysis"
  "outputDir": "vulnerable-dependencies-analysis"
}
```

---

## Output file reference

All output files are written to `<project-dir>/<outputDir>/`
(default: `vulnerable-dependencies-analysis/`). The skill never writes to any
path outside this directory.

| File | Stage | Contents |
|------|-------|----------|
| `temp-DA-NNN.md` | Step 3 — written per finding, immediately after analysis | Per-finding detail: advisory metadata, dependency context, source code trace, exploitability assessment, fix recommendation. Deleted after final reports are written successfully. |
| `npm-audit-findings.json` | Step 4 — written after all findings are complete | Machine-readable findings array. Includes all advisory metadata, source usage, exploitability verdict, and fix command per finding. Sorted by exploitability then severity. Suitable for ingestion by other tools or CI gates. |
| `npm-audit-report.md` | Step 4 — written after all findings are complete | Human-readable consolidated report. Includes executive summary table, per-finding sections with code traces, skipped package list, and ready-to-run fix commands. |

### `npm-audit-findings.json` schema (abbreviated)

```json
{
  "project": "<absolute path>",
  "analysedAt": "<ISO 8601>",
  "auditSource": "<file path | 'npm audit --json (live run)'>",
  "config": { "skipDev": false, "skipPackages": [], "outputDir": "..." },
  "summary": {
    "total": 0, "critical": 0, "high": 0, "moderate": 0, "low": 0,
    "skipped": 0, "exploitable": 0, "notExploitable": 0, "unknown": 0
  },
  "findings": [
    {
      "id": "DA-001",
      "package": "<name>",
      "severity": "critical",
      "isDirect": true,
      "isDevOnly": false,
      "installedVersion": "<version>",
      "advisories": [{ "source": 0, "title": "", "url": "", "cwe": [], "cvss": 0.0, "range": "" }],
      "fixAvailable": { "available": true, "version": "", "isMajorBump": false, "command": "" },
      "sourceUsage": {
        "usedInCode": true,
        "filesImported": ["<file>:<line>"],
        "vulnerableMethodUsed": true,
        "vulnerableMethodCalls": ["<file>:<line> — <snippet>"],
        "reachableFromEntryPoint": true,
        "entryPointPath": "<route>:<line> → <usage>:<line>"
      },
      "exploitability": {
        "exploitableByExternalUser": true,
        "confidence": "high",
        "notes": "<rationale>"
      },
      "fixRecommendation": "<command or manual steps>"
    }
  ]
}
```

---

## FAQ

**Is running all steps required?**

There is only one skill and one command. The five internal steps (pre-flight,
parse, recon, per-finding analysis, consolidation) run automatically in sequence.
There is no multi-stage pipeline to orchestrate manually.

**Does the skill modify any of my project files?**

No. The only files the skill writes are inside the output directory
(`vulnerable-dependencies-analysis/` by default). It does not touch
`package.json`, `package-lock.json`, `node_modules/`, or any source file.

**Does the skill run `npm audit fix` or apply any changes?**

No. The skill is read-only with respect to dependencies. Fix commands are
suggested in the report output for the engineer to review and run manually.
`npm audit fix --force` is always explicitly flagged as requiring changelog
review before use.

**What if `node_modules` is not installed?**

The skill will stop at the pre-flight check and tell you to run `npm install`
first. Both `node_modules/` and `package-lock.json` are required because the
skill uses `npm ls --json` to resolve dependency chains and reads installed
package versions from `node_modules/<pkg>/package.json`.

**Does this work with monorepos or npm workspaces?**

Point `<project-dir>` at the workspace root where `package-lock.json` lives.
The skill will discover source files across subdirectories (up to depth 5 in the
fallback scan). Workspace-specific `node_modules` hoisting is handled correctly
by `npm ls --json`. For very large monorepos (> 200 source files), the skill
will warn and switch to a grep-first approach to stay token-efficient.

**What npm versions are supported?**

The skill uses the npm audit JSON format as documented for npm v7 and later (the
Bulk Advisory Endpoint format). npm v6 and earlier use a different JSON schema
and are not supported. Run `npm --version` to check; upgrade with
`npm install -g npm` if needed.

**What if there is no fix available for a vulnerability?**

The report will say so explicitly and suggest three manual options: (1) pin to
the last safe version, (2) replace the package with a maintained alternative,
(3) accept the risk with documented compensating controls. The skill does not
choose between these — that decision requires context about your application that
only you have.

**Is the exploitability verdict a guarantee?**

No. The skill performs static analysis — it reads code and reasons about it
without executing anything. A verdict of `"not exploitable"` means the skill
could not find evidence of the vulnerable code path being reachable; it does not
mean the vulnerability cannot be triggered under all conditions. Dynamic call
patterns, plugin architectures, and runtime-generated code may not be visible to
static analysis. Treat `"unknown"` findings and any finding in a critical package
as requiring manual review.

---

*This toolkit does not represent the views of Anthropic. It is an independent
community contribution that uses Claude Code. Claude and Claude Code are products
of Anthropic, PBC.*
