     ██████╗ ███████╗██╗   ██╗███████╗███████╗ ██████╗  ██████╗ ██████╗ ███████╗
     ██╔══██╗██╔════╝██║   ██║██╔════╝██╔════╝██╔════╝ ██╔═══██╗██╔══██╗██╔════╝
     ██║  ██║█████╗  ██║   ██║███████╗█████╗  ██║      ██║   ██║██████╔╝███████╗
     ██║  ██║██╔══╝  ╚██╗ ██╔╝╚════██║██╔══╝  ██║      ██║   ██║██╔═══╝ ╚════██║
     ██████╔╝███████╗ ╚████╔╝ ███████║███████╗╚██████╗ ╚██████╔╝██║     ███████║
     ╚═════╝ ╚══════╝  ╚═══╝  ╚══════╝╚══════╝ ╚═════╝  ╚═════╝ ╚═╝     ╚══════╝

                          ██╗    ██╗██╗████████╗██╗  ██╗
                          ██║    ██║██║╚══██╔══╝██║  ██║
                          ██║ █╗ ██║██║   ██║   ███████║
                          ██║███╗██║██║   ██║   ██╔══██║
                          ╚███╔███╔╝██║   ██║   ██║  ██║
                           ╚══╝╚══╝ ╚═╝   ╚═╝   ╚═╝  ╚═╝

                     ██████╗██╗      █████╗ ██╗   ██╗██████╗ ███████╗
                    ██╔════╝██║     ██╔══██╗██║   ██║██╔══██╗██╔════╝
                    ██║     ██║     ███████║██║   ██║██║  ██║█████╗
                    ██║     ██║     ██╔══██║██║   ██║██║  ██║██╔══╝
                    ╚██████╗███████╗██║  ██║╚██████╔╝██████╔╝███████╗
                     ╚═════╝╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚══════╝

---

A community collection of Claude Code skills, CLAUDE.md configurations,
custom commands, and reference materials for DevSecOps engineers.

---

## What this repository is

This repository is a growing, community-maintained toolkit that helps DevSecOps
engineers use Claude and Claude Code to improve the security of infrastructure,
deployed workloads, and development pipelines.

Everything here is practical. Each item in the collection — whether a Claude Code
skill, a CLAUDE.md configuration, a set of reference catalogs, or a custom slash
command — was built to solve a real problem that comes up in security engineering
work. The emphasis is on tools that produce actionable output, not on demos or
one-off scripts.

**What you will find here:**

- **Skills** — Claude Code `SKILL.md` files that give Claude structured,
  expert workflows for specific security tasks (IaC review, threat modeling,
  vulnerability triage, and more).
- **CLAUDE.md configurations** — Ready-to-use project and directory-level
  instructions that tune Claude's behaviour for specific security contexts
  (e.g. a CloudFormation reviewer, a Kubernetes manifest auditor).
- **Custom commands** — Slash commands (`.claude/commands/`) for common
  DevSecOps workflows that teams can drop into any project.
- **Reference catalogs** — Curated threat catalogs, benchmark mappings, and
  check lists used by the skills as reference material.

**What you will not find here:**

This repository does not ship runnable infrastructure or deploy anything to cloud
accounts. Every tool here is a read-only analysis and reasoning aid. Nothing modifies your code, your cloud account, or your pipeline without your explicit instruction.

---

## Contents

- [Philosophy](#philosophy)
- [Repository structure](#repository-structure)
- [Available toolkits](#available-toolkits)
- [How to use this repository](#how-to-use-this-repository)
- [Understanding the building blocks](#understanding-the-building-blocks)
- [Contributing](#contributing)
- [Adding a new toolkit](#adding-a-new-toolkit)
- [Contribution standards](#contribution-standards)
- [Pull request process](#pull-request-process)
- [Code of conduct](#code-of-conduct)

---

## Philosophy

Three principles shape every decision in this repository.

**Staged over monolithic.** Large, single-pass prompts that try to do everything
at once produce noisy output, degrade at context boundaries, and are hard to
debug when they go wrong. The tools here decompose complex security workflows
into focused stages — each stage reads the output of the previous, does one
thing well, and writes structured output for the next. This makes the pipeline
auditable and the individual stages reusable.

**Targeted reading over full ingestion.** Loading every file in a repository
into a single context window wastes tokens and degrades recall for content
loaded early. The tools here use `grep`, `glob`, and structural discovery
before reading anything in full. Files are read only when a specific question
requires their content. Every tool in this collection has a token efficiency
contract — a statement of what it will and will not load, and why.

**Validation over raw output.** An unvalidated finding list is not a security
report — it is a triage queue. The tools here treat initial findings as
candidates that require a second pass: re-verifying the evidence in source,
checking for compensating controls, testing the exploitation path, and adjusting
severity to reflect the deployment context. The final output of any workflow
should be something a security engineer can act on directly.

---

## Repository structure

Every toolkit or standalone tool lives in its own top-level folder. That folder
is self-contained: it includes everything needed to understand and use the tool
without reading anything else in the repository.

```
devsecops-with-claude/
│
├── README.md                        ← you are here
│
├── iac-security-review/             ← IaC template security review pipeline
│   ├── README.md                    ← full usage guide for this toolkit
│   └── .claude/
│       ├── iac-review-config.yml    ← shared config template (copy to your project)
│       └── skills/
│           ├── IAC_MAP/             ← Stage 1: project structure mapper
│           ├── IAC_THREAT_MODEL/    ← Stage 2: STRIDE threat modeling
│           ├── IAC_AUDIT/           ← Stage 3: CIS + threat-model-based audit
│           └── IAC_REASSESS/        ← Stage 4: finding validation and triage
│
└── [future-toolkit-name]/           ← next contribution goes here
    ├── README.md
    └── .claude/
        └── ...
```

**Rules that keep the structure clean:**

1. One folder per toolkit or standalone tool. No shared folders between unrelated items.
2. Every folder at the root level must have a `README.md` that explains the tool
   independently of any other file in the repository.
3. Claude-specific files (skills, commands, CLAUDE.md configs, rules) live inside
   a `.claude/` subdirectory within the toolkit folder. This mirrors exactly how
   they would be installed in a real project.
4. Reference files (threat catalogs, benchmark extracts, check lists) that a skill
   reads at runtime live inside the skill's own folder or a named subfolder
   (`threats-list/`, `cis-benchmarks/`, etc.) — not at the repository root or
   in a shared `assets/` directory.
5. Config templates that users copy into their projects are named clearly
   (e.g. `iac-review-config.yml`) and include inline comments explaining every field.

---

## Available resources/toolkits

→ See [`AVAILABLE_RESOURCES.md`](AVAILABLE_RESOURCES.md)

---

## How to use this repository

### If you want to use an existing toolkit

1. Find the toolkit folder you need and read its `README.md` first. The README
   explains prerequisites, installation steps, and every command the toolkit provides.

2. Copy the `.claude/` directory from the toolkit folder into the root of your project:
   ```bash
   cp -r iac-security-review/.claude/ /path/to/your-project/.claude/
   ```

3. If the toolkit includes a config template (e.g. `iac-review-config.yml`), copy
   it into your project root and fill in your project's values before running anything.

4. Open Claude Code in your project directory and run the skill commands as described
   in the toolkit README.

Each toolkit's `.claude/` folder is designed to be dropped directly into a project.
Nothing in this repository assumes a specific directory layout beyond what Claude Code
itself requires.

### If you want to contribute a new toolkit or improvement

Read the [Contributing](#contributing) section below. The short version: fork the
repository, create a branch, open a pull request against `main`, and assign the
maintainer as reviewer. Nothing merges without review.

---

## Understanding the building blocks

This section explains the Claude Code primitives that the tools in this repository
are built from. If you want to understand why things are structured the way they
are, or if you want to build your own additions, this is the right starting point.

### Skills (`SKILL.md`)

A skill is a reusable, on-demand workflow for Claude Code. It lives in
`.claude/skills/<SKILL_NAME>/SKILL.md` inside a project. When a user types
the skill's name as a slash command, Claude Code reads the `SKILL.md` file
and executes the workflow it describes.

Skills differ from CLAUDE.md instructions in one important way: CLAUDE.md content
is loaded automatically in every session, while skills are invoked explicitly on
demand. Use a skill for any workflow that is: specific to one type of task,
computationally expensive, or verbose enough to clutter the main conversation.

**Anatomy of a `SKILL.md` file:**

```markdown
---
name: my-skill                          # slash command name: /my-skill
description: >-                         # shown in skill picker; used for routing
  One-paragraph description of what this skill does, when to use it,
  and what it produces. Be specific — this is what Claude reads to decide
  whether to invoke this skill.
context: fork                           # runs in isolated sub-agent (recommended
                                        # for any skill that produces verbose output)
argument-hint: "<target> [--option]"    # shown to user when they type /my-skill
allowed-tools:                          # explicit tool allowlist (security boundary)
  - Read
  - Glob
  - Grep
  - Write
  - Task                                # enables sub-agent fan-out
  - Bash(find:*)                        # Bash is allowlisted per-command
  - Bash(grep:*)
---

# /my-skill

[Skill body: step-by-step instructions Claude follows when the skill is invoked]
```

**Key frontmatter fields:**

| Field | Purpose |
|-------|---------|
| `name` | The slash command name. Must be unique within a project. |
| `description` | Critical for routing — Claude reads this to decide whether to invoke the skill. Be specific about inputs, outputs, and trigger phrases. |
| `context: fork` | Runs the skill in an isolated sub-agent context. Output does not pollute the main conversation. Use for any skill that reads many files or produces long output. |
| `argument-hint` | Displayed when the user invokes the skill with no arguments. Describes expected arguments. |
| `allowed-tools` | Explicit allowlist of tools the skill can use. This is a security boundary — list only what the skill genuinely needs. Bash commands are allowlisted per-command using `Bash(command:*)` syntax. |

**Token efficiency rules for skills:**

- Use `Glob` and `Grep` before `Read`. Discovery should never require reading
  file contents.
- When a file must be read, use `head` or targeted `grep` before reading in full.
- Use `Task` (sub-agent fan-out) to parallelize work across multiple files rather
  than loading them sequentially into one context.
- Document the token efficiency contract in the skill body: what the skill will
  and will not load, and why.

### Custom commands (`.claude/commands/`)

A custom slash command is a simpler, lighter alternative to a skill. It lives in
`.claude/commands/<command-name>.md`. When invoked, Claude reads the file and
follows the instructions in it. Commands do not have frontmatter configuration —
they are plain Markdown instruction files.

Use a command (rather than a skill) for short, focused tasks that do not need
sub-agent fan-out, tool restrictions, or context isolation. A code review
checklist, a PR description generator, or a commit message formatter are good
candidates for commands.

```
.claude/
└── commands/
    ├── security-review.md      ← /security-review
    ├── pr-description.md       ← /pr-description
    └── commit-message.md       ← /commit-message
```

Commands stored in `.claude/commands/` are project-scoped and shared via version
control. Commands stored in `~/.claude/commands/` are personal and not shared.

### CLAUDE.md configurations

`CLAUDE.md` files are instruction documents that Claude Code loads automatically
at the start of every session. They set persistent context, conventions, and
constraints that apply throughout the session without needing to be invoked
explicitly.

The configuration follows a three-level hierarchy:

```
~/.claude/CLAUDE.md              # user-level: personal preferences, not version-controlled
<project-root>/.claude/CLAUDE.md # project-level: shared team conventions
<subdirectory>/CLAUDE.md         # directory-level: scoped to one area of the codebase
```

Lower levels inherit from higher levels and can add to or override them. Use
`@import path/to/file.md` to reference external files and keep CLAUDE.md modular.

**When to use CLAUDE.md vs. a skill:**

| Use CLAUDE.md when... | Use a skill when... |
|----------------------|---------------------|
| The instruction applies to every session automatically | The workflow is invoked on demand for a specific task |
| The context is short and persistent (conventions, constraints) | The workflow is long, verbose, or produces structured output |
| You want Claude to always apply a behaviour without being asked | You want Claude to run a workflow only when explicitly requested |

### Path-specific rules (`.claude/rules/`)

Rules files in `.claude/rules/` use YAML frontmatter to scope instructions to
specific file paths. Claude loads a rules file only when the user is working on
a file that matches the glob pattern.

```markdown
---
paths:
  - "terraform/**/*.tf"
  - "cloudformation/**/*.yaml"
---

When reviewing IaC files, always check for hardcoded secrets, open security
groups, and missing encryption settings before anything else.
```

This avoids loading IaC-specific instructions when the user is working on a Python
file, and Python-specific instructions when they are working on a Terraform module.
Use path-specific rules for context that is genuinely scoped to a file type or
directory — not for universal conventions, which belong in the project CLAUDE.md.

---

## Contributing

Contributions are welcome. The goal is a high-quality collection of tools that
DevSecOps engineers can trust — which means every addition goes through review
before it lands in `main`.

### Before you start

- Check open issues and open pull requests to make sure you are not duplicating
  work already in progress.
- If you are planning a large addition (a new multi-stage toolkit, a significant
  refactor of an existing one), open an issue first to discuss the design. This
  avoids investing significant effort in a direction that does not fit the
  repository's scope.
- Read the [contribution standards](#contribution-standards) below before writing
  anything. They describe the quality bar every addition must meet.

### Workflow

```
1. Fork the repository to your own GitHub account.

2. Clone your fork locally:
   git clone https://github.com/<your-username>/devsecops-with-claude.git
   cd devsecops-with-claude

3. Create a branch for your work.
   Use a descriptive name that identifies what you are adding or fixing:

   git checkout -b add/k8s-manifest-audit-skill
   git checkout -b fix/iac-map-exclusion-handling
   git checkout -b improve/iac-audit-cis-section3

   Branch naming conventions:
     add/<short-description>      — new toolkit, skill, command, or reference file
     fix/<short-description>      — bug fix or correction in existing content
     improve/<short-description>  — improvement to existing content
     docs/<short-description>     — documentation-only change

4. Make your changes following the contribution standards below.

5. Commit your changes with clear, descriptive commit messages.
   Each commit should represent one logical change:

   git add .
   git commit -m "add: IAC_MAP skill stage 1 project mapper"
   git commit -m "add: CIS benchmark section 1 IAM reference file"
   git commit -m "docs: add iac-security-review README with usage guide"

   Commit message format:
     <type>: <short description>

   Types: add | fix | improve | docs | refactor | remove

6. Push your branch to your fork:
   git push origin add/k8s-manifest-audit-skill

7. Open a pull request against the main branch of this repository.
   Title: follow the same format as commit messages.
   Body: use the pull request template below.

8. Assign the maintainer as reviewer. Do not merge your own pull request.
   The PR will be reviewed and either approved, or returned with feedback.

9. Address any review comments, push additional commits to the same branch,
   and mark resolved comments as resolved.

10. Once approved, the maintainer merges the PR. Do not squash or rebase
    without being asked — the commit history is kept clean by the branch
    naming and commit message conventions above.
```

### Pull request template

When you open a PR, include the following in the description:

```markdown
## What this PR adds or changes

[One paragraph. What is the new capability, fix, or improvement?]

## Motivation

[Why does this belong in the repository? What problem does it solve for
a DevSecOps engineer?]

## Toolkit / folder affected

[Name of the folder this PR touches, e.g. `iac-security-review` or `new: k8s-audit`]

## Checklist

- [ ] Each new skill has a `SKILL.md` with complete frontmatter
      (name, description, context, argument-hint, allowed-tools)
- [ ] Every new folder at the repository root has its own `README.md`
- [ ] The README explains: what it is, the philosophy behind it, how to install,
      how to use it, and what the benefits are
- [ ] Config templates include inline comments for every field
- [ ] Reference files (threat catalogs, benchmark files) are in a named
      subfolder inside the relevant skill directory
- [ ] No hardcoded credentials, account IDs, ARNs, or environment-specific values
      in any file
- [ ] All skills are read-only with respect to the user's infrastructure
      (no writes to target directories other than a designated output folder)
- [ ] The token efficiency contract is documented in every skill body
      (what the skill reads, what it skips, and why)
- [ ] Tested locally: the skill or command produces correct output when invoked
      in Claude Code against a sample project

## Notes for the reviewer

[Anything that would help the reviewer understand the design decisions,
scope limitations, or areas where you are uncertain.]
```

---

## Adding a new toolkit

A toolkit is a self-contained set of Claude Code tools for one security domain or workflow.
Follow this structure when adding one:

```
<your-toolkit-name>/
├── README.md                    ← required; see requirements below
└── .claude/
    ├── <config-template>.yml    ← optional; include if the toolkit needs
    │                               project-specific configuration
    ├── skills/
    │   └── <SKILL_NAME>/
    │       ├── SKILL.md         ← skill definition with frontmatter
    │       └── <references>/    ← named subfolder for reference files
    │           └── *.md
    ├── commands/
    │   └── <command-name>.md    ← optional slash commands
    └── rules/
        └── <rule-name>.md       ← optional path-scoped rules
```

### README requirements for new toolkits

Every toolkit folder must have a `README.md` that covers:

1. **Overview** — what the toolkit does, in two to four sentences. Someone who
   has never heard of it should understand its purpose within 30 seconds.

2. **Philosophy** — why the toolkit is designed the way it is. What tradeoffs
   were made? What problem does the staging or structure solve that a simpler
   approach would not? This section helps future contributors understand the
   intent so they extend rather than accidentally undermine it.

3. **Benefits** — concrete improvements a DevSecOps engineer should expect.
   Be honest about scope and limitations. Do not overstate what Claude can do.

4. **Standards and checks covered** — if the toolkit implements or references
   a security standard (CIS, NIST, OWASP, STRIDE, CVSSv3.1, etc.), list which
   controls or categories are covered and which are explicitly excluded and why.

5. **Installation** — exact steps to copy the toolkit into a project and
   configure it. Include any prerequisites.

6. **Usage guide** — every command with its arguments, what it does, what it
   produces, and example invocations for the most common scenarios.

7. **Output file reference** — if the toolkit produces files, list them all:
   filename, which stage produces it, what it contains.

8. **FAQ** — at minimum, address: whether all stages/commands are required,
   what formats or providers are supported, what is explicitly out of scope,
   and whether the toolkit modifies any user files.

### Skill quality standards

Every `SKILL.md` must:

- Have complete frontmatter: `name`, `description`, `context`, `argument-hint`,
  `allowed-tools`. No field may be absent or left as a placeholder.
- Include a token efficiency contract near the top of the skill body. State
  explicitly what the skill will read in full, what it will only grep/scan,
  and what it will skip entirely.
- Document every argument in an `## Arguments` section using a consistent format:
  `- <name> (required|optional) — description`.
- Describe its output contract: what files it writes, where, and in what format.
- End with a `## Constraints` section listing what the skill will never do
  (e.g. "never executes target code", "never writes to files outside the
  output directory", "never makes network requests").
- Be read-only with respect to the user's project files. The only permitted
  writes are to a designated output directory (e.g. `iac-review-results/`).

### Reference file standards

Reference files (threat catalogs, benchmark extracts, check lists) that a skill
reads at runtime must:

- Live inside the skill's own folder or a clearly named subfolder, not in a
  shared location at the repository root.
- Begin with a comment block identifying the source, the version or date, and
  the scope of what is included and excluded.
- Use a consistent entry format throughout the file. If a table is used,
  all columns must be present for every row.
- Not include content that cannot be redistributed. Benchmark and standard
  bodies often permit reproduction for non-commercial, attributed use — check
  the specific license before including verbatim excerpts.

---

## Contribution standards

These apply to all contributions regardless of size.

**Security first.** Any tool added here is used by engineers in production
security contexts. Accuracy matters more than coverage. If a check produces
false positives, or a description of a vulnerability is incorrect, it does
real harm — it trains engineers to ignore findings. When in doubt, be more
conservative, not less.

**No fabricated content.** Skills that reference security standards must
reference those standards accurately. Do not invent CVSS scores, CIS control
numbers, STRIDE categories, or CVE references. If you are unsure whether a
claim is accurate, include a source or leave it out.

**Read-only by default.** Tools that write to user projects must write only
to a clearly designated output directory and must document this explicitly.
No tool should ever write to a path outside its designated output location,
modify the user's IaC templates, or make API calls to cloud infrastructure.

**No credentials, no hardcoded values.** No file in this repository may
contain AWS account IDs, IAM role ARNs, IP addresses, hostnames, API keys,
or any other value that is specific to a real environment. Config templates
must use placeholder values with comments instructing the user to replace them.

**One concern per commit.** Commits that mix unrelated changes (e.g. a new
skill plus a fix to an existing README plus a new reference file) make review
harder and history harder to read. Split unrelated changes into separate commits.

**English only.** All documentation, skill bodies, comments, and reference
files must be in English to keep review manageable and the audience broad.

---

## Pull request review process

All pull requests follow this process:

1. **Automated checks** (if configured): linting, link checking.
2. **Maintainer review**: the maintainer reviews the PR for correctness, quality,
   and fit with the repository's scope and standards.
3. **Feedback round**: if changes are requested, the contributor addresses them and
   pushes new commits to the same branch. Do not open a new PR.
4. **Approval**: when all feedback is addressed and the reviewer is satisfied,
   they approve the PR.
5. **Merge**: the maintainer merges the PR into `main`. Contributors do not
   merge their own PRs.

PRs that do not follow the checklist, are missing a README, contain fabricated
security content, or include credentials or hardcoded environment values will be
closed without merge and asked to be resubmitted after correction.

There is no SLA on review time, but the goal is to respond to all new PRs
within one week. If a PR has been open for two weeks with no response, ping the
reviewer in the PR comments.

---

## Code of conduct

This repository is a professional resource for security engineers. Contributions
and discussions should reflect that.

- Be accurate. Security tools that lie are worse than no tools at all.
- Be constructive. Review comments should explain what is wrong and suggest how
  to fix it, not just reject.
- Be respectful. Engineers contribute here in their own time. Treat that time
  with respect.
- Disagree with arguments, not people.

Issues or PRs that are disrespectful, off-topic, or primarily promotional will
be closed.

---

## Maintainer

Pull requests should be assigned to the repository maintainer for review.
Direct questions about scope, design, or contribution process to the maintainer
via a GitHub issue rather than in PR comments.

---

*This repository does not represent the views of Anthropic. It is an independent,
community-maintained collection of tools that use the Claude API and Claude Code.
Claude and Claude Code are products of Anthropic, PBC.*