## What this PR adds or changes

[One paragraph. What is the new capability, fix, or improvement?]

## Motivation

[Why does this belong in the repository? What problem does it solve for
a DevSecOps engineer?]

## Toolkit / folder affected

[Name of the folder this PR touches, e.g. `iac-security-review` or `new: k8s-audit`]

## Checklist

- [ ] Each new skill has a `SKILL.md` with complete frontmatter (name, description,
      context, argument-hint, allowed-tools)
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