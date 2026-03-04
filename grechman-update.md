# Grechman Workflow — Update Spec

This file contains all changes to implement in the grechman workflow.
Execute each section in order. Commit after each section.

---

## Section 1: Orchestrator Context Overflow Fix

**Problem:** Subagent verbose output floods orchestrator context window across multiple tasks.

**Changes to `/grechman` orchestrator prompt:**

1. When spawning a subagent for a task, pass these instructions to the subagent:

```
Upon task completion, write ONLY the following to .grechman/task-reports/task_<N>.yaml:

task_id: <N>
status: completed | blocked | partial
commits:
  - <hash>: <message>
files_changed:
  - <list of changed files>
tools_used:
  - <list of tools used>
decision: <one sentence: what was done and how>
blockers: <if any, else null>

Return to orchestrator ONLY: "Task <N> complete. Report: .grechman/task-reports/task_<N>.yaml"
Do NOT return reasoning, code, logs, or any other output.
```

2. Orchestrator reads `.grechman/task-reports/task_<N>.yaml` after each task instead of receiving raw subagent output.

3. After ALL tasks complete, orchestrator must:
   - Read all `task_<N>.yaml` files
   - Write `.grechman/task-reports/session_summary.yaml` with aggregated results
   - Append completed task decisions to `ontology.yaml` under `_manual.decisions`
   - Move entire `task-reports/` folder to `.grechman/sessions/<YYYY-MM-DD>/`

**Create the following:**
- Folder: `.grechman/task-reports/` (add to `.gitignore`)
- Folder: `.grechman/sessions/` (add to `.gitignore`)

**Acceptance criteria:** Orchestrator context window stays flat after each task completion. No manual compact needed.

---

## Section 2: Depwire Integration — File Change Hook

**Problem:** Agents edit files without knowing what depends on them, causing silent breakage in other files.

**Changes to `/grechman` subagent instructions:**

Add a mandatory pre-edit step. Before any subagent edits or creates a file, it must:

1. Call depwire `get_architecture_summary` for the target file
2. Read the dependency list returned
3. Add affected files to its working context
4. Proceed with the edit only after reviewing dependencies

Add this block to subagent KNOWLEDGE BLOCK injection:

```
PRE-EDIT RULE (mandatory):
Before editing any file:
1. Call depwire: get_architecture_summary for <target_file>
2. Review what depends on this file and what this file depends on
3. Include affected files in your context
4. If your change breaks a dependency — fix it in the same task or flag it in blockers field of your task report

Before creating a new file:
1. Check depwire if similar module already exists
2. After creation — depwire will auto-update the graph (no manual action needed)

Before deleting a file:
1. Call depwire to check what depends on this file
2. If dependents exist — do NOT delete without resolving dependencies first
3. Document the deletion decision in your task report under `decision` field
```

**Important:** Do NOT write dependency information manually into `ontology.yaml`.
- `ontology.yaml` = human decisions, conventions, architectural choices
- depwire = live dependency graph from actual code (always up to date automatically)

---

## Section 3: Grechman Ontology — Session Integration

**Changes to `/grechman-ontology` prompt:**

Add a new flag: `--session-import`

```
--session-import <path>   Import decisions from a session_summary.yaml into _manual.decisions
```

When `--session-import` is called:
1. Read the specified `session_summary.yaml`
2. Extract all `decision` fields from completed tasks
3. Append each decision to `_manual.decisions` in `ontology.yaml` with format:
   ```yaml
   - "<decision text>"  # session: <YYYY-MM-DD>, task: <N>
   ```
4. Preserve all existing `_manual` content byte-for-byte
5. Output: "Imported N decisions from session <date> into ontology.yaml"

This keeps ontology updated without manual intervention after each work session.

---

## Section 4: .gitignore Updates

Add to `.gitignore`:

```
.grechman/task-reports/
.grechman/sessions/
```

---

## Acceptance Criteria (Full)

- [ ] Orchestrator handles 14 sequential tasks without context overflow
- [ ] Each subagent starts with clean context + scoped ontology
- [ ] Each subagent writes structured report to `.grechman/task-reports/task_<N>.yaml`
- [ ] Orchestrator context grows by ~500 tokens per task maximum
- [ ] Before any file edit — depwire is called automatically
- [ ] Broken dependencies are caught before commit, not after
- [ ] Session decisions flow into ontology.yaml via `--session-import`
- [ ] No manual compact needed on orchestrator
- [ ] Working directory stays clean between sessions
