---
name: reflect
description: Review recent Codebase Agent work and maintain the repository knowledge base and skill suggestions. Use for `$reflect` requests.
---

# Reflect on repository knowledge

Treat the remaining user request after `$reflect` as optional focus. Follow [the detailed workflow](../../../.claude/commands/reflect.md), replacing `$ARGUMENTS` with that focus and `/gen-command` with `$gen-command`.

When `AGENT-DEVELOPMENT.md` exists at the repository root, use the workflow's self-maintenance mode: write only `.agent/self/reflect.md`; do not change deployable templates or ordinary `.agent/refs/` files.
