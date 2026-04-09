# Grechman Step 2 — Knowledge & Dispatch

You are a Grechman agent. Fetch library docs (if needed), build the knowledge block, and set up the dispatch plan.

## Inputs (from orchestrator prompt)
- Plan path, step count, budget
- Libraries detected (from planning — may be empty)
- Ontology loaded: true/false
- Depwire available: true/false
- Qodo rules path (if any)
- Project root path
- Git: on/off

## Procedure

### 1. Library Docs (SKIP entirely if libraries list is empty)
For each library in the list:
- Check if `knowledge.md` exists and has entry for this library
- If entry exists and `date_queried` < 90 days old → use as-is
- If version-pinned → always use existing entry
- If missing or stale → use `resolve-library-id` (context7 MCP) → `query-docs` → append to knowledge.md

Format per library in knowledge.md:
```
## <Library>
- date_queried: YYYY-MM-DD
- version: X.Y.Z
- key_apis: [list]
- gotchas: [list]
```

If knowledge.md > 200 lines: archive unused entries to `knowledge-archive.md`.

### 2. Ontology Scoping (SKIP if ontology not loaded)
Read `ontology.yaml`. Build scoped excerpt:
- If `_generated.dependencies` exists: use `load_bearing_files` to identify high-risk files. Cross-reference with task files — if a task touches a load-bearing file, flag it.
- Include all entities if <10, otherwise only entities matching task description/files
- Full `_manual` block always included

### 3. Build Knowledge Block
Write `.grechman/knowledge-block.md` — this file is read by ALL coding agents. Contents:
- Library docs (if any were fetched — omit section entirely if none)
- Ontology excerpt (if available — omit section entirely if not)
- Qodo rules summary (if available)
- If ontology has `dependencies.load_bearing_files`: include a DEPENDENCY AWARENESS section:
  ```
  HIGH FAN-IN FILES (many dependents — changes here have wide blast radius):
  - <path>: <fan_in> dependents
  Before modifying these files: verify no dependents break. If unsure, flag in blockers.
  Before deleting a file referenced in load_bearing_files: do NOT delete without resolving all dependents.
  ```

If no libraries, no ontology, and no qodo: write a minimal knowledge-block.md with just the project root path and a note that no external context was needed for this task.

### 4. Analyze Dependencies
Read the plan. For each step, identify:
- Files it creates
- Files it modifies
- Which other steps it depends on

### 5. Choose Mode

**SEQUENTIAL_SUBAGENTS** — choose when ANY of:
- 3+ ordered stages with distinct handoff artifacts (schema → models → API → tests)
- Total steps > 8
- Crossing work domains (DB / API / UI / tests)

**RALPH_LOOP** (default) — choose when:
- Tightly coupled steps
- ≤ 8 steps
- Single work domain
- No natural phase boundary

### 6. Write grechman-dispatch.md

```markdown
# Grechman Dispatch

Mode: <ralph | sequential>
Plan: <path>
Budget: <N>

## Steps (RALPH) / Phases (SEQUENTIAL)

### Step/Phase N
- Steps: [list of step numbers]
- Files create: [list]
- Files modify: [list]
- Depends on: [step/phase numbers]
```

For SEQUENTIAL: group steps into phases. Ensure no overlapping file sets between phases.

If plan has setup steps (scaffolding, schema, env) that ALL phases depend on: mark them as Phase 0.

### 7. Update CLAUDE.md
Add `Libraries: [list]` to session entry (or `Libraries: none` if empty).

### 8. Commit (if git on)
Detect VCS (jj or git). Stage knowledge.md (if exists), knowledge-block.md, CLAUDE.md.
```bash
# jujutsu
jj commit -m "grechman(knowledge): load context"
# git
git add CLAUDE.md knowledge.md .grechman/knowledge-block.md && git commit -m "grechman(knowledge): load context"
```

## Return

Return ONE line to orchestrator:
```
DISPATCH COMPLETE: mode=<ralph|sequential> phases=<N|null> steps=<N> knowledge=<true|false>
```

Or if blocked:
```
DISPATCH BLOCKED: <reason>
```

Do NOT write YAML task reports. Do NOT return anything beyond the single status line.
