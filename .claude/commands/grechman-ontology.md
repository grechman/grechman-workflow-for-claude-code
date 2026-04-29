# /grechman-ontology — Project Ontology Generator

You are running the Grechman Ontology command. Parse $ARGUMENTS before doing anything.

## Arguments from user:
$ARGUMENTS

---

## WHAT THIS COMMAND DOES

Generates and maintains `ontology.yaml` — a structured, machine-readable model of the project's domain, stack, constraints, and conventions. This file is automatically read by `/grechman` at the start of every session and injected into each agent's KNOWLEDGE BLOCK.

The ontology is **stack-agnostic**. It detects whatever the project actually uses (Go, Rust, Python, Node, Ruby, JVM, .NET, Elixir, etc.) by inspecting standard manifest files. It does NOT assume any particular framework, database, or hosting provider.

**Flags:**
- `--init` (default) — full interview, creates ontology.yaml from scratch
- `--diff` — re-extracts `_generated` block from live sources, preserves `_manual` block untouched
- `--scope <file1> [file2...]` — prints the scoped ONTOLOGY excerpt for given files (used internally by grechman)

---

## FLAG: --scope

If `--scope` is set: read `ontology.yaml` → filter entities to those referenced in the given files → print YAML excerpt and EXIT. Do not proceed to interview or generation.

Format:
```
# ONTOLOGY (scoped to: <files>)
_manual: [full _manual block]
_generated:
  stack: [full stack entry]
  entities: [only referenced entities]
```

---

## FLAG: --diff

If `--diff` is set:
1. Read existing `ontology.yaml` — extract and preserve `_manual` block verbatim.
2. Run AUTO-EXTRACTION (Phase 1 below) to regenerate `_generated` block.
3. Merge: new `_generated` + original `_manual` → overwrite `ontology.yaml`.
4. Show diff summary: what changed in `_generated`.
5. Ask: "Any updates to `_manual` while we're here? Y/N" — if Y: go to INTERVIEW (Phase 2).
6. Done.

---

## FLAG: --init (default)

Run Phase 1 (auto-extraction) then Phase 2 (interview), then Phase 3 (write file).

---

## PHASE 1 — AUTO-EXTRACTION

Run silently. Collect everything you can from structured sources. Do NOT ask the user anything yet. Detect what exists — don't assume any specific stack.

### 1a. Stack (from manifest files)

Inspect the project root for **any** of these manifest files. For each one present, parse what it tells you. The list below is exhaustive — do not stop at the first match, multi-language repos are common.

| Manifest | What to extract |
|---|---|
| `go.mod` | Go module path, go version, top direct deps |
| `go.sum` | (presence implies Go) |
| `Cargo.toml` | Crate name, edition, top deps |
| `pyproject.toml` | Project name, Python version, build backend, top deps |
| `requirements.txt` / `Pipfile` / `poetry.lock` / `uv.lock` | Python deps |
| `package.json` | Name, top deps, scripts (build/test/dev), engines |
| `pnpm-lock.yaml` / `yarn.lock` / `bun.lockb` | Node package manager flavour |
| `tsconfig.json` | Target, module, path aliases (only if TS in use) |
| `deno.json` / `deno.jsonc` | Deno tasks and imports |
| `Gemfile` / `Gemfile.lock` | Ruby deps, framework (Rails / Sinatra / etc.) |
| `composer.json` | PHP deps |
| `mix.exs` | Elixir deps |
| `pom.xml` / `build.gradle(.kts)` | JVM project, top deps |
| `*.csproj` / `*.fsproj` / `*.sln` | .NET project, target framework |
| `Package.swift` | Swift deps |
| `pubspec.yaml` | Dart/Flutter deps |
| `CMakeLists.txt` / `Makefile` / `BUILD.bazel` | Build system |
| `Dockerfile` / `docker-compose.yml` / `compose.yaml` | Runtime, base images, exposed services |
| `Procfile` / `fly.toml` / `railway.toml` / `vercel.json` / `netlify.toml` / `app.yaml` / `serverless.yml` / `wrangler.toml` | Hosting target |
| `.tool-versions` / `.nvmrc` / `.python-version` / `.ruby-version` / `.go-version` | Pinned runtime versions |
| `.env.example` / `.env.sample` / `.env.local` | Env var **names** only (NEVER read values) |

Build `_generated.stack` as a flat object reflecting whatever was actually found. Do not invent fields for things that aren't there. Schema is open — typical keys: `languages`, `frameworks`, `runtimes`, `package_managers`, `build_system`, `hosting`, `containerization`, `env_vars`. Omit any key that has no value.

### 1b. Database / persistence (detect from files + tools)

Scan the project for persistence signals. Do not assume a particular database.

File-based detection (pick whatever applies):
- `prisma/schema.prisma` → Prisma + provider listed inside
- `drizzle.config.*` → Drizzle ORM
- `db/schema.rb`, `db/structure.sql` → Active Record
- `alembic.ini`, `alembic/versions/` → SQLAlchemy + Alembic
- `migrations/`, `db/migrate/`, `supabase/migrations/`, `prisma/migrations/` → migrations directory
- `*.sql` files in `schema/`, `sql/`, `db/` → raw SQL schema
- `ent/schema/` → Ent (Go)
- `models.py` / `models/*.py` with Django ORM → Django models
- `gorm.io` import in Go files → GORM
- `sqlc.yaml` → sqlc-generated Go DB layer
- `.dbmlrc` → DBML

Tool-based detection — use whatever **MCP tools are actually connected** in the current session. Examples (use only if available, do NOT install anything):
- A Postgres / Supabase / MySQL / SQLite / Mongo MCP → list tables, columns, FKs
- An ORM-aware tool (Prisma MCP, etc.) → list models

Build `_generated.entities` from what was found. If nothing structured can be extracted, write `entities: # not detected — add manually in _manual or run --diff after schema is in place` and move on. Do not block the run.

### 1c. Dependency graph (if depwire CLI available)

Try: `depwire --version 2>/dev/null` or `npx depwire-cli --version 2>/dev/null`. If either resolves (exit 0):

1. Run `depwire parse . --pretty --stats` (or `npx depwire-cli parse . --pretty --stats`) → captures the dependency graph as JSON.
2. Save raw output to `.depwire/graph.json` (gitignore it).
3. Extract from the JSON:
   - Top-level modules/directories and their outbound dependency counts
   - Files with highest fan-in (most depended on) — load-bearing files
   - Circular dependency chains if any
4. Build `_generated.dependencies`:
   ```yaml
   dependencies:
     tool: "depwire-cli"
     parsed_at: "<YYYY-MM-DD>"
     total_symbols: N
     total_edges: N
     load_bearing_files:
       - path: "<repo-relative path>"
         fan_in: 23
     circular_deps: []  # or list of chains
   ```

If depwire is not available: skip this section entirely. Do not nag the user to install it.

### 1d. Architecture decisions (if ADRs exist)

Check for ADR directories: `doc/adr/`, `docs/adr/`, `adr/`, `decisions/`, `architecture/decisions/`. If any exist:
- List all `*.md` files
- For each: extract title (first H1) and status field if present (`Accepted` / `Superseded` / `Deprecated` / `Proposed`)
- Build `_generated.decisions_index` as a list

### 1e. Show extraction summary

Output a clear summary of what was found:
```
Auto-extraction complete:
  Languages/runtimes: [list or "none detected"]
  Frameworks: [list or "none detected"]
  Build/package: [list or "none detected"]
  Hosting: [list or "none detected"]
  Persistence: [N entities found / "not detected"]
  Dependencies: [N symbols, N edges / "depwire not installed"]
  ADRs: [N decisions found / "none"]
```

Then proceed to Phase 2.

---

## PHASE 2 — INTERVIEW

Ask questions ONE AT A TIME using AskUserQuestion. Wait for each answer before asking the next.

Before asking, explain what `_manual` is:
> "Now I'll ask you a few questions to capture the knowledge that can't be extracted from code — conventions, decisions, and lessons learned. These go into the `_manual` block and are injected into every future grechman session."

### Question sequence:

**Q1 — Conventions**
Ask: "What are the key conventions agents must follow in this codebase? Give as many as you like — one per line. Generic examples: 'All HTTP handlers go through middleware X', 'Domain logic stays out of transport layer'. Type 'done' or 'skip' when finished."

Accept multi-line input. Parse into a list. If "skip" → conventions: [].

**Q2 — Architectural decisions**
Ask: "What architectural decisions have been made that agents should know? Format: 'We use X instead of Y because Z'. Type 'done' or 'skip' when finished."

Generic examples to show: "We use CQRS for the order service — read and write paths have different scaling needs", "We picked Postgres over Mongo — relational integrity matters here"

**Q3 — Rejected approaches**
Ask: "Have you tried any approaches that didn't work and shouldn't be tried again? One per line, format: 'Tried X — failed because Y'. Type 'done' or 'skip' when finished."

Parse into `rejected_approaches` list with `approach` and `reason` fields. Add `session: <today's date>`.

**Q4 — Constraints**
Ask: "Any hard constraints agents must never violate? (e.g. 'Never bypass the migration system', 'Never log PII in plaintext'). Type 'done' or 'skip' when finished."

**Q5 — Confirmation**
Show the full YAML that will be written. Ask: "Does this look right? Y to write, E to edit anything."

If E: ask what to change → apply → show again → ask again.

---

## PHASE 3 — WRITE FILE

Write `ontology.yaml` to project root with this exact structure (the example values below are illustrative — actual content reflects the detected stack):

```yaml
# ontology.yaml — Grechman Ontology System
# Generated: <YYYY-MM-DD>
#
# _generated: AUTO-EXTRACTED — overwritten by /grechman-ontology --diff
# _manual: APPEND-ONLY — written by agents and user, never overwritten by tooling
#
# Usage: /grechman reads this file in Step 0o and injects it into KNOWLEDGE BLOCK.
# Update: run /grechman-ontology --diff after schema/manifest changes.
#         run /grechman-ontology --init to rebuild from scratch.

_generated:
  last_updated: <YYYY-MM-DD>
  stack:
    # [populated from whatever manifest files were found]
  entities:
    # [populated from schema/migrations/MCP if available, otherwise empty]
  dependencies:
    # [populated from depwire CLI if installed, otherwise omitted]
  state:
    # [migrations applied, deploy targets, etc., if available]

_manual:
  conventions:
    # [from Q1 — one string per item]
  decisions:
    # [from Q2 — one string per item]
  rejected_approaches:
    # [from Q3 — structured objects]
  constraints:
    # [from Q4 — one string per item]
```

After writing:
- Output: `ontology.yaml written. Run /grechman-ontology --diff to update after manifest or schema changes.`
- If depwire available: `Tip: depwire is installed — grechman will use it to scope ontology entities to each task's files.`

---

## ONTOLOGY YAML SCHEMA REFERENCE (illustrative — substitute the real stack)

The example below uses a Go + Postgres + Fly.io project. If the project is Rust + SQLite + a Lambda, the keys and values change accordingly. Schema is open — never invent fields that don't apply.

```yaml
_generated:
  last_updated: "2026-04-29"
  stack:
    languages: ["go@1.22"]
    frameworks: ["chi", "ent"]
    runtimes: ["go@1.22"]
    package_managers: ["go modules"]
    build_system: ["make"]
    persistence: ["postgres@15"]
    hosting: ["fly.io"]
    containerization: ["docker"]
    env_vars: ["DATABASE_URL", "SENTRY_DSN", "JWT_SECRET"]
  entities:
    User:
      source: "ent/schema/user.go"
      relations:
        - has_many: Order
    Order:
      source: "ent/schema/order.go"
      constraints:
        - "INSERT only via service layer (orders.Create)"
  dependencies:
    tool: "depwire-cli"
    parsed_at: "2026-04-29"
    total_symbols: 247
    total_edges: 89
    load_bearing_files:
      - path: "internal/db/db.go"
        fan_in: 23
      - path: "internal/auth/auth.go"
        fan_in: 18
    circular_deps: []
  state:
    migrations_applied: 14
    last_migration: "20260420_add_order_status"
    deploy_targets: ["fly.io app: msannikov-api"]

_manual:
  conventions:
    - "Domain logic stays in internal/ — never in cmd/"  # session: 2026-04-29
    - "All DB writes go through the service layer, no handler hits the repo directly"
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

## RULES

- Never read `.env` values — only key names from `.env.example` / `.env.sample` / `.env.local`.
- Never overwrite `_manual` block during `--diff`. Read it, preserve it byte-for-byte.
- Never create ontology.yaml during a grechman session — only grechman agents APPEND to it (on BLOCKED or success).
- `--scope` is read-only: never write anything, just print and exit.
- If `ontology.yaml` already exists and `--init` is run: warn the user that `_manual` will be preserved but confirm before overwriting `_generated`.
- Do not assume any specific framework, database, hosting provider, or language. Detect from files; ask the user if unclear.
