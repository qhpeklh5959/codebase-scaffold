---
name: review
description: Audit current changes against the repository's established .agent/refs conventions and patterns, and report every finding with its refs citation. Use for $review requests. Read-only; fixes stay with $extend and knowledge updates with $reflect.
---

# Review changes against repository refs

Treat the remaining user request after `$review` as the optional path or commit range; when absent, review the current working tree. Follow [the detailed workflow](../../../.claude/commands/review.md), replacing `$ARGUMENTS` with that request and every `/name` reference (`/bootstrap`, `/extend`, `/rollback`, `/reflect`, `/gen-command`) with its `$name` form.

This is a read-only audit: do not edit source, refs, or shadow state. Fixes belong to `$extend`; knowledge updates belong to `$reflect`.
