---
name: bootstrap
description: Analyze a codebase and initialize its Codebase Agent knowledge base. Use when the user asks to bootstrap, initialize, or build `.agent/refs` for a repository.
---

# Bootstrap codebase knowledge

Treat the remaining user request after `$bootstrap` as the optional repository path; otherwise use the current repository. Follow the complete workflow in [the legacy workflow](../../../.claude/commands/bootstrap.md), with these Codex mappings:

- `$ARGUMENTS` means the remaining user request.
- `/trace` and `/gen-command` mean `$trace` and `$gen-command`.
- Create and update the same `.agent/refs/` files. Preserve the 200-line splitting rule in `AGENTS.md`.
- Suggestions created by this workflow are Codex skills, not Claude commands.
- Generate the target repository's `CODEBASE.md` with the work-versus-refs guidance required by the workflow; never reuse this scaffold repository's generated refs as the target's content.
