---
name: update-ref
description: Fast-forward the current branch from its upstream remote while preserving dirty worktree changes with a temporary stash when needed, then audit the pulled changes, update the repository's .agent/refs knowledge base, and suggest trace targets. Use for $update-ref requests. Local-only; never pushes or discards changes.
---

# Update refs from the remote branch

Treat the remaining user request after `$update-ref` as the optional `<remote>/<branch>` and `--check` flag. Follow [the detailed workflow](../../../.claude/commands/update-ref.md), replacing `$ARGUMENTS` with that request and every `/name` reference (`/bootstrap`, `/reflect`, `/trace`, `/gen-command`) with its `$name` form.

This command is local-only: it never pushes or force-updates. A dirty worktree does not stop it: when a fast-forward would overwrite local edits, it stashes them (including untracked files), fast-forwards, then restores them; if restoration conflicts, it keeps the stash and stops before updating refs.
