# Grechman

A Claude Code slash command that runs a full dev workflow. Give it a task, it plans, branches, implements with subagents, and reviews its own work before finishing.

For medium and hard tasks only. Simple stuff doesn't need this.

---

## What it does

`/grechman add OAuth login` will:

1. Check required skills are installed
2. Optionally refresh project ontology (`--ontology on`)
3. Count implementation steps, check they fit in the budget
4. Write a design doc and plan, using ontology data if available (conventions, constraints, past failures)
5. Create a `grechman/<slug>` branch
6. Pull library docs from Context7 and cache them
7. Pick an execution mode, dispatch coding agents
8. Write a fallback file if it gets stuck so you can resume
9. Run a review loop (up to 3 passes, fixes its own issues, alerts you if the same bug keeps coming back)
10. Refresh ontology to capture any dependency drift

---

## Install

Copy these files to your Claude Code config:

```
~/.claude/commands/grechman.md
~/.claude/commands/grechman-ontology.md
~/.claude/grechman/steps/00-setup.md
~/.claude/grechman/steps/01-planning.md
~/.claude/grechman/steps/01u-ultraplan-handoff.md
~/.claude/grechman/steps/02-dispatch.md
~/.claude/grechman/steps/03-agent-contract.md
~/.claude/grechman/steps/04-review.md
~/.claude/grechman/steps/05-finish.md
~/.claude/grechman/steps/fallback.md
```

Reload Claude Code, type `/grechman` to check it shows up.

---

## Required skills

Won't start without these:

- `superpowers:*` (brainstorming, writing-plans, requesting-code-review, finishing-a-development-branch)

Uses these if installed:

- `frontend-design` -- UI/CSS work
- `humanizer` -- cleans PR descriptions when `--github on`
- `code-review:code-review` -- reviewing existing PRs
- Playwright MCP -- browser/frontend tasks
- MCP Memory, Sequential Thinking MCP
- `adr-tools` -- architecture decision records

---

## depwire (optional, recommended)

depwire gives the ontology system a dependency graph of your codebase via tree-sitter. No LLM calls, runs locally.

```bash
sudo npm install -g depwire-cli
```

Called once during ontology extraction (`/grechman-ontology --diff`). Not an MCP server. Supports TypeScript, JavaScript, Python, Go, Rust, C.

---

## Usage

```
/grechman <task> [options]
```

| Flag | Default | Values | What it does |
|---|---|---|---|
| `--complexity` | `medium` | `medium` / `hard` | Sets iteration budget (15 / 25) |
| `--git` | `on` | `on` / `off` | Branch and commit after each step |
| `--github` | `off` | `on` / `off` | Push and open a PR when done |
| `--pre-specified` | `off` | `on` / `off` | Skip brainstorming if you already have a plan |
| `--ontology` | `off` | `on` / `off` | Run `/grechman-ontology --diff` before planning |
| `--planning` | `auto` | `local` / `ultra` / `auto` | Planning mode (auto = ultra if hard, local if medium) |
| `--budget` | 15 / 25 | any number | Override the iteration cap |
| `--resume` | -- | path to fallback file | Resume a stuck session |

```bash
/grechman add avatar upload to profile page

/grechman refactor auth to JWT --complexity hard --github on --ontology on

/grechman implement plan in docs/plans/api-redesign.md --pre-specified on

/grechman --resume grechman-fallback.md --complexity hard --git on
```

---

## Execution modes

Picked before running, confirmed with you:

- RALPH_LOOP -- one agent per step, sequential
- SEQUENTIAL_SUBAGENTS -- one agent per phase, phases defined in dispatch

---

## Review loop

After coding completes, the review agent runs up to 3 times:

1. Runs `/grechman-ontology --diff` to catch dependency drift
2. Code review via `superpowers:requesting-code-review`
3. Security review (XSS, SQL injection, auth, secrets, input validation)
4. Fixes what it finds and commits
5. If the same issue shows up twice across iterations, stops and asks you what to do

---

## Git

With `--git on`:

- Branch: `grechman/<slug>`
- Commits: `grechman(step N): <description>`
- Review fixes: `grechman(review): fix review issues iteration N`
- Final: `grechman(done): session complete`
- Never touches `main` or `master`

---

## When it gets stuck

Hits the iteration limit, a merge conflict, or something it can't recover from? It writes `grechman-fallback.md` with the last stable SHA and remaining work, then stops:

```bash
/grechman --resume grechman-fallback.md --complexity hard --git on
```

---

## Ontology system

Maintains a structured model of your project so agents don't start from zero each session. Two blocks: `_generated` (auto-extracted, overwritten on refresh) and `_manual` (your conventions and decisions, never overwritten).

### Setup

```bash
# Install the ontology command
cp .claude/commands/grechman-ontology.md ~/.claude/commands/

# Optional: depwire for dependency graphs (no LLM tokens)
sudo npm install -g depwire-cli

# Optional: adr-tools for architecture decision records
npm install -g adr-tools
adr init doc/adr  # once per project
```

### First ontology

```bash
/grechman-ontology
```

Interviews you, auto-extracts from package.json, tsconfig, depwire, Supabase (if MCP available). Creates `ontology.yaml` in your project root.

### Refresh after changes

```bash
/grechman-ontology --diff
```

Re-extracts `_generated`, leaves `_manual` alone.

### How the workflow uses it

- Before planning (`--ontology on`): refreshes ontology. Planning agent reads conventions, constraints, rejected approaches, load-bearing files.
- During dispatch: knowledge block flags high fan-in files so coding agents know what has a wide blast radius.
- During review: ontology refreshed again to detect new circular deps or shifted dependencies.
- During coding: agents read `_manual` for rules, append new conventions or rejected approaches as they go.

### ontology.yaml

```yaml
_generated:           # auto-extracted, overwritten by --diff
  last_updated: "2026-02-28"
  stack:
    framework: "next@14.2.0"
    database: "supabase-postgres"
  entities:
    Post:
      source: "public.posts"
      rls: true
      constraints: ["INSERT only via Edge Function post-create"]
  dependencies:       # from depwire CLI, no LLM tokens
    tool: "depwire-cli"
    parsed_at: "2026-02-28"
    total_symbols: 247
    total_edges: 89
    load_bearing_files:
      - path: "src/lib/db.ts"
        fan_in: 23
    circular_deps: []

_manual:              # append-only, written by agents and you
  conventions:
    - "Supabase client only in server components"
  decisions:
    - "Use Supabase Auth over NextAuth -- native RLS support"
  rejected_approaches:
    - step: 3
      session: "2026-02-28"
      approach: "Direct Postgres trigger for post creation"
      reason: "Cannot add business logic in trigger"
  constraints:
    - "Never change DB schema without a migration file"
```

---

## Files it creates

| File | What |
|---|---|
| `CLAUDE.md` | Session log |
| `knowledge.md` | Cached library docs |
| `ontology.yaml` | Project domain model |
| `.depwire/graph.json` | Raw depwire graph (gitignored) |
| `docs/plans/*.md` | Design docs and plans |
| `doc/adr/*.md` | ADRs (if adr-tools installed) |
| `grechman-fallback.md` | Resume state |
| `grechman-dispatch.md` | Work manifest (deleted when done) |
| `.grechman/task-reports/` | Per-task YAML reports |
| `.grechman/knowledge-block.md` | Context block for coding agents |

---

## Steps

| # | Step | File | Skip when |
|---|---|---|---|
| 0 | Setup | `00-setup.md` | never |
| 0o | Ontology Refresh | inline | `--ontology off` or `--resume` |
| 1 | Planning | `01-planning.md` | `--resume` or `planning=ultra` |
| 1U | Ultraplan Handoff | `01u-ultraplan-handoff.md` | `planning=local` or `--resume` |
| 2 | Dispatch | `02-dispatch.md` | `--resume` (reads existing dispatch) |
| 3-N | Coding | `03-agent-contract.md` | -- |
| R | Review (loop, max 3) | `04-review.md` | -- |
| F | Finish | `05-finish.md` | -- |

---

## License

MIT
