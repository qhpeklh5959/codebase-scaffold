---
name: rollback
description: Restore the original backup of a shadowed method, class, or package, including affected callers. Use only for explicit `$rollback` requests.
---

# Roll back a shadow

Treat the remaining user request after `$rollback` as its arguments. Follow [the detailed workflow](../../../.claude/commands/rollback.md), replacing `/shadow` and `/extend` with `$shadow` and `$extend`. Obtain the workflow's explicit confirmation immediately before overwriting or recreating files.
