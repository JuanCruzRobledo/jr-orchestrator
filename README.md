# jr-orchestrator

Thin orchestrator skill for the SDD/OpenSpec project foundation flow.

## What it does

`jr-orchestrator` is a **thin** orchestrator. It does not generate any artifact itself — it dispatches each foundation phase to its dedicated sub-skill in order:

1. `openspec init` — scaffold the OpenSpec workspace
2. `kb-creator` — build `knowledge-base/` (discovery + canonical docs)
3. `roadmap-generator` — build `CHANGES.md` from the KB
4. `find-skill` — recommend agent skills for the detected stack
5. `agent-instruction` — generate `CLAUDE.md` / `AGENTS.md`

State is shared across phases via `.jr-orchestrator-state.json` (schema v2, frozen contract).

## Triggers

| Command | Effect |
|---|---|
| `/jr-orchestrator:init` | Full foundation flow |
| `/jr-orchestrator:kb` | Only `kb-creator` phase |
| `/jr-orchestrator:rules` | Only `agent-instruction` phase |
| `/jr-orchestrator:openspec` | Only `openspec init` |
| `/jr-orchestrator:find-skill` | Only `find-skill` phase |
| `/jr-orchestrator:devops` | Optional devops scaffolding (full mode) |

## Install

```bash
npx skills add https://github.com/JuanCruzRobledo/jr-orchestrator
```

## Architecture

This skill is part of the **JR Stack** foundation. The sub-skills (`kb-creator`, `roadmap-generator`, `find-skill`, `agent-instruction`) are independent skills, each in its own repo. `jr-orchestrator` does **NOT** embed them — it dispatches via the `Skill` tool and falls back gracefully if a sub-skill is missing.

For the full architecture and frozen state contract, see [SKILL.md](SKILL.md).

## License

MIT
