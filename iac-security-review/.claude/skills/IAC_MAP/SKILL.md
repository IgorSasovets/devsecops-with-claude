---
name: iac-map
description: >-
  Stage 1 of the IaC Security Review pipeline. Scans a project directory to
  locate all IaC templates (Terraform .tf/.tfvars, CloudFormation .yaml/.json),
  maps the project structure, and produces PROJECT_MAP.md in iac-review-results/.
  Uses Grep/Glob first; reads files only when needed. Respects .iac-review-config.yml
  exclusions. Output feeds /iac-threat-model and /iac-audit. Use when asked to
  "map IaC", "scan project structure", or as the first step of an IaC review.
context: fork
argument-hint: "<target-dir> [--exclude <glob>] [--no-inventory]"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Bash(find:*)
  - Bash(ls:*)
  - Bash(wc:*)
  - Bash(head:*)
  - Bash(cat:*)
  - Bash(grep:*)
  - Bash(rg:*)
---

# /iac-map

Stage 1 of the IaC Security Review pipeline. Maps a project to find all IaC
templates and produces a structured `PROJECT_MAP.md` for downstream stages.

**Token efficiency contract:** Use Glob and Grep first. Read individual files
only when a targeted read is needed (e.g. to count resources in a small
template). Never read entire large files for structure discovery — use `head`
or `grep` instead.

**Tool fallback:** Prefer Glob/Grep tools. When unavailable, fall back to
`find`, `rg --files`, `grep -rn`, `ls -R` via Bash. Only these Bash commands
are permitted.

---

## Arguments

- `<target-dir>` (required) — project root to scan. Relative or absolute.
- `--exclude <glob>` — additional exclusion glob on top of config file and
  built-in defaults. Repeatable.
- `--no-inventory` — skip per-file resource inventory (faster for large repos;
  PROJECT_MAP.md will list files but not resource counts).

---

## Step 0 — Version check

Before proceeding, verify Claude Code version supports the `Task` tool (sub-agent
fan-out). If the version cannot be determined, note it as unknown and continue.
If the user is on a significantly outdated Claude Code version, recommend upgrading
before running the full pipeline.

---

## Step 1 — Load configuration

1. Look for `<target-dir>/.iac-review-config.yml`. If present, parse it:
   - `exclude_paths` — merge with built-in exclusions (see Step 2).
   - `iac_type` — narrows discovery scope (`terraform`, `cloudformation`, or `both`).
   - `output.results_dir` — override default output path (`iac-review-results`).
   - `output.include_resource_inventory` — override `--no-inventory`.
   - `environment` block — copy verbatim into PROJECT_MAP.md for downstream stages.
2. If config is absent, use defaults. Note the absence in PROJECT_MAP.md.
3. Merge CLI `--exclude` args on top of config exclusions.

**Built-in exclusions (always applied regardless of config):**
```
.git/**, .terraform/**, **/.terraform/**, *.tfstate, *.tfstate.backup,
*.tfstate.*.backup, **/node_modules/**, **/__pycache__/**, **/.terragrunt-cache/**,
**/.tfsec/**, **/.checkov/**, **/dist/**, **/build/**
```

---

## Step 2 — Discover IaC files

Run these discovery passes in order. Stop each pass as soon as you have
the file list — do NOT read file contents during discovery.

### 2a — Terraform discovery (skip if `iac_type: cloudformation`)

```bash
# Find all .tf files, excluding built-in and config exclusions
find <target-dir> -name "*.tf" -not -path "*/.terraform/*" -not -path "*/node_modules/*"
find <target-dir> -name "*.tfvars" -not -path "*/.terraform/*"
```

Or with Glob: `**/*.tf`, `**/*.tfvars` with exclusion filters applied.

### 2b — CloudFormation discovery (skip if `iac_type: terraform`)

CloudFormation files require two passes because they have no single file
extension — they can be `.yaml`, `.yml`, `.json`, or `.template`.

Pass 1 — find candidate files:
```bash
find <target-dir> \( -name "*.yaml" -o -name "*.yml" -o -name "*.json" -o -name "*.template" \) \
  -not -path "*/.terraform/*" -not -path "*/node_modules/*"
```

Pass 2 — filter to actual CFN templates (check for CFN signature without reading
the whole file):
```bash
grep -rl "AWSTemplateFormatVersion\|\"Resources\"\s*:" <candidate-files>
# or
grep -rl "AWSTemplateFormatVersion" <target-dir> --include="*.yaml" --include="*.yml"
grep -rl "AWSTemplateFormatVersion" <target-dir> --include="*.json" --include="*.template"
```

### 2c — Apply exclusions

Filter the discovered file list against all active exclusions (built-in +
config + CLI). Report how many files were excluded and why (category, not
individual paths, unless count < 5).

---

## Step 3 — Classify and group files

For each discovered file, assign:

| Field | Values |
|---|---|
| `type` | `terraform-module`, `terraform-root`, `terraform-vars`, `cfn-template`, `cfn-nested` |
| `provider` | `aws` (v1 only) |
| `path` | relative path from `<target-dir>` |

**Terraform classification rules:**
- File contains `terraform { required_providers` → `terraform-root`
- File is `*.tfvars` → `terraform-vars`
- File references `module "…"` source → `terraform-module` (check with Grep)
- Otherwise → `terraform-module` (generic)

**CloudFormation classification rules:**
- `Transform: AWS::Serverless` → SAM template (note it)
- `Type: AWS::CloudFormation::Stack` resource present → has nested stacks (note it)
- Otherwise → `cfn-template`

Group files by directory. If a directory contains 5+ `.tf` files, it is a
Terraform module root — label it as such.

---

## Step 4 — Resource inventory (skip if `--no-inventory`)

For each IaC file, extract a resource count and type list using targeted
Grep — do NOT read the full file unless it is under 50 lines.

**Terraform — count resource blocks:**
```bash
grep -c "^resource \"" <file>
# Extract resource types:
grep "^resource \"" <file> | sed 's/resource "\([^"]*\)".*/\1/' | sort | uniq -c | sort -rn
```

**CloudFormation — count Resources section entries:**
```bash
grep -c "^\s*Type: AWS::" <file>
# Extract resource types:
grep "^\s*Type: AWS::" <file> | sed 's/.*Type: //' | sort | uniq -c | sort -rn
```

Build a per-file summary table:
```
| File | Type | Resources | Top AWS Services |
|------|------|-----------|-----------------|
```

Also build a cross-repo resource type frequency table (aggregated across all files).

---

## Step 5 — Security-relevant metadata scan

Run targeted Grep passes across all discovered files to flag items that will
be high-value for downstream stages. Do NOT read files — Grep only.

**Hardcoded secrets / sensitive values:**
```bash
grep -rn "password\s*=\|secret\s*=\|api_key\s*=\|access_key\s*=" <iac-files> --include="*.tf"
grep -rn "NoEcho\s*:\s*false\|password\|secret" <iac-files> --include="*.yaml"
```

**Public exposure indicators:**
```bash
grep -rn "0\.0\.0\.0/0\|::/0\|PubliclyAccessible\s*:\s*true\|publicly_accessible\s*=\s*true" <iac-files>
grep -rn "\"public-read\"\|\"public-read-write\"\|ACL.*public" <iac-files>
```

**Encryption disabled:**
```bash
grep -rn "encrypted\s*=\s*false\|StorageEncrypted\s*:\s*false\|EnableDnsSupport" <iac-files>
```

**IAM wildcards:**
```bash
grep -rn '"\*"' <iac-files> --include="*.tf"
grep -rn '"Action"\s*:\s*"\*"\|"Resource"\s*:\s*"\*"' <iac-files>
```

Record matches as a "hot spots" list (file + line reference only — do not
extract full content here; that is Stage 3's job).

---

## Step 6 — Write PROJECT_MAP.md

Create `<target-dir>/iac-review-results/` if it does not exist.
Write `<target-dir>/iac-review-results/PROJECT_MAP.md` with the structure below.

```markdown
# IaC Project Map
Generated: <ISO8601 timestamp>
Target: <target-dir>
Config: <found | not found — using defaults>

## Project summary

| Field              | Value                        |
|--------------------|------------------------------|
| Project name       | <from config or directory name> |
| Provider           | aws                          |
| IaC frameworks     | <terraform | cloudformation | both> |
| Total IaC files    | <N>                          |
| Files excluded     | <N>                          |
| Terraform modules  | <N>                          |
| CFN templates      | <N>                          |
| Scan timestamp     | <ISO8601>                    |

## Environment context
<paste environment block from config, or "Not configured — downstream stages
will use conservative defaults for CVSSv3.1 environmental scoring.">

## Active exclusions
<list of all active exclusion globs (built-in + config + CLI)>

## File inventory

### Terraform files
<grouped by directory, with type classification>

| File | Type | Lines |
|------|------|-------|

### CloudFormation templates
<grouped by directory, with type classification>

| File | Type | Lines |
|------|------|-------|

## Resource inventory
<omitted if --no-inventory>

### Resource type frequency (all files)
| AWS Resource Type | Count | Files |
|------------------|-------|-------|

### Per-file resource summary
| File | Resource Count | Top Services |
|------|---------------|-------------|

## Security hot spots
<items found in Step 5 — file + line reference only>

### Potential hardcoded secrets
<file:line references>

### Public exposure indicators
<file:line references>

### Encryption flags
<file:line references>

### IAM wildcard usage
<file:line references>

## Recommended review focus areas
<3–7 focus areas derived from file inventory + hot spots. Format:>

| Priority | Focus Area | Rationale | Files |
|----------|-----------|-----------|-------|

## Notes for downstream stages
- `/iac-threat-model`: <key observations about architecture complexity>
- `/iac-audit`: <suggested scan order based on resource types found>
- Known issues file: <path from config, or "not configured">
```

---

## Step 7 — Hand back

Tell the user:

1. Files found: N Terraform (M modules, K roots) + P CloudFormation templates.
2. Files excluded: X (reasons by category).
3. Hot spots summary: count of each category.
4. Top 3 recommended focus areas.
5. Output written to: `<path>/PROJECT_MAP.md`
6. Next steps:
   - `> /iac-threat-model <target-dir>` — optional threat modeling (recommended for unfamiliar projects)
   - `> /iac-audit <target-dir>` — proceed directly to audit (uses PROJECT_MAP.md automatically)

---

## Constraints

- **Never read a file in full during discovery.** Use Grep/Glob for discovery;
  targeted `head` or `grep` for metadata extraction.
- **Never modify target files.** This skill is read-only.
- **Stay within `<target-dir>`.** Do not follow symlinks or `..` paths.
- **Cap resource inventory at 200 files.** If more are found, sample the
  largest 50 + 150 random and note the cap in the output.
