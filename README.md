# agent-skills

A collection of **agent skills** for AI coding agents (Claude Code, Cursor, Codex, Grok Build, Windsurf, and similar tools).

Skills are portable instruction packages (`SKILL.md`) that teach an orchestrator agent *when* and *how* to run a specialized workflow. Each skill is self-contained; this README is only the catalog.

## Skills

| Skill | Path | Description |
|-------|------|-------------|
| **grok-implementer** | [`grok-implementer/`](./grok-implementer/) | Route hands-on implementation to **Grok headless CLI** — only when the user explicitly asks. Orchestrator freezes a work-order; Grok implements; orchestrator reviews the diff and re-runs proof. |

Add a skill → add a row here and a directory with `SKILL.md`. Full contracts, flags, and prompts live in each skill’s own docs, not in this file.

## Repository layout

```text
agent-skills/
├── README.md                 # this catalog
├── LICENSE
├── .gitignore
└── <skill-name>/
    └── SKILL.md              # frontmatter + full skill body
```

Optional per skill: `scripts/`, `references/`, examples — whatever that skill needs. Keep the skill name aligned with the folder name.

## Installing a skill

How you install depends on the agent:

| Agent | Typical approach |
|-------|------------------|
| **Claude Code** | Copy or symlink the skill folder into a skills directory your setup discovers (project, user, or plugin skills path). |
| **Grok Build** | Place under a skills path Grok loads (e.g. user/project skills), or follow Grok’s skill/plugin docs. |
| **Cursor / Codex / others** | Point the agent at this repo or copy the relevant skill folder into that product’s skill/rules location. |

Minimal install of one skill:

```bash
git clone https://github.com/nguyentan2808/agent-skills.git
# then copy or symlink <skill-name>/ into your agent’s skills directory
```

Refer to your agent’s documentation for the exact skills path and discovery rules. Install only the skills you need — folders are independent.

## Using a skill

1. Install the skill folder (see above) and meet any prerequisites listed in that skill’s `SKILL.md`.
2. Explicitly ask the orchestrator to use it when you want it (e.g. “use grok-implementer …”).
3. Follow the workflow defined in that skill; do not expect silent auto-activation unless the skill says otherwise.

## Adding a skill

1. Create `<skill-name>/SKILL.md` with YAML frontmatter at least:

   ```yaml
   ---
   name: skill-name
   description: >
     When to use this skill (triggers) and what it does in one short blurb.
   ---
   ```

2. Document in the body: who runs it, when to use / skip, invoke steps, failure handling, and any auth/binary prerequisites.
3. Add one row to the **Skills** table above.
4. Prefer concrete recipes over vague advice; match the operational tone of existing skills.

## License

[MIT](./LICENSE) © 2026 Nguyen Tan
