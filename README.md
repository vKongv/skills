# Agent Skills

Personal [Agent Skills](https://agentskills.io) for Cursor, Codex, Claude Code, and other agents that support the open skills ecosystem.

Company workflows (claims, JLFM building migration) live in the private Pebble Tech repo [`pebble-tech/skills`](https://github.com/pebble-tech/skills) — not in this catalog.

[![skills.sh](https://skills.sh/b/vkongv/skills)](https://skills.sh/vkongv/skills)

## Install

Install interactively (pick skills and agents):

```bash
npx skills add vkongv/skills
```

Install everything globally for Cursor:

```bash
npx skills add vkongv/skills --all -g -a cursor -y
```

Install one skill:

```bash
npx skills add vkongv/skills --skill let-cursor-cook -g -a cursor -y
npx skills add vkongv/skills --skill orchestrate -g -a cursor -y
```

List available skills without installing:

```bash
npx skills add vkongv/skills --list
```

## Skills catalog

| Skill | Description | Attach |
|-------|-------------|--------|
| [let-cursor-cook](./skills/let-cursor-cook/SKILL.md) | Orchestrate Cursor CLI agents from an external harness (Codex, Claude Code, etc.). Manager owns requirements; Cursor agents do research, implementation, debugging, and review. | Auto |
| [orchestrate](./skills/orchestrate/SKILL.md) | PM-style delivery through research, proposal approval, requirement grilling, task-ledger contract, implementor/reviewer/verifier subagents, PRs, CI, and review-thread closure. Orchestrator edits only docs/skills, not product code. | Manual (`disable-model-invocation`) |

## Repository layout

```text
skills/
  <skill-name>/
    SKILL.md
    references/   # optional
    scripts/      # optional
```

Each skill is a folder with a `SKILL.md` file (YAML frontmatter + markdown body). Install with `npx skills add`; agents load skills on demand.

## License

MIT — see [LICENSE](./LICENSE).