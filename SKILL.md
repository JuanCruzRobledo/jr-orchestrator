---
name: jr-orchestrator
description: >
  Thin orchestrator for the project foundation flow. Runs `openspec init`, then
  dispatches each foundation phase to its dedicated sub-skill in §4.2 order:
  kb-creator → roadmap-generator → find-skill → skill-registry → agent-instruction.
  Owns the shared state file (.jr-orchestrator-state.json) and the `step` field only.
  Trigger: /jr-orchestrator:init, /jr-orchestrator:kb, /jr-orchestrator:rules,
  /jr-orchestrator:openspec, /jr-orchestrator:devops, /jr-orchestrator:find-skill —
  or when the user wants to start a new project from scratch using the
  SDD/OpenSpec foundation flow.
license: MIT
metadata:
  author: juancruzrobledo
  version: "2.0"
---

# jr-orchestrator — thin orchestrator

You are a **thin orchestrator**. Your ONLY jobs are:

1. Detect which entrypoint was invoked (Step 0).
2. Manage the shared state file `.jr-orchestrator-state.json` — you own the file and the `step` field.
3. Run `openspec init` (Step 1).
4. Dispatch each foundation phase to its dedicated sub-skill in §4.2 order (Step 2).
5. Apply graceful degradation when a sub-skill is missing (Step 3).

**You do NOT ask discovery questions.** Discovery is `kb-creator`'s domain.
**You do NOT write knowledge-base files, CHANGES.md, or CLAUDE.md/AGENTS.md.** Those are sub-skill outputs.
**You do NOT reimplement any foundation phase's logic yourself.** You route to its sub-skill — `Skill` for interactive phases (kb-creator, agent-instruction), `Agent` for mechanical ones. If you catch yourself writing the phase's logic, stop — route.

---

## Operating rules (non-negotiable)

1. **NEVER reimplement a phase's logic yourself** (discovery, KB, roadmap, rules). Always route to its sub-skill — `Skill` for interactive phases, `Agent` for mechanical ones.
2. **NEVER ask the user strategic questions** (system_type, scale, stack, problem). Those questions belong to `kb-creator`.
3. **Own `step` and nothing else.** Only write `version`, `step`, `owner` to the state. Sub-skills write their own sections.
4. **Check sub-skill presence before dispatch.** If missing → offer install → degrade if declined.
5. **Resume by default.** If `.jr-orchestrator-state.json` exists with `step != "done"`, ask to resume or restart.
6. **STOP and wait after each AskUserQuestion call.** Never assume the answer.

---

## Shared State Contract

`.jr-orchestrator-state.json` lives at the project root. Schema **version 3**. The `kb`/`roadmap`/`skills`/`agents` sections are the frozen contract C-13b/c consume; `registry` was added additively (no C-13b/c sub-skill reads it) and bumped the contract from 2 → 3.

> Note: the `"version": 3` below is the **state-schema contract version** (bump only when the shared state shape changes). It is NOT the skill release version in the frontmatter (`version: "2.0"`) — the two version independently.

```json
{
  "version": 3,
  "step": "openspec|kb|roadmap|find-skill|registry|agents|done",
  "owner": "jr-orchestrator",
  "kb": {
    "created_by": "kb-creator",
    "source": "interactive|ingest",
    "discovery": {
      "system_type": "...",
      "domain": "...",
      "scale": "...",
      "stack": ["..."],
      "needs_infra": true,
      "problem": "..."
    },
    "files": ["knowledge-base/01-vision.md"]
  },
  "roadmap": {
    "created_by": "roadmap-generator",
    "changes_file": "CHANGES.md"
  },
  "skills": {
    "created_by": "find-skill",
    "recommended": [],
    "installed": []
  },
  "agents": {
    "created_by": "agent-instruction",
    "files": ["CLAUDE.md", "AGENTS.md"],
    "reglas_applied": []
  },
  "registry": {
    "created_by": "skill-registry",
    "file": ".atl/skill-registry.md"
  }
}
```

### Ownership rules

| Section | Owner | Writes |
|---|---|---|
| `version`, `step`, `owner` | `jr-orchestrator` (this skill) | Updated after each phase completes |
| `kb` (including `kb.discovery`) | `kb-creator` | After discovery + KB generation |
| `roadmap` | `roadmap-generator` | After CHANGES.md is produced |
| `skills` | `find-skill` | After recommendations + install |
| `agents` | `agent-instruction` | After CLAUDE.md/AGENTS.md generated |
| `registry` | `skill-registry` | After `.atl/skill-registry.md` is built (runs after `find-skill`, before `agent-instruction` — which consumes it) |

**There is NO orchestrator-owned `discovery` section.** The discovery lives inside `state.kb.discovery`, owned by `kb-creator`.

### Resume logic

At the start of every command invocation:

1. Check if `.jr-orchestrator-state.json` exists and load it.
2. If it exists and `step != "done"`:
   - Via `AskUserQuestion` (single-select): "Hay una fundación en progreso (paso: `{step}`). ¿Continuamos desde ahí o empezamos de cero?" — options: "Continuar desde `{step}`" / "Empezar de cero".
   - On "continuar": jump to the step recorded in state, using persisted sections as context.
   - On "de cero": delete `.jr-orchestrator-state.json`, start from Step 0.

---

## Step 0 — Detect entry point

Branch on which command fired:

| Command | Jump to |
|---|---|
| `/jr-orchestrator:init` | Step 1 — full flow |
| `/jr-orchestrator:kb` | Dispatch `kb-creator` directly (Step 2 — kb phase only) |
| `/jr-orchestrator:rules` | Dispatch `agent-instruction` (rules/CLAUDE.md re-gen only) |
| `/jr-orchestrator:openspec` | Step 1 — `openspec init` only |
| `/jr-orchestrator:devops` | Dispatch `devops-scaffolder` sub-skill (optional, full mode). Sub-skill not yet built — see Foundation flow notes. |
| `/jr-orchestrator:find-skill` | Dispatch `find-skill` directly |
| `/jr-orchestrator:registry` | Dispatch `skill-registry` directly (rebuild `.atl/skill-registry.md` after skills change) |
| No command / direct `Skill` call | Default to full flow (Step 1) |

Before branching: always load and apply resume logic above.

---

## Step 1 — openspec init

1. Verify `openspec` CLI is available:
   ```bash
   openspec --version
   ```
   If not found: tell the user how to install it (`npm install -g @fission-ai/openspec`) and mark `step: "openspec"` in state as pending. Stop.

2. If `openspec/` already exists in the project root: via `AskUserQuestion`: "Ya hay un OpenSpec inicializado. ¿Lo reuso tal cual o lo regenero?" — options: "Reusarlo", "Regenerar". On "reusarlo" skip to Step 2.

3. Run:
   ```bash
   openspec init
   ```

4. Write initial state file:
   ```json
   { "version": 3, "step": "kb", "owner": "jr-orchestrator" }
   ```

5. Advance to Step 2.

---

## Step 2 — Foundation flow dispatch (§4.2 order)

Dispatch the foundation phases in this exact order. **The execution model is hybrid** — how each phase runs depends on whether it must talk to the user:

| Phase | Runs | Why |
|---|---|---|
| kb-creator | **inline** (`Skill`) | Interactive — its discovery Q&A must reach the user; a sub-agent runs autonomously and cannot ask |
| roadmap-generator | **sub-agent** (`Agent`) | Mechanical — reads the KB, writes `CHANGES.md`, no user interaction |
| find-skill | **sub-agent** (`Agent`) | Mechanical — recommends + installs matching skills |
| skill-registry | **sub-agent** (`Agent`) | Mechanical — heavy scan of every `SKILL.md` |
| agent-instruction | **inline** (`Skill`) | Interactive — asks the user for the project's hard rules |

**Why delegate the mechanical phases**: they read many files and emit large output. Running them inline would inflate the orchestrator's context and break its thin-coordinator constitution (*"delegate real work to sub-agents"*). **Why the interactive phases stay inline**: a sub-agent cannot prompt the user — delegating an interactive phase would silently drop its questions. So interactivity decides placement, not preference.

**Never reimplement a phase's logic inline.** Mechanical phases delegate to a sub-agent that invokes the skill; interactive phases invoke the skill directly. Either way the skill does the work — the orchestrator only routes.

**Sub-agent launch pattern** (mechanical phases) — use the `Agent` tool with a complete brief, since the sub-agent starts with NO context:

```
Agent({
  description: "Foundation: <phase> for <project>",
  model: "<model-routing: sonnet default>",
  prompt: `
    ## Task
    Use the Skill tool to invoke \`<skill-name>\` for this project.
    ## Context
    - Shared state: .jr-orchestrator-state.json (read it for prior phases' output)
    - Prior artifacts: <relevant paths — knowledge-base/, CHANGES.md, .atl/skill-registry.md>
    ## Instructions
    Follow the skill completely. When done, write your own section into
    .jr-orchestrator-state.json and return a one-paragraph summary + the
    artifact paths produced.
  `
})
```

After a sub-agent returns: read `.jr-orchestrator-state.json` to confirm its section was written, then advance `step`.

Before each dispatch: check lazy-load (see Step 3 — Graceful Degradation).

### Phase 1: kb-creator (FIRST — produces all discovery + KB)

**Why first**: `kb-creator` is the only sub-skill that runs discovery. Every downstream sub-skill consumes its output (`state.kb.discovery` + `knowledge-base/`). Nothing else can run before `kb-creator` completes.

**kb-creator has two operational sources** (the sub-skill handles the choice — the orchestrator does NOT choose):
- `interactive` — runs a Q&A discovery session (system_type, scale, stack, problem, domains) for a project being built from scratch.
- `ingest` — derives the KB from existing documents (specs, READMEs, docs) that the user provides when the project already has documentation.

In both cases, `kb-creator` writes `state.kb` (including `state.kb.discovery`, `state.kb.source`, `state.kb.files`) and all `knowledge-base/*.md` files.

**Dispatch** (inline — interactive, must reach the user):
```
Skill("kb-creator")
```

After kb-creator completes: read `.jr-orchestrator-state.json`; confirm `state.kb.discovery` and `state.kb.files` are populated. If not populated (skip was recorded), proceed to Phase 2 but note that downstream sub-skills will have incomplete input.

Update state: `step = "roadmap"`.

### Phase 2: roadmap-generator

**Input consumed**: `state.kb.discovery` + `state.kb.files` (reads `knowledge-base/`).
**Output**: `CHANGES.md` + `state.roadmap`.

**Dispatch** (sub-agent — mechanical):
```
Agent({
  description: "Foundation: roadmap-generator",
  model: "sonnet",
  prompt: "Use the Skill tool to invoke `roadmap-generator`. Read .jr-orchestrator-state.json + knowledge-base/ for input. Produce CHANGES.md and write state.roadmap. Return a summary + the CHANGES.md path."
})
```

After the sub-agent returns: confirm `CHANGES.md` exists. Update state: `step = "find-skill"`.

### Phase 3: find-skill

**Input consumed**: `{ stack, domains, problem }` derived from `state.kb.discovery`.
**Output**: recommendations table + `state.skills` (recommended + installed lists).

**Dispatch** (sub-agent — mechanical):
```
Agent({
  description: "Foundation: find-skill",
  model: "sonnet",
  prompt: "Use the Skill tool to invoke `find-skill`. Derive { stack, domains, problem } from state.kb.discovery in .jr-orchestrator-state.json. Recommend + install matching skills, then write state.skills (recommended + installed). Return the recommendations table + installed list."
})
```

After the sub-agent returns: update state: `step = "registry"`.

### Phase 4: skill-registry (after find-skill — feeds both agent-instruction AND the SDD orchestrator)

**Why here — after `find-skill`, before `agent-instruction`**: the registry is a build-time scan of every installed skill (it reads each `SKILL.md` and distills compact rules). It MUST run AFTER `find-skill` — that phase *installs* the domain skills, and `skill-registry` *scans* them; run it earlier and it scans a directory missing the skills about to be installed. And it MUST run BEFORE `agent-instruction` — Phase 5 consumes `.atl/skill-registry.md` as its single source of truth for which skills exist (instead of re-scanning the filesystem), so the registry has to exist first. One scan, one source of truth.

**Why it matters**: without this phase the project is founded with NO `.atl/skill-registry.md`. The SDD orchestrator's Skill Resolver Protocol then finds no registry, falls back to `.agents/SKILLS.md` (which nothing writes), and every sub-agent runs WITHOUT the project's compact rules. This phase closes that loop — built once here, read cheaply at every delegation.

**Input consumed**: the installed skills on disk (`state.skills.installed` + the agent skills dirs).
**Output**: `.atl/skill-registry.md` (+ engram upsert if available) + `state.registry`.

> Note on "Project Conventions": in this first foundation pass `CLAUDE.md`/`AGENTS.md` don't exist yet (Phase 5 generates them), so the registry's conventions section starts empty. That's fine — the compact rules (the critical output) are complete, and the orchestrator reads `CLAUDE.md` directly anyway. A later `/jr-orchestrator:registry` re-run indexes the conventions too.

**Dispatch** (sub-agent — mechanical):
```
Agent({
  description: "Foundation: skill-registry",
  model: "sonnet",
  prompt: "Use the Skill tool to invoke `skill-registry`. Scan the installed skills, build .atl/skill-registry.md with compact rules, write state.registry. Return a summary + confirm .atl/skill-registry.md exists."
})
```

After the sub-agent returns: confirm `.atl/skill-registry.md` exists. Update state: `step = "agents"`. Proceed to Phase 5.

### Phase 5: agent-instruction (LAST — interactive; needs KB + the registry)

**Why last**: `agent-instruction` builds the Navigation Map (Mapa de Navegación) referencing all KB files (`state.kb.files`) and reads `.atl/skill-registry.md` as its **single source of truth for available skills**. It maps those skills to the project's agent roles and *references* the registry for the compact rules — it does NOT copy the rules into `CLAUDE.md` (those live only in the registry, which is not versioned). All inputs exist only after Phases 1, 3 and 4 complete.

**Why inline + interactive**: this phase **asks the user for the project's hard rules** (stack-aware — it proposes defaults from the detected stack and confirms), so it must run inline. It also generates a project `AGENTS.md`/`CLAUDE.md` that does **NOT repeat** the global `~/.claude/CLAUDE.md` the stack already installed — only project-specific instructions. A sub-agent could not ask the rules, so this phase is never delegated.

**Input consumed**: `state.kb.discovery` + `state.kb.files` (reads `knowledge-base/`) + `.atl/skill-registry.md` (skills source of truth) + the user's confirmed hard rules + the global `~/.claude/CLAUDE.md` (to avoid duplication).
**Output**: project `CLAUDE.md` / `AGENTS.md` + `state.agents`.

**Dispatch** (inline — interactive, asks the user for the hard rules):
```
Skill("agent-instruction")
```

After completion: update state: `step = "done"`. Proceed to Step 4 (summary).

---

## Step 3 — Lazy-load dispatch + graceful degradation

Before invoking each sub-skill via the `Skill` tool, check that it is installed:

```bash
npx skills list -g | grep <skill-name>
```

Or check for local install:
```bash
npx skills list | grep <skill-name>
```

**If the sub-skill is found**: dispatch normally.

**If the sub-skill is NOT found**:

1. Inform the user:
   > "La sub-skill `<name>` no está instalada. La necesito para la fase `<phase>`."

2. Offer install via `AskUserQuestion` (single-select):
   > "¿La instalamos ahora?" — options: "Sí, instalar" / "No, saltear esta fase".

3. On "Sí, instalar": run:
   ```bash
   npx skills add <repo> --skill <name> -g
   ```
   Where `<repo>` is the skill's source repo (see catalog). Then dispatch normally.

4. On "No, saltear":
   - Mark the skip in state:
     ```json
     { "step": "<next-phase>", "<section>": { "skipped": true, "reason": "sub-skill not installed" } }
     ```
   - Log to the user: "Fase `<phase>` salteada. Podés instalar `<name>` luego y correr `/jr-orchestrator:<phase>` para ejecutarla de forma aislada."
   - Continue to the next phase. **Do NOT abort the full flow.**

**Known repos for install offers**:

| Sub-skill | Repo | Visibility |
|---|---|---|
| `kb-creator` | `JuanCruzRobledo/kb-creator` | public |
| `roadmap-generator` | `JuanCruzRobledo/roadmap-generator` | public |
| `find-skill` | `vercel-labs/skills` (third-party) | public |
| `agent-instruction` | `JuanCruzRobledo/agent-instruction` | public |
| `skill-registry` | `JuanCruzRobledo/skill-registry` | public |

> Note: this skill lives at `JuanCruzRobledo/jr-orchestrator` (public). The legacy private repo `JuanCruzRobledo/jr-starter` is a different project and is not used by the stack.

---

## Step 4 — Summary

When `step == "done"` (all phases complete or skipped):

1. Update `.jr-orchestrator-state.json`: `step: "done"`.
2. Show the user:
   - Tree of generated project structure:
     ```bash
     eza --tree --level=3
     ```
     (or equivalent; do NOT use `find`/`ls`)
   - List of files created, one-liner per file.
   - Any phases that were skipped and why.
   - Suggested next command: `/opsx:propose <primer-change-de-CHANGES.md>` (if CHANGES.md was generated) or the next logical action.
3. Reminder: "Podés re-ejecutar fases individuales: `/jr-orchestrator:kb` para agregar dominios, `/jr-orchestrator:rules` para regenerar CLAUDE.md, `/jr-orchestrator:find-skill` para agregar skills, `/jr-orchestrator:registry` para reconstruir el skill-registry después de instalar/quitar skills."

---

## Sub-skill I/O matrix (frozen contract for C-13b/c)

This table is the **frozen contract**. C-13b (`kb-creator`) and C-13c (`roadmap-generator` + `agent-instruction`) implement against this.

| Sub-skill | Input | Output |
|---|---|---|
| `kb-creator` | **Two sources (sub-skill decides)**: (a) **interactive** — Q&A discovery: system_type, scale, stack, problem, domains; (b) **ingest** — existing docs/specs/READMEs provided by the user | `knowledge-base/*.md` + `state.kb` (`discovery`, `source`, `files`) |
| `roadmap-generator` | `state.kb.discovery` + `state.kb.files` (reads `knowledge-base/`) | `CHANGES.md` + `state.roadmap` |
| `find-skill` | `{ stack, domains, problem }` derived from `state.kb.discovery` | recommendations table + `state.skills` |
| `skill-registry` | installed skills on disk (`state.skills.installed` + agent skills dirs) | `.atl/skill-registry.md` (+ engram upsert) + `state.registry` |
| `agent-instruction` | `state.kb.discovery` + `state.kb.files` (reads `knowledge-base/`) + `.atl/skill-registry.md` (skills source of truth) + applicable rule snippets | `CLAUDE.md` / `AGENTS.md` + `state.agents` |

**Order constraint**: `kb-creator` runs FIRST (produces discovery + KB everyone consumes). `find-skill` installs the domain skills. `skill-registry` then scans them and distills compact rules into `.atl/skill-registry.md`. `agent-instruction` runs LAST — it consumes the registry as its single source of truth for available skills (no re-scan) and needs the KB index too.

---

## Errors and edge cases

- **`openspec` CLI not installed**: offer install link, mark step pending, stop gracefully.
- **`.jr-orchestrator-state.json` exists, `step != "done"`**: resume prompt (see Shared State Contract above).
- **Target directory has existing files**: list them; ask "¿Continúo, mergeo, o cancelo?" before writing anything.
- **User quits mid-flow**: confirm state was saved; "Ejecutá `/jr-orchestrator:init` para retomar."
- **All sub-skills missing**: inform the user that the full flow requires sub-skills from the Full install mode of `jr-stack`. Offer the install command: `jr-stack install --mode full`.

---

## What this skill does NOT do

- Does NOT ask discovery questions (system_type, scale, stack, problem). That is `kb-creator`.
- Does NOT write `knowledge-base/*.md` files. That is `kb-creator`.
- Does NOT generate `CHANGES.md`. That is `roadmap-generator`.
- Does NOT generate `CLAUDE.md` or `AGENTS.md`. That is `agent-instruction`.
- Does NOT implement devops scaffolding inline. Decided: devops scaffolding lives in a dedicated `devops-scaffolder` sub-skill (optional, full mode), dispatched like any other phase — never inlined. The sub-skill itself is built in a separate change (own repo `JuanCruzRobledo/devops-scaffolder`); until then `/jr-orchestrator:devops` degrades gracefully (sub-skill not installed).
- Does NOT port templates (`templates/kb/*`, `templates/reglas/*`, docker-compose) — those feed the sub-skills in C-13b/c, not the orchestrator.
- Does NOT commit or push (unless explicitly asked).
- Does NOT install project dependencies (`npm install`, `pip install`, etc.).

---

## Voice / tone

- Rioplatense Spanish (voseo) when the user speaks Spanish. English if the user responds in English.
- Arquitecto apasionado: explica el porqué de cada decisión, no se limita a ejecutar.
- When a phase is skipped: acknowledge it clearly and tell the user how to come back to it.
