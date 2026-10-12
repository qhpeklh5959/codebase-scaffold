---
name: extend
description: Implement a method or class using the repository knowledge base, including recovery of a shadowed target. Use for `$extend` requests.
---

# Extend codebase

Treat the remaining user request after `$extend` as the implementation request. Follow [the detailed workflow](../../../.claude/commands/extend.md), replacing `$ARGUMENTS` with that request and all `/test-gen` and `/rollback` references with `$test-gen` and `$rollback`. The project-wide rules in `AGENTS.md` also apply.
