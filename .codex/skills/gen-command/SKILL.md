---
name: gen-command
description: Suggest or create a repository-specific Codex skill from a repeated workflow. Use for `$gen-command` requests.
---

# Generate a repository-specific skill

Treat the remaining user request after `$gen-command` as its arguments. Follow [the detailed workflow](../../../.claude/commands/gen-command.md), with these required Codex substitutions:

- `$ARGUMENTS` means the remaining user request.
- Inspect `.codex/skills/`, not `.claude/commands/`, when detecting existing capabilities.
- Create a skill directory at `.codex/skills/{kebab-case-name}/SKILL.md`, not `.claude/commands/{name}.md`.
- The new `SKILL.md` needs YAML frontmatter with `name` and a discriminating `description`, followed by the generated workflow. Use `$skill-name` in its usage examples rather than `/{name}`.
- Keep the suggestion lifecycle, refs dependencies, confirmation-before-writing, and `failures.md` maintenance from the detailed workflow unchanged.
