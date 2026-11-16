# DSH skill bridge

DeepSeek Harness (DSH) discovers local skills only from these roots (one level deep, `<root>/<name>/SKILL.md` or `<root>/<name>.md`):

- `<projectRoot>/.dsh/skills` (project)
- `<projectRoot>/.agents/skills` (project)
- `~/.dsh/skills` (user)
- `~/.agents/skills` (user)

It does **not** scan `.codex/skills`, so the Codex skills would be invisible to DSH. The skill format itself is identical (a directory bundle with `SKILL.md` plus `name`/`description` frontmatter), so no content conversion is needed.

`.dsh/skills` is therefore a symlink to `../.codex/skills`:

- `.codex/skills` stays the single source of truth for Codex.
- DSH sees the same skills, and new skills created by `$gen-command` under `.codex/skills` appear automatically.
- The `../../../.claude/commands/*.md` links in each `SKILL.md` still resolve, because the symlink sits at the same depth as `.codex/skills`.

On platforms where symlinks are unavailable (e.g. Windows without developer mode), replace the link with a junction/copy:

```powershell
mklink /D .dsh\skills ..\.codex\skills
```
