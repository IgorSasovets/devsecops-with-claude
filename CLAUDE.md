# DevSecOps with Claude — Project Configuration
# This file is loaded automatically by Claude Code at the start of every session.
# It applies to all work done within this repository.

## What this repository is

A community-maintained collection of Claude Code skills, CLAUDE.md configurations,
custom slash commands, and reference catalogs that help DevSecOps engineers use
Claude to improve the security of infrastructure and deployed workloads.

The current contents:
- `iac-security-review/` — four-stage IaC template security review pipeline
  (`/iac-map`, `/iac-threat-model`, `/iac-audit`, `/iac-reassess`)

Read `README.md` for the full picture before starting any work session.

---

## Orientation: read before touching anything

When you start a session in this repository, always run this sequence before
making any changes:

1. `ls -1` at the repository root to see the current toolkit folders.
2. Read `README.md` for structure rules and contribution standards.
3. For the specific toolkit you are working on, read its `README.md`.
4. For any skill you are modifying, read its `SKILL.md` in full.

Never modify a file without first reading it. Never create a new toolkit folder
without reading the structure rules in `README.md` first.

---

## Repository structure rules (enforced)

Every toolkit lives in its own top-level folder. A toolkit folder is only valid if:

- It has a `README.md` covering: overview, philosophy, benefits, standards covered,
  installation, usage guide, output file reference, and FAQ.
- All Claude-specific files (skills, commands, rules, config templates) live inside
  a `.claude/` subdirectory within that folder.
- Reference files used by skills at runtime (threat catalogs, benchmark extracts)
  live inside the skill's own folder or a named subfolder — never at the repo root.

If you are creating a new toolkit or file and these conditions cannot be met, stop
and ask the user to clarify scope before proceeding.

---

## How to work on this repository

### When asked to add a new toolkit or skill

1. Use plan mode. Outline the full folder structure, all files to be created, and
   their purposes before writing anything. Wait for confirmation.
2. Create the README for the new toolkit first, before writing any skill or reference
   file. The README defines what the toolkit does and why — everything else follows
   from it.
3. Create skills in order: SKILL.md frontmatter → skill body → reference files.
   Never leave frontmatter fields empty or as placeholders.
4. After creating all files, verify the folder structure matches the layout in
   `README.md` under "Repository structure".

### When asked to modify an existing skill or reference file

1. Read the file in full before making any change.
2. Read the toolkit's README to understand the design intent.
3. Make the smallest change that addresses the request. Do not refactor or restructure
   anything that was not part of the request.
4. After editing, verify the change does not break any cross-references (other skills
   that read the modified file, README sections that describe it).

### When asked to review a contribution or PR

Apply the checklist from `README.md` under "Pull request template". Report each
item as PASS, FAIL, or N/A. Do not approve a contribution that fails any of:
- Missing or incomplete README
- SKILL.md frontmatter with empty or placeholder fields
- Any hardcoded credential, account ID, ARN, or environment-specific value
- A skill that writes outside its designated output directory
- Fabricated CVSS scores, CIS control numbers, or CVE references

---

## Absolute constraints — these never change

**Read-only with respect to repository content.**
When helping a user understand or use the tools in this repository, Claude reads
files from the repository. It does not modify repository files unless the user has
explicitly asked to edit or create a specific file and has confirmed the intent.

**Read-only with respect to user infrastructure.**
The tools in this repository are analysis aids. Claude does not execute Terraform,
deploy CloudFormation stacks, call AWS APIs, modify cloud resources, or run any
command that touches live infrastructure. If asked to do any of these things,
decline and explain what the right tool is (Terraform CLI, AWS CLI, etc.).

**No fabrication.**
When working on skills or reference files that reference security standards (CIS,
NIST, OWASP, STRIDE, CVSSv3.1, CVE), every claim must be accurate. Do not invent
control numbers, scores, or vulnerability identifiers. If uncertain whether a claim
is accurate, say so and leave the field blank rather than guessing.

**No credentials or hardcoded values.**
No file in this repository may contain real AWS account IDs, IAM role ARNs, IP
addresses, API keys, or any value specific to a real environment. Config templates
must use placeholder values with comments. If you notice such a value in a file you
are reading, flag it immediately before doing anything else.

**One concern per commit.**
When creating or editing files, keep each logical change atomic. Do not mix a new
skill with a README fix with a reference file update in a single operation. If the
user asks for multiple unrelated changes, address them one at a time.

---

## Writing standards for this repository

All content must be in English.

**Prose style:** Clear and direct. Write for an experienced security engineer, not
a beginner. Avoid marketing language ("powerful", "seamlessly", "effortlessly").
Say what a thing does, not how impressive it is.

**SKILL.md bodies:** Use imperative step-by-step instructions. Claude follows these
instructions when the skill is invoked — they are directives, not descriptions.
Present tense, active voice: "Read the config file. Extract the exclusion list."
Not: "The skill will read the config file and extract the exclusion list."

**README files:** Prose sections (overview, philosophy, FAQ answers) use full
sentences. Technical sections (command reference, output file tables) use tables
and lists. Do not mix styles within a section.

**Reference catalogs (threat catalogs, benchmark files):** Every entry must use
the file's established column format. Never add a partial row. Source and version
must appear in the file's header comment block.

---

## Token efficiency rules

When working in this repository, apply these rules to every file operation:

- Use `Glob` and `Grep` to discover files before reading them.
- Use `head` or targeted `grep` before reading a large file in full.
- Read the README for a toolkit before reading any of its skill files — the README
  tells you what exists and where.
- When multiple skills need to be read, read them one at a time as needed rather
  than all upfront.
- When a reference file exceeds 200 lines, grep for the specific section needed
  rather than reading the whole file.

---

## Escalation — when to stop and ask

Stop and ask the user before proceeding if:

- A request would require creating a file outside a toolkit folder (at the repo root,
  or in a path that does not follow the structure rules).
- A request asks you to modify more than one toolkit in a single session without
  a clear plan that covers all of them.
- A request references a security standard, CVE, or benchmark that you cannot verify
  from the existing files in this repository.
- A request would result in a file containing a real credential, account ID, or
  environment-specific value.
- The scope of a change is ambiguous and you cannot determine from context which
  toolkit or files are affected.

Do not make assumptions to avoid asking. A short clarifying question costs far less
than a change that needs to be reverted.
