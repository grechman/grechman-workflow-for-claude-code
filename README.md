# Grechman

A set of Claude Code slash commands for running disciplined dev work end to end. Three commands:

- **`/grechman`** — runs a full dev workflow on a task. Plans, branches, implements with subagents, reviews its own work, finishes.
- **`/grechman-review`** — multi-agent review of a commit range or branch. Three parallel agents (security, correctness, architecture) write reports; an editor + resolver pair applies fixes on demand.
- **`/grechman-ontology`** — generates and maintains a stack-agnostic project ontology that the other two commands read for context.

Stack-agnostic: works the same on Go, Rust, Python, Node, Ruby, JVM, .NET, Elixir, etc. The ontology detects the stack from manifest files (`go.mod`, `Cargo.toml`, `pyproject.toml`, `package.json`, `Gemfile`, `pom.xml`, `*.csproj`, `mix.exs`, etc.). No assumptions about framework, database, or hosting provider.

`/grechman` is for medium and hard tasks. Simple stuff doesn't need this.

---

## Install

Copy these files into your Claude Code config (mirror the layout):

```
~/.claude/commands/grechman.md
~/.claude/commands/grechman-ontology.md
~/.claude/commands/grechman-review.md

~/.claude/grechman/steps/00-setup.md
~/.claude/grechman/steps/01-planning.md
~/.claude/grechman/steps/02-dispatch.md
~/.claude/grechman/steps/03-agent-contract.md
~/.claude/grechman/steps/04-review.md
~/.claude/grechman/steps/05-finish.md
~/.claude/grechman/steps/fallback.md

~/.claude/agents/grechman-architecture.md
~/.claude/agents/grechman-correctness.md
~/.claude/agents/grechman-editor.md
~/.claude/agents/grechman-resolver.md
~/.claude/agents/grechman-security.md
```

Or one-shot:

```bash
git clone https://github.com/grechman/grechman-workflow-for-claude-code.git
cp -r grechman-workflow-for-claude-code/.claude/. ~/.claude/
```

Reload Claude Code, type `/grechman` to verify.

---

## Required plugins / skills

`/grechman` won't start without:

- `superpowers:*` — brainstorming, writing-plans, test-driven-development, systematic-debugging, verification-before-completion, requesting-code-review, finishing-a-development-branch, using-git-worktrees

Used if installed:

- `frontend-design` — UI/CSS work
- `humanizer` — cleans PR descriptions when `--github on`
- `code-review:code-review` — reviewing existing PRs
- Playwright MCP — browser/frontend tasks
- MCP Memory, Sequential Thinking MCP
- A database MCP (Postgres / SQLite / Mongo / Supabase / etc.) — fed into the ontology if connected
- `adr-tools` — architecture decision records

---

## depwire (optional, recommended)

depwire builds a tree-sitter dependency graph of your codebase. No LLM calls, runs locally. The ontology uses it to flag load-bearing files (high fan-in) so coding agents know what has wide blast radius.

```bash
npm install -g depwire-cli
```

Called once during ontology extraction (`/grechman-ontology --diff`). Supports TypeScript, JavaScript, Python, Go, Rust, C.

---

## /grechman — usage

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
| `--budget` | 15 / 25 | any number | Override the iteration cap |
| `--resume` | — | path to fallback file | Resume a stuck session |

```bash
/grechman add avatar upload to profile page

/grechman refactor auth to JWT --complexity hard --github on --ontology on

/grechman implement plan in docs/plans/api-redesign.md --pre-specified on

/grechman --resume grechman-fallback.md --complexity hard --git on
```

---

## /grechman — execution modes

Picked before running, confirmed with you:

- **RALPH_LOOP** — one agent per step, sequential. Default for tightly coupled work, ≤ 8 steps.
- **SEQUENTIAL_SUBAGENTS** — one agent per phase. For 3+ ordered stages, > 8 steps, or work crossing distinct domains.

---

## /grechman — review loop

After coding completes, the review agent runs up to 3 times:

1. Runs `/grechman-ontology --diff` to catch dependency drift
2. Code review via `superpowers:requesting-code-review`
3. Security review (XSS, SQL injection, auth, secrets, input validation, rate limiting)
4. Fixes what it finds and commits
5. If the same issue shows up twice across iterations, stops and asks you what to do

---

## /grechman — git

With `--git on`:

- Branch: `grechman/<slug>`
- Commits: `grechman(step N): <description>`
- Review fixes: `grechman(review): fix review issues iteration N`
- Final: `grechman(done): session complete`
- Never touches `main` or `master`
- Auto-detects jujutsu (`jj`) if installed, falls back to git

---

## /grechman — when it gets stuck

Hits the iteration limit, a merge conflict, or something it can't recover from? It writes `grechman-fallback.md` with the last stable SHA, decision log, and remaining work, then stops:

```bash
/grechman --resume grechman-fallback.md --complexity hard --git on
```

---

## /grechman — steps

| # | Step | File | Skip when |
|---|---|---|---|
| 0 | Setup | `00-setup.md` | never |
| 0o | Ontology Refresh | inline | `--ontology off` or `--resume` |
| 1 | Planning | `01-planning.md` | `--resume` |
| 2 | Dispatch | `02-dispatch.md` | `--resume` (reads existing dispatch) |
| 3-N | Coding | `03-agent-contract.md` | — |
| R | Review (loop, max 3) | `04-review.md` | — |
| F | Finish | `05-finish.md` | — |

---

## /grechman-review — usage

Two-stage workflow.

```bash
# Stage 1 — research only. Three agents write YAML reports.
/grechman-review                       # current branch vs main/master/develop
/grechman-review <sha-or-tag-or-branch>  # since that ref to HEAD
/grechman-review --since v1.2.3
/grechman-review --range A..B
/grechman-review --last 30
/grechman-review "changes made on m06-m12"   # natural-language scope, asks to confirm

# Stage 2 — apply fixes from the most recent research run.
/grechman-review --fixall
```

Three review agents run in parallel:

- `grechman-security` — SQLi, auth bypass, crypto, code execution, data exposure
- `grechman-correctness` — logic bugs, edge cases, concurrency, error handling, invariants
- `grechman-architecture` — coupling, cohesion, boundaries, abstraction, YAGNI on the change

Reports land in `.grechman/review/<timestamp>/reports/{security,correctness,architecture}.yaml`. Read them, then call `--fixall` when ready.

`--fixall` dispatches `grechman-editor` to plan and apply fixes one commit at a time, with `grechman-resolver` validating each commit (tests, typecheck, lint when present, plus semantic judgment) and reverting any that break things.

| Flag | Meaning |
|---|---|
| `--fixall` | Apply fixes from the most recent research run |
| `--since <ref>` | Review everything from `<ref>` to HEAD |
| `--range <A..B>` | Explicit two-point git range |
| `--last <N>` | Last N commits |
| `--base <branch>` | Diff HEAD against this branch |
| `--fail-on <sev>` | Exit nonzero if residual ≥ severity |
| `--max-fixes <N>` | Cap on fixes applied (50 default in --fixall) |
| `--timeout-research <N>` | Seconds per research agent (omit to wait indefinitely) |

---

## /grechman-ontology — usage

```bash
# First time: full interview + auto-extraction
/grechman-ontology

# After manifest or schema changes: refresh _generated, leave _manual alone
/grechman-ontology --diff

# Used internally by /grechman to scope ontology to specific files
/grechman-ontology --scope <file1> [file2...]
```

Two blocks in `ontology.yaml`:

- `_generated` — auto-extracted (stack, entities, dependency graph, migrations). Overwritten on `--diff`.
- `_manual` — your conventions, decisions, rejected approaches, constraints. Append-only, never overwritten.

### Stack detection

The ontology scans for any of these manifests in the project root and parses what it finds. No single stack is assumed:

- Go: `go.mod`, `go.sum`
- Rust: `Cargo.toml`
- Python: `pyproject.toml`, `requirements.txt`, `Pipfile`, `poetry.lock`, `uv.lock`
- Node / TS / Deno / Bun: `package.json`, `tsconfig.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`, `deno.json`
- Ruby: `Gemfile`
- PHP: `composer.json`
- Elixir: `mix.exs`
- JVM: `pom.xml`, `build.gradle(.kts)`
- .NET: `*.csproj`, `*.fsproj`, `*.sln`
- Swift: `Package.swift`
- Dart / Flutter: `pubspec.yaml`
- Build / runtime: `Dockerfile`, `docker-compose.yml`, `Procfile`, `fly.toml`, `vercel.json`, `netlify.toml`, `wrangler.toml`, `serverless.yml`
- Pinned versions: `.tool-versions`, `.nvmrc`, `.python-version`, `.ruby-version`, `.go-version`
- Env var **names** only: `.env.example`, `.env.sample` (values are never read)

### Persistence detection

Schema files: `prisma/schema.prisma`, `drizzle.config.*`, `db/schema.rb`, `alembic/versions/`, `ent/schema/`, `sqlc.yaml`, raw `*.sql` in `schema/`/`sql/`/`db/`, etc. Migration directories: `migrations/`, `db/migrate/`, `prisma/migrations/`, etc.

If a database MCP is connected (Postgres / SQLite / Mongo / Supabase / Prisma / etc.), entities are extracted from there too. Nothing is installed automatically.

### How the workflow uses it

- Before planning (`--ontology on`): refreshes ontology. Planning agent reads conventions, constraints, rejected approaches, load-bearing files.
- During dispatch: knowledge block flags high fan-in files so coding agents know what has a wide blast radius.
- During review: ontology refreshed again to detect new circular deps or shifted dependencies.
- During coding: agents read `_manual` for rules, append new conventions or rejected approaches as they go.

### ontology.yaml example (illustrative — substitute the real stack)

```yaml
_generated:
  last_updated: "2026-04-29"
  stack:
    languages: ["go@1.22"]
    frameworks: ["chi", "ent"]
    persistence: ["postgres@15"]
    hosting: ["fly.io"]
    containerization: ["docker"]
    env_vars: ["DATABASE_URL", "SENTRY_DSN", "JWT_SECRET"]
  entities:
    Order:
      source: "ent/schema/order.go"
      constraints: ["INSERT only via service layer (orders.Create)"]
  dependencies:
    tool: "depwire-cli"
    parsed_at: "2026-04-29"
    total_symbols: 247
    total_edges: 89
    load_bearing_files:
      - path: "internal/db/db.go"
        fan_in: 23
    circular_deps: []

_manual:
  conventions:
    - "Domain logic stays in internal/ — never in cmd/"
    - "All DB writes go through the service layer"
  decisions:
    - "Use ent over GORM — compile-time safety on a domain that changes weekly (adr: 0001)"
  rejected_approaches:
    - step: 3
      session: "2026-04-29"
      approach: "Single shared transaction across the request"
      reason: "Caused contention under load — moved to per-aggregate transactions"
  constraints:
    - "Never modify migrations once they've shipped — write a new one"
    - "Never log JWT_SECRET or full Authorization headers"
```

---

## Files the workflow creates

| File | What |
|---|---|
| `CLAUDE.md` | Session log |
| `knowledge.md` | Cached library docs (from context7 MCP if installed) |
| `ontology.yaml` | Project domain model |
| `.depwire/graph.json` | Raw depwire graph (gitignored) |
| `docs/plans/*.md` | Design docs and plans |
| `doc/adr/*.md` | ADRs (if adr-tools installed) |
| `grechman-fallback.md` | Resume state |
| `grechman-dispatch.md` | Work manifest (deleted when done) |
| `.grechman/task-reports/` | Per-task YAML reports (gitignored) |
| `.grechman/sessions/` | Archived session reports (gitignored) |
| `.grechman/knowledge-block.md` | Context block for coding agents |
| `.grechman/review/<ts>/` | `/grechman-review` reports + fix logs |

Recommended `.gitignore`:

```
.grechman/task-reports/
.grechman/sessions/
.grechman/review/
.depwire/
```

---

## License

MIT
