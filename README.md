# sw6-skills

Community-maintained Claude skill files for Shopware 6 client project development.

## What is this?

Each `SKILL.md` file is a structured instruction set that Claude reads before
tackling a task. They encode accumulated best practices, gotchas, and patterns
for Shopware 6 and its common ecosystem tools.

These skills are for **developing client projects with Shopware 6** — not for
contributing to Shopware 6 core itself.

## Structure

```
skills/
├── shopware6/         # Core SW6: plugins, DAL, entities, events, CLI, search, storefront, API
└── ddev/              # DDEV local development: config, services, hot-reload, commands
```

## Usage

### Claude Code (CLI / IDE)
Place skill files where Claude Code can read them directly, or reference them
in your project's `CLAUDE.md`:
```
For Shopware 6 tasks, read and follow: skills/shopware6/SKILL.md
For DDEV tasks, read and follow: skills/ddev/SKILL.md
```

### claude.ai custom instructions
Add to your custom instructions:
> "For any Shopware 6 task, first fetch and follow:
> https://raw.githubusercontent.com/vanWittlaer/sw6-skills/main/skills/shopware6/SKILL.md"

### API / Claude for Teams
Paste the skill content directly into the system prompt for project conversations.

## Versioning

Skills target current Shopware versions:
- `sw-6.6.x` — Shopware 6.6 compatible
- `sw-6.7.x` — Shopware 6.7 compatible

## Contributing

PRs welcome. Keep skills factual, concise, and example-driven.
Avoid opinion where Shopware's own docs are authoritative — link instead.

## Maintainers

- [@vanWittlaer](https://github.com/vanWittlaer)
