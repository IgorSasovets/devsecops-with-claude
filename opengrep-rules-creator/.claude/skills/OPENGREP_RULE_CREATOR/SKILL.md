---
name: opengrep-rule-creator
description: >-
  Stage 2 of the Opengrep Rules Creator pipeline. Reads CODE_RECON.md produced
  by /opengrep-code-recon and creates production-ready Opengrep .yaml rules.
  Two input modes: reads KNOWN_ISSUES.md if present (or helps create it via
  interview), and/or runs an interactive interview to discover coverage areas.
  Proceeds rule-by-rule with operator approval; supports autonomous batch mode.
  Validates rules with opengrep CLI when available, falls back to static review.
  Tailored single-rule mode also available for specific CVE/library requests.
  Writes all output to opengrep-rules-creator/.
  Use when asked to "create opengrep rules", "write security rules", or as
  stage 2 of the Opengrep pipeline.
context: fork
argument-hint: "<target-dir> [--mode interview|known-issues|auto|tailored] [--rule <description>] [--autonomous] [--lang <lang>]"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Bash(find:*)
  - Bash(ls:*)
  - Bash(cat:*)
  - Bash(head:*)
  - Bash(opengrep:*)
  - Bash(which:*)
  - Bash(mkdir:*)
  - Bash(echo:*)
  - Bash(date:*)
  - AskUserQuestion
---

# /opengrep-rule-creator

Stage 2 of the Opengrep Rules Creator pipeline. Generates production-ready,
low-false-positive Opengrep rules tailored to the project's actual code patterns
discovered in Stage 1.

**Token efficiency contract:** Load CODE_RECON.md once and reference it
throughout. Do not re-read source files unless targeted rule validation
requires a specific pattern check. Write rules one at a time; only load
the full rule backlog when operator requests `--autonomous` mode.

---

## Arguments

- `<target-dir>` (required) — project root. Must contain
  `opengrep-rules-creator/CODE_RECON.md` from Stage 1 (or operator must
  confirm it will be created now via `/opengrep-code-recon`).
- `--mode <interview|known-issues|auto|tailored>` — input strategy:
  - `interview` — interactive Q&A with operator to discover coverage areas.
  - `known-issues` — reads/creates `KNOWN_ISSUES.md` only, no interview.
  - `auto` — reads CODE_RECON.md hot spots and proposes rules autonomously.
  - `tailored` — single-rule mode for a specific CVE/library/pattern (use
    with `--rule`).
- `--rule "<description>"` — for `tailored` mode: short description of the
  rule to create (e.g. "lodash merge prototype pollution" or
  "SNYK-JS-LODASH-15869619").
- `--autonomous` — skip per-rule approval; create all proposed rules without
  pausing. Use with care.
- `--lang <lang>` — restrict rule output to one language.

When `--mode` is omitted, the skill auto-selects:
- If `KNOWN_ISSUES.md` exists → `known-issues` + `auto` combined.
- Otherwise → `interview` + `auto` combined.

---

## Step 0 — Pre-flight

1. Confirm `<target-dir>/opengrep-rules-creator/CODE_RECON.md` exists.
   - If missing: prompt operator to run `/opengrep-code-recon <target-dir>`
     first. Offer to run it inline if operator confirms.
2. Read `CODE_RECON.md` in full (it is the compact structured output of Stage 1;
   reading it once is the correct token trade-off).
3. Check for opengrep CLI and confirm `validate` subcommand is available:
   ```bash
   which opengrep 2>/dev/null || which opengrep-ce 2>/dev/null
   opengrep --version 2>/dev/null
   opengrep validate --help 2>/dev/null | head -5
   ```
   Record availability. If `opengrep validate` is found and exits without
   "unknown command" errors, validation will use CLI (Step 6a).
   If not found or validate subcommand is unavailable, validation falls
   back to static review (Step 6b).
4. Create output directories:
   ```
   <target-dir>/opengrep-rules-creator/rules/        ← individual .yaml rules
   <target-dir>/opengrep-rules-creator/tests/        ← per-rule test files
   <target-dir>/opengrep-rules-creator/logs/         ← validation logs
   ```
5. State: **this skill does not execute target application code.** It may
   invoke `opengrep scan` against test fixture files only.

---

## Step 1 — Load KNOWN_ISSUES.md

### 1a — If `KNOWN_ISSUES.md` exists

Read `<target-dir>/opengrep-rules-creator/KNOWN_ISSUES.md` and extract:
- Each known vulnerability entry (CVE/ID, class, affected component, pattern).
- These become **high-priority rule candidates** added to the queue first.

### 1b — If `KNOWN_ISSUES.md` is missing and mode ≠ `tailored`

Ask the operator:

> "No KNOWN_ISSUES.md found. This file captures previously identified
> vulnerabilities so rules can be tailored to your history. Would you like to:
> **(A)** Create it now with my help (I'll interview you — ~5 min), or
> **(B)** Skip and work only from code recon + OWASP coverage?"

**If A — create KNOWN_ISSUES.md via interview:**

Ask the following questions one at a time (do not front-load all questions):

1. "What types of vulnerabilities have you seen in this codebase or similar
   ones? (e.g. SQLi, XSS, auth bypass, IDOR, race conditions)"
2. "Have there been any CVEs, bug bounty reports, or pentest findings you'd
   like to prevent recurring? Share IDs or descriptions."
3. "Are there specific libraries or third-party packages where you've had
   security issues?"
4. "Any business-logic issues (e.g. price manipulation, privilege escalation,
   account takeover flows)?"
5. "What is your deployment context? (web app / API / mobile / CLI / library)"

After answers, draft `KNOWN_ISSUES.md` using the template in Appendix A,
write it to `<target-dir>/opengrep-rules-creator/KNOWN_ISSUES.md`, and show
the operator for confirmation before proceeding.

---

## Step 2 — Coverage area interview (skip if `--mode known-issues` or `tailored`)

Lead the operator through a structured interview to identify rule coverage
areas. Present areas as suggestions based on CODE_RECON.md findings + OWASP
references. **Ask one question at a time.**

### 2a — Present suggested areas

Based on CODE_RECON.md, generate a personalised suggestion list. Template:

> "Based on the code recon, I suggest we cover these areas. Mark Y/N or
> add your own areas:
>
> **From your codebase:**
> [dynamically generated from CODE_RECON.md hot spots — list up to 8]
>
> **OWASP Top 10 Web / API / Mobile gaps detected:**
> [list applicable OWASP items not yet covered by validators found in recon]
>
> **From opengrep-rules community (relevant to your stack):**
> [list 3–5 from https://github.com/opengrep/opengrep-rules relevant to
>  detected languages/frameworks]"

### 2b — Suggested area categories by stack (reference list)

Use this reference to generate the personalised list above. Only surface
categories relevant to detected languages/frameworks:

**Web (JS/TS/Python/PHP/Go/Java):**
- Injection: SQLi, Command, LDAP, XPath, Template injection
- XSS: Reflected, Stored, DOM-based
- Auth: JWT weaknesses, session fixation, missing auth checks, CSRF
- IDOR / Broken access control
- Open redirect
- Path traversal / LFI
- Deserialization (pickle, yaml.load, unserialize)
- SSRF (Server-Side Request Forgery)
- Secrets / hardcoded credentials
- Insecure crypto (MD5/SHA1 for passwords, ECB mode, weak PRNG)
- Dependency confusion / typosquatting patterns
- DoS: regex complexity (ReDoS), unbounded loops on input
- Race conditions in auth flows
- GraphQL / REST mass assignment
- Log injection / log4shell patterns

**Mobile (Swift/Dart/Kotlin/Java):**
- Insecure data storage (SharedPreferences, UserDefaults, plain text files)
- Exported components without permission checks (Android)
- Insecure deeplink handling
- Certificate pinning bypass patterns
- Weak crypto / hardcoded keys
- Root/jailbreak detection bypass patterns
- Excessive permissions
- Unprotected IPC / intents

**API-specific:**
- Missing rate limiting
- Broken object-level authorization (BOLA/IDOR)
- Excessive data exposure in responses
- Mass assignment vulnerabilities
- API key exposure in logs / responses

### 2c — Confirm final coverage list

Show the operator the combined list (from KNOWN_ISSUES + interview answers +
suggested areas). Ask: "Shall I proceed with these, or remove/add anything?"

Do not proceed until operator confirms the list.

---

## Step 3 — Rule queue construction

Build an ordered queue of rule candidates. Priority order:

1. **KNOWN_ISSUES entries** (highest priority — directly from history)
2. **CODE_RECON.md hot spots** matched to confirmed coverage areas
3. **Coverage-area interview results** not yet matched to hot spots
4. **OWASP gaps** identified in Step 2

For each candidate, record:
```
id: <rule-id>                    # kebab-case, e.g. js-express-sql-injection
title: <short title>
language: <lang(s)>
category: <OWASP ref>
mode: <taint|pattern>            # see rule mode selection guide below
source_evidence: <file:line from CODE_RECON.md>
priority: <1-5>
```

**Rule mode selection guide:**

| Condition | Mode |
|---|---|
| User input flows to dangerous sink | `taint` (always prefer) |
| Configuration / static pattern only | `pattern` |
| Framework-specific decorator / annotation | `pattern` |
| Library method usage regardless of input | `pattern` or `taint` |
| Cross-function data flow needed | `taint` + `--taint-intrafile` note |

In non-autonomous mode, present the queue to the operator for review before
proceeding. The operator may reorder, skip, or add entries.

---

## Step 4 — Rule creation loop

Process queue **one rule at a time** unless `--autonomous` is set.

For each rule candidate:

### 4a — Before writing the rule

1. Search CODE_RECON.md for relevant entry points, sinks, and sanitizers
   for this rule's category.
2. Identify the **exact code pattern** that would trigger the rule (specific
   to this project, not generic).
3. Identify **safe patterns** (sanitizers, validators, parameterized forms)
   that must NOT trigger — these become `pattern-not` or `pattern-sanitizers`.

### 4b — Write the rule

Create `<target-dir>/opengrep-rules-creator/rules/<rule-id>.yaml`:

```yaml
rules:
  - id: <rule-id>
    # ── Metadata ──────────────────────────────────────────────────────────
    message: >
      <Clear 1-2 sentence description of the vulnerability and its impact.
       Include fix guidance inline.>
    severity: <CRITICAL|HIGH|MEDIUM|LOW|INFO>
    languages: [<lang(s)>]

    metadata:
      # Full metadata as agreed
      category: security
      cwe:
        - "CWE-<N>: <Name>"
      owasp:
        - "<OWASP Top 10 ref, e.g. A03:2021 – Injection>"
      cvss_v3:
        score: <0.0-10.0>
        vector: "CVSS:3.1/AV:<>/AC:<>/PR:<>/UI:<>/S:<>/C:<>/I:<>/A:<>"
      references:
        - "https://owasp.org/..."
        - "https://cwe.mitre.org/data/definitions/<N>.html"
        - "<CVE or advisory URL if applicable>"
      fix_guidance: >
        <Concrete 2-3 sentence fix recommendation specific to the
         detected pattern in this project.>
      confidence: <HIGH|MEDIUM|LOW>
      likelihood: <HIGH|MEDIUM|LOW>
      source: opengrep-rule-creator
      project_specific: true

    # ── Rule logic ─────────────────────────────────────────────────────────
    # (choose taint OR pattern block — not both)

    # --- TAINT MODE ---
    mode: taint
    pattern-sources:
      - pattern: <source pattern>
        # Add more sources as needed
    pattern-sanitizers:
      - pattern: <sanitizer pattern>  # from CODE_RECON.md validators
    pattern-sinks:
      - pattern: <sink pattern>
        # Focus on specific sinks found in CODE_RECON.md

    # --- OR: PATTERN MODE ---
    # pattern: <pattern>
    # pattern-not:
    #   - pattern: <safe pattern 1>
    #   - pattern: <safe pattern 2>
    # pattern-inside:
    #   - pattern: <context required>
```

**Rule quality checklist** (verify before writing test file):
- [ ] `id` is unique, kebab-case, starts with `<lang>-<framework>-<vuln>`
- [ ] `message` includes what the vulnerability is AND a concrete fix hint
- [ ] `severity` matches CVSSv3 score (CRITICAL≥9.0, HIGH≥7.0, MEDIUM≥4.0)
- [ ] All sanitizers found in CODE_RECON.md are listed in `pattern-sanitizers`
      or `pattern-not` (this is the primary false-positive control)
- [ ] Language list is specific (never use `generic` unless no alternative)
- [ ] `--taint-intrafile` noted in metadata if cross-function flow is needed
- [ ] CVSSv3 vector is filled completely (all 8 metrics)

### 4c — Write the test file

Create `<target-dir>/opengrep-rules-creator/tests/<rule-id>.<ext>`:

Include **all four test case types**:

```python
# ── Vulnerable (must trigger) ──────────────────────────────────────────

# ruleid: <rule-id>
<minimal vulnerable code snippet from the actual project pattern>

# ruleid: <rule-id>
<second variant — different coding style, same vulnerability>

# ── Safe / sanitized (must NOT trigger) ───────────────────────────────

# ok: <rule-id>
<same sink but with sanitizer/validator from CODE_RECON.md>

# ok: <rule-id>
<same sink but with parameterized / safe form>

# ok: <rule-id>
<hardcoded literal — not user input>

# ok: <rule-id>
<input that passes through an auth gate found in CODE_RECON.md>
```

### 4d — Non-autonomous: present and wait for approval

In non-autonomous mode, show the operator:
1. Rule YAML (abbreviated — sources, sinks, sanitizers, metadata).
2. Test cases.
3. Ask: "**[A]** Approve and proceed to next | **[M]** Modify this rule |
   **[S]** Skip this rule | **[Q]** Stop here"

Only proceed to Step 5 (validation) after operator approves.

---

## Step 5 — Opengrep-specific optimisations

Before validation, apply these Opengrep-specific enhancements:

### 5a — Taint intrafile flag

If the rule uses `mode: taint` and the source → sink chain in CODE_RECON.md
spans multiple functions within the same file, add to metadata:
```yaml
opengrep_flags:
  - "--taint-intrafile"
```
And note in the `message` that this flag is required for full detection.

### 5b — Higher-order function patterns

If the codebase uses callbacks, array methods (`map`, `forEach`, `filter`),
or custom iterators near sinks (detected in CODE_RECON.md), add alternative
`pattern-sinks` entries to capture taint flowing through those constructs.

### 5c — Constructor / field taint (Python/JS)

For OOP patterns where taint flows through class constructors or field
assignments (common in Django models, Express middleware), use Opengrep's
constructor taint tracking capability in pattern design.

---

## Step 6 — Validation

### 6a — CLI validation (when opengrep is available)

Use `opengrep validate` to verify the rule is structurally correct and
syntactically valid **without running a scan against target source files**.
This is the correct validation command — it catches schema errors, invalid
pattern syntax, missing required fields, and malformed YAML before any
scan is attempted.

```bash
# Step 1 — Validate rule syntax and schema
opengrep validate \
  --config <target-dir>/opengrep-rules-creator/rules/<rule-id>.yaml \
  2>&1 | tee <target-dir>/opengrep-rules-creator/logs/<rule-id>-validate.log

# Exit code 0 = valid. Non-zero = errors; read the log and fix before proceeding.
```

Interpret the output:
- **Exit 0, no errors** → rule is structurally valid; proceed to test annotation check.
- **`Invalid rule schema`** → fix YAML structure (missing required fields, bad
  indent, wrong type). Re-run validate after each fix.
- **`Invalid pattern`** → the pattern syntax is not parseable for the declared
  language. Check metavariable naming (`$VAR` not `$var`), ellipsis usage
  (`...`), and language-specific syntax constraints.
- **`Unknown key`** → remove or rename the unrecognised metadata field.

```bash
# Step 2 — Check test annotations against the rule
# opengrep validate also checks that test file annotations are consistent
# with the rule id when a test file is passed alongside the rule.
opengrep validate \
  --config <target-dir>/opengrep-rules-creator/rules/<rule-id>.yaml \
  <target-dir>/opengrep-rules-creator/tests/<rule-id>.<ext> \
  2>&1 | tee <target-dir>/opengrep-rules-creator/logs/<rule-id>-validate-tests.log
```

After both validate commands pass (exit 0), record the outcome:

```bash
echo "Rule: <rule-id> | validate: PASS | $(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  >> <target-dir>/opengrep-rules-creator/logs/validation-summary.log
```

**Do not proceed to the next rule until `opengrep validate` exits 0.**
If validate passes but the operator suspects the pattern logic is wrong
(e.g. it would miss a real variant), use the static review checklist in
6b as a complementary reasoning pass — not as a substitute for validate.

Log full output to `<logs/<rule-id>-validate.log>`.

### 6b — Static review (fallback when CLI unavailable)

Perform structured self-review:

**Pattern correctness:**
- [ ] Source pattern matches the exact input-read patterns in CODE_RECON.md
- [ ] Sink pattern matches the exact dangerous call site patterns found
- [ ] Sanitizer patterns cover all validators identified in CODE_RECON.md
- [ ] `pattern-not` covers all safe variants from the test file

**False positive risk assessment:**
- [ ] Would this rule fire on the `# ok:` test cases? (reason through each)
- [ ] Are there framework-level sanitizers not listed? (check CODE_RECON.md)
- [ ] Could the sink pattern match in a context where input is trusted?

**Coverage completeness:**
- [ ] Does the rule cover all common coding styles for this pattern?
- [ ] Are there equivalent sinks in other methods/functions not yet covered?

Document review findings in `<logs/<rule-id>-static-review.md>`.

---

## Step 7 — Rule index update

After each approved + validated rule, append to
`<target-dir>/opengrep-rules-creator/RULES_INDEX.md`:

```markdown
| Rule ID | Language | Category | OWASP | Severity | CVSS | Mode | Validated | File |
|---|---|---|---|---|---|---|---|---|
| <rule-id> | <lang> | <category> | <owasp> | <severity> | <score> | <taint/pattern> | <validate-pass/static> | rules/<rule-id>.yaml |
```

---

## Step 8 — Tailored mode (`--mode tailored`)

For single-rule creation based on a specific CVE, library, or operator request:

1. Parse `--rule "<description>"` — extract: library name, method name,
   CVE ID (if any), vulnerability class.
2. If a CVE ID is given, fetch the advisory for technical details:
   ```
   https://security.snyk.io/vuln/<CVE-ID>
   https://nvd.nist.gov/vuln/detail/<CVE-ID>
   ```
3. Search CODE_RECON.md for usages of the library/method in the project.
4. Build a rule that:
   - Matches only the vulnerable method/API — not the library generally.
   - Includes the library version constraint in metadata if applicable.
   - Uses `pattern-not` to exclude already-patched or safe call patterns.
5. Show operator the rule + test file for approval.
6. Validate per Step 6.
7. Write to `rules/<rule-id>.yaml` and update `RULES_INDEX.md`.

**Example — Lodash prototype pollution (SNYK-JS-LODASH-15869619):**
```yaml
rules:
  - id: js-lodash-merge-prototype-pollution
    message: >
      _.merge() called with user-controlled input can cause prototype
      pollution. Use a safe merge alternative like lodash/fp merge or
      validate that keys do not include __proto__ / constructor.
    severity: HIGH
    languages: [javascript, typescript]
    mode: taint
    pattern-sources:
      - pattern: req.body
      - pattern: req.query
    pattern-sinks:
      - pattern: _.merge($OBJ, ...)
      - pattern: merge($OBJ, ...)
    pattern-not:
      - pattern: _.merge({}, ...)   # merge into a fresh object is safe
    metadata:
      cwe: ["CWE-1321: Improperly Controlled Modification of Object Prototype Attributes"]
      owasp: ["A08:2021 – Software and Data Integrity Failures"]
      references:
        - "https://security.snyk.io/vuln/SNYK-JS-LODASH-15869619"
      fix_guidance: >
        Replace _.merge() with structuredClone() + Object.assign(), or use
        lodash/fp merge which does not mutate prototypes. Alternatively,
        validate input keys with a sanitizer that strips __proto__ and
        constructor before merging.
```

---

## Step 9 — Final summary

After all rules are created (or operator stops), produce a summary:

```markdown
# Opengrep Rules Creation — Session Summary
Date: <ISO8601>
Target: <target-dir>

## Rules Created
<N> rules created, <M> skipped, <K> modified.

## Coverage Map

| OWASP Category | Rules | Languages | Status |
|---|---|---|---|

## Validation Results
- CLI validated: <N>
- Static reviewed only: <N>
- Failures / iterations needed: <N>

## Run Instructions

### Scan with all created rules
\`\`\`bash
opengrep scan \
  --config <target-dir>/opengrep-rules-creator/rules/ \
  --json \
  <target-dir>/ \
  > opengrep-results.json
\`\`\`

### For rules requiring --taint-intrafile
\`\`\`bash
opengrep scan \
  --config <target-dir>/opengrep-rules-creator/rules/ \
  --taint-intrafile \
  --json \
  <target-dir>/ \
  > opengrep-results-deep.json
\`\`\`

### CI integration (GitHub Actions)
\`\`\`yaml
- name: Opengrep scan
  run: |
    opengrep scan \
      --config opengrep-rules-creator/rules/ \
      --taint-intrafile \
      --sarif \
      . > opengrep.sarif
  continue-on-error: true
\`\`\`

## Next Steps
- Review RULES_INDEX.md for full coverage map.
- Run against full codebase and review findings.
- False positive? Add pattern-not entries and re-validate.
- Add new known issues to KNOWN_ISSUES.md and re-run /opengrep-rule-creator.
```

---

## Appendix A — KNOWN_ISSUES.md Template

```markdown
# Known Issues

This file documents previously identified vulnerabilities to guide
Opengrep rule creation. Add entries using the format below.

## Format

Each entry represents one known vulnerability class or specific finding.
The more detail provided, the more precise the generated rule.

---

## Entry Template

### KI-001: <Short Title>

| Field | Value |
|---|---|
| ID | KI-001 |
| CVE / Reference | CVE-XXXX-XXXX or internal ticket |
| Vulnerability class | SQLi / XSS / IDOR / etc. |
| Affected component | e.g. `src/api/users.js` — `getUserById()` |
| Affected language | Python / JS / Go / etc. |
| Status | Open / Patched / Won't Fix |
| Severity | Critical / High / Medium / Low |
| Description | |
| Root cause pattern | e.g. "User-supplied ID concatenated into SQL query" |
| Safe alternative | e.g. "Use parameterized queries via `.prepare()`" |
| Recurrence risk | High / Medium / Low |

---

<!-- Add entries below -->
```

---

## Constraints

- **One rule per YAML file.** Never combine multiple rules into one file.
- **No `generic` language.** Always use a specific language identifier.
- **Test-first mindset.** The test file defines expected behaviour; the rule
  must satisfy it — not the other way around.
- **False-positive reduction is non-negotiable.** Every sanitizer and safe
  pattern found in CODE_RECON.md must appear in the rule as a guard.
- **Operator approval required between rules** (unless `--autonomous`).
  Do not create the next rule until the current one is approved or skipped.
- **Never execute target application code.** Rule validation uses
  `opengrep validate --config` only — no scan is run against the target
  codebase. The skill does not build, run, or instrument any application code.
- **No `todook` or `todoruleid` annotations** in test files.
- **CVSSv3 vector must be complete.** All 8 metrics required; do not leave
  any as placeholders.
