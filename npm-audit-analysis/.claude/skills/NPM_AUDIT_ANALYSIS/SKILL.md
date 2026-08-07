---
name: npm-audit-analysis
description: >-
  Ingests an npm audit JSON report (or runs npm audit --json itself), enriches
  each vulnerable dependency with source-code exploitability tracing, and
  produces a structured Markdown + JSON report under
  vulnerable-dependencies-analysis/. Adaptive depth: quick pass for all
  severities, deep call-chain trace for HIGH/CRITICAL only. Writes per-finding
  temp files as it goes so progress survives token limits. Use when asked to
  "analyse npm audit results", "check if vulnerable dependencies are exploitable",
  "review npm vulnerabilities", or "scan package dependencies for risk".
context: fork
argument-hint: "<project-dir> [--report <path/to/audit.json>] [--skip-dev] [--skip <pkg1,pkg2>] [--no-config]"
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Task
  - Bash(npm audit --json *)
  - Bash(npm ls --json *)
  - Bash(cat *)
  - Bash(ls *)
  - Bash(find *)
  - Bash(grep -rn *)
  - Bash(rg *)
  - Bash(wc -l *)
  - Bash(head *)
  - Bash(sed *)
  - Bash(mkdir *)
  - Bash(rm *)
---

# /npm-audit-analysis

Enriched npm dependency vulnerability analysis with source-code exploitability
tracing. Reads an npm audit JSON report (or generates one), maps each finding to
actual source code usage, and writes progressive per-finding reports plus a final
consolidated Markdown + JSON report under
`<project-dir>/vulnerable-dependencies-analysis/`.

Runs in a forked sub-agent context so verbose output does not pollute the main
conversation.

---

## Token Efficiency Contract

| What | Strategy |
|------|----------|
| Source files | `find` + `grep -l` for discovery; **never read in full** if > 100 lines |
| Dependency chains | One `npm ls --json <pkg>` call per package maximum |
| Advisory data | Extracted from audit JSON in Step 1 — not re-fetched per finding |
| Deep call-chain trace | Only for HIGH/CRITICAL; capped at 5 `grep -rn` calls per finding |
| Test/build files | Skipped entirely unless severity is CRITICAL and build-time execution is relevant |
| Context extraction | `grep -n` + `sed -n 'START,ENDp'` to read ±10–20 lines around hits only |
| Progress resilience | Temp file written after each finding — session interruptions lose at most one finding |

---

## Arguments

- `<project-dir>` (optional) — path to the Node.js project root. Defaults to the current working directory if omitted.
- `--report <path>` (optional) — path to a pre-generated `npm audit --json` output file. If omitted, the skill runs `npm audit --json` itself inside `<project-dir>`.
- `--skip-dev` (optional) — skip all vulnerabilities that are introduced exclusively through `devDependencies`. Overrides the config file value.
- `--skip <pkg1,pkg2,...>` (optional) — comma-separated list of package names to exclude from analysis entirely. Overrides the config file value.
- `--no-config` (optional) — ignore any `.npm-audit-skill.json` config file found in the project root.

---

## Output Contract

All output is written to `<project-dir>/vulnerable-dependencies-analysis/`. The
skill never writes to any path outside this directory and never modifies the
user's `package.json`, `package-lock.json`, `node_modules`, or any other project
file.

| File | When written | Contents |
|------|-------------|----------|
| `temp-DA-NNN.md` | After each finding is analysed | Per-finding detail: advisory, dependency context, code trace, exploitability, fix recommendation |
| `npm-audit-findings.json` | After all findings are complete | Machine-readable findings array with full metadata, sorted by exploitability then severity |
| `npm-audit-report.md` | After all findings are complete | Human-readable consolidated report: executive summary, per-finding detail, fix commands |

Temp files are deleted after `npm-audit-report.md` and `npm-audit-findings.json`
are successfully written. If writing the final report fails, temp files are
retained and the user is told which file to check.

---

## Step 0 — Pre-flight checks

### 0a. Resolve arguments

Parse `$ARGUMENTS` in this priority order:

1. `--report <path>` — path to a pre-generated audit JSON file.
2. First positional token — the `<project-dir>`. If absent, use the current working directory.
3. `--skip-dev` flag.
4. `--skip <pkg1,pkg2,...>` — comma-separated package skip list.
5. `--no-config` flag.

### 0b. Read project config (unless `--no-config`)

Look for `<project-dir>/.npm-audit-skill.json`. If present, parse it.
CLI flags override any value set in the config file. Missing keys use defaults:

```json
{
  "skipDev": false,
  "skipPackages": [],
  "outputDir": "vulnerable-dependencies-analysis"
}
```

### 0c. Verify environment

Check for each requirement and stop with an actionable error if any is missing:

| Requirement | Check | Error to show user |
|---|---|---|
| `package.json` | `ls <project-dir>/package.json` | "No package.json at `<project-dir>`. Is this a Node.js project root?" |
| `node_modules/` | `ls <project-dir>/node_modules` | "`node_modules` not found — run `npm install` in `<project-dir>` first." |
| `package-lock.json` | `ls <project-dir>/package-lock.json` | "`package-lock.json` not found — run `npm install` or `npm install --package-lock-only`." |

### 0d. Obtain the audit report

**If `--report <path>` was given:** read and parse the file as JSON. If parsing
fails, stop: `"Could not parse audit report at <path>. Ensure it was generated with: npm audit --json > audit.json"`

**If no `--report`:** run `npm audit --json` inside `<project-dir>`. A non-zero
exit code is expected when vulnerabilities exist — continue if the output is
valid JSON. If the output is not valid JSON, stop:
`"npm audit did not produce valid JSON. Try: npm audit --json > audit.json and pass it with --report."`

Create the output directory (`mkdir -p`). Tell the user:

```
✓ Audit report loaded — N vulnerabilities found (CRITICAL: X | HIGH: Y | MODERATE: Z | LOW: W)
  Output: <project-dir>/vulnerable-dependencies-analysis/
```

---

## Step 1 — Parse and group findings

### 1a. Extract vulnerabilities

From the audit JSON `vulnerabilities` map, collect for each entry:

- `name` — package name
- `severity` — critical / high / moderate / low
- `isDirect` — boolean
- `via` — array of advisory objects `{source, name, title, url, severity, cwe, cvss, range}` or string re-exports
- `effects` — downstream metavulnerable packages
- `range` — affected version range
- `nodes` — install paths
- `fixAvailable` — fix metadata (`{name, version, isSemVerMajor}` or `false`)

### 1b. Determine dev-only status

For each package, run `npm ls --json <pkg-name>` (one call per package, maximum).
Mark `isDevOnly: true` if the package is reachable only through `devDependencies`
chains and never required by any production dependency.

### 1c. Apply skip rules

Remove from the working set:
- Any package in `skipPackages`.
- If `skipDev` is true: any package where `isDevOnly === true`.

Report skipped packages in a single summary line before proceeding.

### 1d. Group and assign IDs

Group findings that share the same root advisory `source` integer — multiple
package paths exposing the same upstream advisory are one finding with multiple
paths. Assign stable IDs `DA-001`, `DA-002`, ... sorted by (severity DESC,
package name ASC).

Print a summary table before proceeding:

```
ID      Package        Severity   Direct  DevOnly  CVEs  Paths
DA-001  lodash         CRITICAL   yes     no       10    1
DA-002  express        HIGH       yes     no       2     3
```

---

## Step 2 — Source-code recon (run once, before per-finding analysis)

Build a coarse file map so per-finding analysis can target only relevant files.
This avoids repeating directory discovery for every finding.

1. Discover source files with:
   ```bash
   find <project-dir>/src <project-dir>/lib <project-dir>/app \
        <project-dir>/routes <project-dir>/controllers \
        <project-dir>/middleware <project-dir>/pages \
        <project-dir>/api <project-dir>/server* <project-dir>/index.* \
        -type f \( -name "*.js" -o -name "*.ts" -o -name "*.mjs" -o -name "*.cjs" \) \
        2>/dev/null | head -200
   ```
   Fall back to `find <project-dir> -maxdepth 5 -type f -name "*.js" ...` if
   none of the standard directories exist.

2. If source file count > 200, warn:
   `"Large codebase (N files) — grep-first approach will be used to stay token-efficient."`

3. Identify entry points from `package.json` `"main"` and `"scripts.start"` values,
   plus common filenames: `app.js`, `server.js`, `index.js`, `main.js`, `src/index.*`.

Store the file list and entry point set in memory for Step 3 — do not re-run
discovery per finding.

---

## Step 3 — Per-finding analysis (sequential, one finding at a time)

Process findings in DA-ID order. For each finding:

### 3a. Summarise the advisory

From the pre-parsed `via` objects (Step 1), extract:
- Advisory title and URL
- CWE list
- CVSS score
- Affected version range
- Installed version (read from `node_modules/<pkg>/package.json` `version` field)

### 3b. Trace source-code usage — Adaptive depth

**Quick pass (all severities):**

```bash
grep -rn --include="*.js" --include="*.ts" --include="*.mjs" \
     "require.*['\"]<pkg>['\"]\\|from ['\"]<pkg>['\"]" \
     <project-dir>/src <project-dir>/lib <project-dir>/app \
     <project-dir>/routes <project-dir>/index.* 2>/dev/null
```

For each file that imports the package:
- Use `grep -n "<method-name>" <file>` to find call sites.
- Extract ±10 lines of context with `sed -n '<start>,<end>p' <file>`.
- Identify which methods/functions of the package are called and whether they
  match the vulnerable function(s) named in the advisory.

**Never read a full file.** Discovery is grep-only; context is extracted with
`sed` line ranges.

**Deep pass (CRITICAL and HIGH only):**

For each file calling a vulnerable method, trace one level of callers:
- `grep -rn "<call-pattern>" <project-dir>` scoped to the source file list.
- Determine if the call site is reachable from a route handler, exported
  function, or public entry point by reading ±20 lines of context.
- **Cap at 5 `grep -rn` calls per finding.** If the cap is reached before the
  call chain is confirmed, mark: `"Call chain indeterminate — manual review recommended"`.

### 3c. Assess exploitability

Assign the following fields based on the trace:

| Field | Values |
|-------|--------|
| `usedInCode` | `true` / `false` / `unknown` |
| `vulnerableMethodUsed` | `true` / `false` / `unknown` |
| `reachableFromEntryPoint` | `true` / `false` / `unknown` |
| `exploitableByExternalUser` | `true` / `false` / `unknown` |
| `exploitabilityConfidence` | `high` / `medium` / `low` |
| `exploitabilityNotes` | one-paragraph rationale with specific file and line references |

Scoring rules:
- Package not imported anywhere → `usedInCode: false`, `exploitableByExternalUser: false`, confidence `high`.
- Imported but vulnerable method not found in calls → `vulnerableMethodUsed: false`, confidence `medium` (dynamic require or re-export possible).
- Vulnerable method called but not traceable to a public entry point → `reachableFromEntryPoint: false`, confidence `medium`.
- Vulnerable method called AND reachable from a public route or export → `exploitableByExternalUser: true`, confidence `high`.
- `isDevOnly: true` → `exploitableByExternalUser: false` unless dev tooling is deployed (e.g. webpack-dev-server in production).

### 3d. Compose fix recommendation

| `fixAvailable` condition | Recommendation |
|---|---|
| `isSemVerMajor: false` | `npm audit fix` — safe minor bump, no breaking changes expected |
| `isSemVerMajor: true` | `npm audit fix --force` — major version bump; review changelog for `<pkg>@<new-version>` before running |
| No fix available | Manual options: (1) pin to last safe version, (2) replace with alternative, (3) accept risk with documented compensating controls |
| Fix resolves multiple advisories in same package | Note all advisory IDs resolved by this single fix |

### 3e. Write temp file immediately

Write to `<project-dir>/vulnerable-dependencies-analysis/temp-<DA-ID>.md`:

```markdown
# <DA-ID>: <package-name> (<SEVERITY>)

**Status:** TEMP — analysis in progress
**Analysed at:** <ISO timestamp>

## Advisory Summary
- **Title:** <title>
- **URL:** <url>
- **CWE:** <list>
- **CVSS Score:** <score>
- **Affected Range:** <range>
- **Installed Version:** <version>
- **Fix Available:** <yes — npm audit fix [--force] | no — manual>

## Dependency Context
- **Direct dependency:** <yes/no>
- **Dev-only:** <yes/no>
- **Introduced via:** <dependency chain from package-lock.json>
- **Metavulnerability affects:** <downstream packages, if any>

## Source Code Usage
- **Imported in:** <file:line list, or "Not found in source">
- **Vulnerable methods called:** <list with file:line, or "Not detected">
- **Reachable from entry point:** <yes/no/unknown>

## Exploitability Assessment
- **Used in code:** <yes/no/unknown>
- **Vulnerable method used:** <yes/no/unknown>
- **Reachable from entry point:** <yes/no/unknown>
- **Exploitable by external user:** <yes/no/unknown>
- **Confidence:** <high/medium/low>
- **Notes:** <rationale citing specific files and lines>

## Code Trace
```
<file>:<line> — <context snippet>
  └─ called from: <caller file>:<line>  ← deep pass only, HIGH/CRITICAL
```

## Fix Recommendation
<fix command or manual steps>
```

This file is written immediately after analysis is complete for this finding.
If the session ends early, all written temp files remain available in the output
directory.

---

## Step 4 — Final consolidation (run only when all findings are analysed)

### 4a. Check completeness

Verify a `temp-DA-NNN.md` exists for every DA-ID in the working set. For any
missing (session interrupted), include a row in the final report marked
`"INCOMPLETE — analysis was interrupted before this finding was processed"`.

### 4b. Write JSON report

Write `<project-dir>/vulnerable-dependencies-analysis/npm-audit-findings.json`:

```json
{
  "project": "<project-dir>",
  "analysedAt": "<ISO 8601>",
  "auditSource": "<file path | 'npm audit --json (live run)'>",
  "config": {
    "skipDev": false,
    "skipPackages": [],
    "outputDir": "vulnerable-dependencies-analysis"
  },
  "summary": {
    "total": 0,
    "critical": 0,
    "high": 0,
    "moderate": 0,
    "low": 0,
    "skipped": 0,
    "exploitable": 0,
    "notExploitable": 0,
    "unknown": 0
  },
  "findings": [
    {
      "id": "DA-001",
      "package": "<name>",
      "severity": "<critical|high|moderate|low>",
      "isDirect": true,
      "isDevOnly": false,
      "installedVersion": "<version>",
      "advisories": [
        {
          "source": 0,
          "title": "<title>",
          "url": "<url>",
          "cwe": ["<CWE-N>"],
          "cvss": 0.0,
          "range": "<range>"
        }
      ],
      "dependencyPath": "node_modules/<pkg>",
      "metavulnAffects": [],
      "fixAvailable": {
        "available": true,
        "version": "<new-version>",
        "isMajorBump": false,
        "command": "npm audit fix"
      },
      "sourceUsage": {
        "usedInCode": true,
        "filesImported": ["<file>:<line>"],
        "vulnerableMethodUsed": true,
        "vulnerableMethodCalls": ["<file>:<line> — <snippet>"],
        "reachableFromEntryPoint": true,
        "entryPointPath": "<route-file>:<line> → <usage-file>:<line>"
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

Sort findings: `exploitableByExternalUser: true` first, then severity DESC,
then confidence DESC.

### 4c. Write Markdown report

Write `<project-dir>/vulnerable-dependencies-analysis/npm-audit-report.md`:

````markdown
# npm Dependency Vulnerability Analysis

**Project:** `<project-dir>`
**Analysed:** <ISO timestamp>
**Audit source:** <path or "live npm audit run">

---

## Executive Summary

| Metric | Count |
|--------|-------|
| Total vulnerabilities analysed | N |
| Critical | X |
| High | Y |
| Moderate | Z |
| Low | W |
| Skipped (config/flags) | S |
| **Exploitable by external user** | **E** |
| Not exploitable / dev-only | NE |
| Unknown (manual review needed) | U |

---

## Findings

### DA-001 — `<package>` (CRITICAL) ⚠️ EXPLOITABLE

| Field | Value |
|-------|-------|
| Advisory | [<title>](<url>) |
| CWE | <list> |
| CVSS | <score> |
| Affected range | <range> |
| Installed version | <version> |
| Direct dependency | yes/no |
| Dev-only | yes/no |
| Introduced via | <dependency path> |
| Metavulnerability affects | <downstream packages or none> |

**Exploitability:** Exploitable by external user — HIGH confidence
> <notes citing specific file:line>

**Code trace:**
```
<file>:<line> — <snippet>
  └─ <caller>:<line>
```

**Fix:** `npm audit fix` — bumps to `<version>`, no breaking changes expected.

---

### DA-002 — `<package>` (HIGH) ✅ NOT EXPLOITABLE

...

---

## Skipped Packages

| Package | Reason |
|---------|--------|
| <pkg> | Listed in `skipPackages` config |
| <pkg> | `isDevOnly: true` with `skipDev` enabled |

---

## Fix Commands

```bash
# Safe fixes (minor bumps, no breaking changes)
npm audit fix

# Major version bumps — review changelogs before running
# npm audit fix --force    ← affects: <pkg-list>

# Manual fixes (no automated fix available)
# <pkg>: pin to <safe-version> in package.json, then npm install
```

---

*Generated by /npm-audit-analysis*
````

### 4d. Clean up temp files

Delete all `temp-DA-*.md` files after the final reports are written successfully.
If writing either final file fails, retain all temp files and tell the user:
`"Final report write failed — temp files retained at <output-dir>/temp-DA-*.md"`

---

## Step 5 — Hand-off

```
✅ npm audit analysis complete

Findings: N total (CRITICAL: X | HIGH: Y | MODERATE: Z | LOW: W)
Exploitable by external user: E
Requires manual review: U

Output:
  📋 <project-dir>/vulnerable-dependencies-analysis/npm-audit-report.md
  📊 <project-dir>/vulnerable-dependencies-analysis/npm-audit-findings.json

Top exploitable findings:
  1. DA-001 — lodash (CRITICAL) — _.merge() reachable from POST /api/transform [src/routes/api.js:22]
  2. DA-002 — <pkg> (HIGH) — ...

Next steps:
  • Safe fixes:    npm audit fix
  • Major bumps:   npm audit fix --force  (review changelogs first)
  • Manual fixes:  see Fix Commands section in the report
  • Re-verify:     /npm-audit-analysis <project-dir> after applying fixes
```

---

## Constraints

- **Never executes application code.** No `node`, no `npx`, no test runners, no build commands. Source code is only read via `grep` and `sed`; it is never executed.
- **Never modifies user project files.** The only permitted writes are to `<project-dir>/<outputDir>/`. The skill does not touch `package.json`, `package-lock.json`, `node_modules/`, or any source file.
- **Never fetches external resources.** All advisory data comes from the npm audit JSON. No outbound HTTP calls are made.
- **Never fabricates line numbers or file paths.** Every `file:line` reference cited in a finding must be sourced from an actual `grep -n` result in this session. If the exact line cannot be confirmed, the finding notes `"line approximate — see function <name>"`.
- **Never writes outside the output directory.** If `outputDir` is a relative path, it is resolved relative to `<project-dir>` only.
- **Never suggests running `npm audit fix --force` without explicit warning.** Major-bump fixes are always flagged as requiring changelog review before execution.
- **Retains temp files on failure.** If the final report cannot be written, temp files are not deleted so progress is not lost.

---

## Customisation reference

### `.npm-audit-skill.json` (place in project root)

```jsonc
{
  // Set to true to skip vulnerabilities only reachable through devDependencies.
  // Useful if you are confident devDependencies are never shipped to production.
  "skipDev": false,

  // List package names to exclude from analysis entirely.
  // Use for internal stubs, known-safe forks, or packages already triaged externally.
  "skipPackages": [],

  // Output directory for temp files and final reports.
  // Relative to the project root. Default: "vulnerable-dependencies-analysis"
  "outputDir": "vulnerable-dependencies-analysis"
}
```

### CLI flags (override config file)

| Flag | Effect |
|------|--------|
| `--skip-dev` | Skip all dev-only dependency findings |
| `--skip pkg1,pkg2` | Skip named packages |
| `--report <path>` | Use pre-generated audit JSON instead of running npm audit |
| `--no-config` | Ignore `.npm-audit-skill.json` entirely |

### Error recovery after interrupted sessions

If the session was interrupted mid-analysis, re-invoke the skill with the same
arguments. The skill checks for existing `temp-DA-*.md` files and resumes from
the first DA-ID that does not have a corresponding temp file:
`"Resuming analysis from DA-00N — found N completed temp files."`
