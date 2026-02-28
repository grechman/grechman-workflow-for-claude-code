# /grechman-ontology — Project Ontology Generator

You are running the Grechman Ontology command. Parse $ARGUMENTS before doing anything.

## Arguments from user:
$ARGUMENTS

---

## WHAT THIS COMMAND DOES

Generates and maintains `ontology.yaml` — a structured, machine-readable model of your project's domain, stack, constraints, and conventions. This file is automatically read by `/grechman` at the start of every session and injected into each agent's KNOWLEDGE BLOCK.

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

Run silently. Collect everything you can from structured sources. Do NOT ask the user anything yet.

### 1a. Stack (from files)

Check for these files in project root and parse:
- `package.json` → framework, libraries, versions (top 5-10 most relevant)
- `tsconfig.json` → target, module, path aliases
- `next.config.*` → if present, note framework = Next.js
- `vercel.json` → if present, note hosting = Vercel
- `.env.example` or `.env.local` → env var names (NOT values — never read values)

Build `_generated.stack` from findings.

### 1b. Database entities (if Supabase MCP available)

If Supabase MCP tools are available:
- List all tables in `public` schema
- For each table: get columns, types, foreign keys, RLS enabled/disabled
- List Edge Functions
- Get applied migrations count + last migration name

Build `_generated.entities` and `_generated.state` from findings.

If Supabase MCP not available: skip this section, note "entities: # Supabase MCP not available — add manually".

### 1c. Dependency graph (if depwire MCP available)

If depwire MCP is available:
- Call `get_architecture_summary` → extract key architectural layers and their relationships

Store for display in Phase 2. Do not put in ontology.yaml directly — it changes too frequently.

### 1d. Architecture decisions (if adr-tools installed)

Check if `doc/adr/` directory exists. If yes:
- List all `*.md` files in `doc/adr/`
- For each: extract title and status (Accepted / Superseded / Deprecated)
- Build list for `_generated.decisions_index`

### 1e. Show extraction summary

Output a clear summary of what was found:
```
Auto-extraction complete:
  Stack: [list what was found or "not found"]
  Entities: [N tables found / "Supabase MCP not available"]
  ADRs: [N decisions found / "doc/adr/ not found"]
  depwire: [available / not available]
```

Then proceed to Phase 2.

---

## PHASE 2 — INTERVIEW

Ask questions ONE AT A TIME using AskUserQuestion. Wait for each answer before asking the next.

Before asking, explain what `_manual` is:
> "Now I'll ask you a few questions to capture the knowledge that can't be extracted from code — conventions, decisions, and lessons learned. These go into the `_manual` block and are injected into every future grechman session."

### Question sequence:

**Q1 — Conventions**
Ask: "What are the key conventions agents must follow in this codebase? Give as many as you like — one per line. Examples: 'Supabase client only in server components', 'All mutations via Edge Functions'. Type 'done' or 'skip' when finished."

Accept multi-line input. Parse into a list. If "skip" → conventions: [].

**Q2 — Architectural decisions**
Ask: "What architectural decisions have been made that agents should know? Format: 'We use X instead of Y because Z'. Type 'done' or 'skip' when finished."

Examples to show: "We use Supabase Auth over NextAuth — native RLS support", "Mutations go through Edge Functions — not direct DB calls from client"

**Q3 — Rejected approaches**
Ask: "Have you tried any approaches that didn't work and shouldn't be tried again? One per line, format: 'Tried X — failed because Y'. Type 'done' or 'skip' when finished."

Parse into `rejected_approaches` list with `approach` and `reason` fields. Add `session: <today's date>`.

**Q4 — Constraints**
Ask: "Any hard constraints agents must never violate? (e.g. 'Never change DB schema without a migration'). Type 'done' or 'skip' when finished."

**Q5 — Confirmation**
Show the full YAML that will be written. Ask: "Does this look right? Y to write, E to edit anything."

If E: ask what to change → apply → show again → ask again.

---

## PHASE 3 — WRITE FILE

Write `ontology.yaml` to project root with this exact structure:

```yaml
# ontology.yaml — Grechman Ontology System
# Generated: <YYYY-MM-DD>
#
# _generated: AUTO-EXTRACTED — overwritten by /grechman-ontology --diff
# _manual: APPEND-ONLY — written by agents and user, never overwritten by tooling
#
# Usage: /grechman reads this file in Step 0d and injects it into KNOWLEDGE BLOCK.
# Update: run /grechman-ontology --diff after schema/package changes.
#         run /grechman-ontology --init to rebuild from scratch.

_generated:
  last_updated: <YYYY-MM-DD>
  stack:
    # [populated from package.json / tsconfig / config files]
  entities:
    # [populated from Supabase MCP or left as placeholder]
  state:
    # [migrations, deploy config if available]

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
- Output: `ontology.yaml written. Run /grechman-ontology --diff to update after schema changes.`
- If depwire available: `Tip: depwire is installed — grechman will use it to scope ontology entities to each task's files.`

---

## ONTOLOGY YAML SCHEMA REFERENCE

```yaml
_generated:
  last_updated: "2026-02-28"
  stack:
    framework: "next@14.2.0"
    runtime: "nodejs@20"
    database: "supabase-postgres"
    auth: "supabase-auth"
    hosting: "vercel"
    ui: ["tailwind@3", "shadcn-ui@latest"]
  entities:
    User:
      source: "auth.users"
      rls: true
      relations:
        - has_many: Post
    Post:
      source: "public.posts"
      rls: true
      constraints:
        - "INSERT only via Edge Function post-create"
  state:
    migrations_applied: 14
    last_migration: "20260228_add_comments"
    edge_functions: ["post-create", "user-delete"]

_manual:
  conventions:
    - "Supabase client only in server components — never client components"  # step 2, 2026-02-28
    - "All mutations via Edge Functions — no direct INSERT from client"
  decisions:
    - "Use Supabase Auth over NextAuth — native RLS support (adr: 0001)"
  rejected_approaches:
    - step: 3
      session: "2026-02-28"
      approach: "Direct Postgres trigger for post creation"
      reason: "Cannot add business logic in trigger — moved to Edge Function"
  constraints:
    - "Never change DB schema without a migration file"
    - "Never expose service_role key to client"
```

---

## RULES

- Never read `.env` values — only `.env.example` or `.env.local` key names.
- Never overwrite `_manual` block during `--diff`. Read it, preserve it byte-for-byte.
- Never create ontology.yaml during a grechman session — only grechman agents APPEND to it (on BLOCKED or success).
- `--scope` is read-only: never write anything, just print and exit.
- If `ontology.yaml` already exists and `--init` is run: warn the user that `_manual` will be preserved but confirm before overwriting `_generated`.
