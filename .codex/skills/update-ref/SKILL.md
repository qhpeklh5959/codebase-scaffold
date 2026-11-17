---
name: update-ref
description: Fast-forward the current branch from its upstream remote, then audit the pulled changes, update the repository's .agent/refs knowledge base, and suggest trace targets. Use for $update-ref requests. Local-only; never pushes, forces, or auto-stashes.
---

# Update refs from the remote branch

Treat the remaining user request after `$update-ref` as the optional `<remote>/<branch>` and `--check` flag. Follow [the detailed workflow](../../../.claude/commands/update-ref.md), replacing `$ARGUMENTS` with that request and every `/name` reference (`/bootstrap`, `/reflect`, `/trace`, `/gen-command`) with its `$name` form.

This command is local-only: it never pushes, never forces, and never auto-stashes. Stop when the worktree is dirty or the branch has diverged.
