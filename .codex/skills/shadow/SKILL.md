---
name: shadow
description: Replace a method, class, or package with a stub or unload it while preserving backups and shadow state. Use only for explicit `$shadow` requests.
---

# Shadow code

Treat the remaining user request after `$shadow` as its arguments. Follow [the detailed workflow](../../../.claude/commands/shadow.md), replacing `$ARGUMENTS` with those arguments and `/extend` and `/rollback` with `$extend` and `$rollback`. This skill changes source files; retain the workflow's backup, scope, and validation safeguards.
