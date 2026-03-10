# Grechman Step 3 — Knowledge

You are a Grechman agent. Do everything below, then write your report.

## Inputs (from orchestrator prompt)
- Libraries list (from planning report)
- Ontology loaded: true/false
- Depwire available: true/false
- Qodo rules path (if any)
- Project root path
- Git: on/off

## Procedure

### 1. Library Docs
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

### 2. Ontology Scoping (if ontology loaded)
Read `ontology.yaml`. Build scoped excerpt:
- If depwire available: call `get_file_context` for task files → include only referenced entities
- If no depwire: include all entities if <10, otherwise only entities matching task description/files
- Full `_manual` block always included

### 3. Architecture Graph (if depwire available)
Call `get_architecture_summary` → store for knowledge block.

### 4. Build Knowledge Block
Write `.grechman/knowledge-block.md` — this file is read by ALL coding agents. Contents:
- Library docs (scoped to this task's libraries)
- Ontology excerpt (if available)
- Architecture graph (if available)
- Qodo rules summary (if available)
- depwire PRE-EDIT RULE (if depwire available):
  ```
  At task start — call depwire get_architecture_summary ONCE for ALL files in your assigned set (CREATE + MODIFY), batched in a single call. Review the combined dependency context.
  Exemptions: test files, config files, newly created files.
  If a change breaks a dependency — fix it or flag in blockers.
  Before deleting a file: verify no dependents; if dependents exist — do NOT delete without resolving.
  ```

### 5. Update CLAUDE.md
Add `Libraries: [list]` to session entry.

### 6. Batch Commit 2 (if git on)
```bash
git add CLAUDE.md knowledge.md .grechman/knowledge-block.md
git commit -m "grechman(knowledge): load library context"
```

## Report

Write to `.grechman/task-reports/task_3.yaml`:
```yaml
task_id: 3
status: completed | blocked
knowledge_md_path: <path>
knowledge_block_path: <path>
libraries_queried:
  - name: <lib>
    version: <ver>
    fresh: true | false
knowledge_sha: <commit SHA or null>
blockers: <if any, else null>
```

Return to orchestrator ONLY: `"Task 3 complete. Report: .grechman/task-reports/task_3.yaml"`
