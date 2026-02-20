# Grechman

Claude Code slash command that runs a full dev workflow for medium/hard tasks. Give it a task, it plans, branches, fetches docs, implements, and does a security review before finishing.

---

## What it does

`/grechman add OAuth login` will:

1. Check required skills are installed
2. Count implementation steps and make sure they fit in the iteration budget
3. Write a design doc and a plan
4. Create a `grechman/<slug>` branch (checks your tree is clean first)
5. Pull library docs from Context7 and cache them
6. Pick an execution mode and run
7. Drop a fallback file if it gets stuck so you can resume
8. Do a code + security review before the final commit

---

## Install

Copy `grechman.md` to your Claude Code commands folder:

```
~/.claude/commands/grechman.md
```

Windows:
```
%USERPROFILE%\.claude\commands\grechman.md
```

Reload Claude Code, type `/grechman` to check it shows up.

---

## Required skills

These two need to be installed or it won't start:

- `ralph-loop`
- `superpowers:*`

Uses these if they're there:

- `frontend-design` — UI/CSS work
- `humanizer` — cleans PR descriptions when `--github on`
- `code-review:code-review` — reviewing existing PRs
- Playwright MCP — browser/frontend tasks
- MCP Memory, Sequential Thinking MCP, Desktop Commander MCP

---

## Usage

```
/grechman <task> [options]
```

| Flag | Default | Values | What it does |
|---|---|---|---|
| `--complexity` | `medium` | `medium` / `hard` | Sets iteration budget |
| `--git` | `on` | `on` / `off` | Branch and commit after each step |
| `--github` | `off` | `on` / `off` | Push and open a PR when done |
| `--pre-specified` | `off` | `on` / `off` | Skip brainstorming if you already have a plan |
| `--max-iterations-ralph` | 15 / 25 | any number | Override the iteration cap |
| `--resume` | — | path to fallback file | Resume a stuck session |

```bash
/grechman add avatar upload to profile page

/grechman refactor auth to JWT --complexity hard --github on

/grechman implement plan in docs/plans/api-redesign.md --pre-specified on

/grechman --resume grechman-fallback.md --complexity hard --git on
```

---

## Execution modes

It picks one before running anything and asks you to confirm:

- **RALPH_LOOP** — default, for anything with 8 or fewer tightly coupled steps
- **SEQUENTIAL_SUBAGENTS** — 3+ stages with clear handoffs (schema → models → API → tests)
- **PARALLEL** — multiple groups of steps that touch completely different files

---

## Git

With `--git on`:

- Branch: `grechman/<slug>`
- Commits: `grechman(step N): <description>`
- Always makes three batch commits: `grechman(init)`, `grechman(knowledge)`, `grechman(done)`
- Won't commit until verification passes
- Never touches `main` or `master`

---

## When it gets stuck

If it hits the iteration limit, a merge conflict, a missing package, or a rollback it can't recover from — it writes `grechman-fallback.md` with the last stable SHA and what's left to do, then stops and prints the resume command:

```bash
/grechman --resume grechman-fallback.md --complexity hard --git on
```

---

## Files

| File | What it is |
|---|---|
| `CLAUDE.md` | Session log |
| `knowledge.md` | Cached library docs |
| `docs/plans/*.md` | Design docs and plans |
| `grechman-fallback.md` | Resume state |
| `grechman-dispatch.md` | Work manifest (deleted when done) |

---

## Security

Before the final commit it checks: XSS, exposed paths, SQL injection, auth logic, input validation, rate limiting, hardcoded secrets. Finds something it can't fix in one attempt — it stops and tells you. Doesn't skip it.

---

## License

MIT
