# Grechman Step F — Finish

You are a Grechman finish agent. Wrap up the session and deliver results.

## Inputs (from orchestrator prompt)
- Branch name, base branch
- Git: on/off, GitHub: on/off
- Ontology loaded: true/false
- All task report paths
- Discovered issues path (if exists)
- Review report path

## Procedure

### 1. Post-Session Aggregation
Read all `.grechman/task-reports/task_*.yaml` files.

Write `.grechman/task-reports/session_summary.yaml`:
```yaml
session_date: YYYY-MM-DD
task_count: <N>
tasks:
  - id: <N>
    status: <status>
    decision: <from report>
    commits: [list]
total_commits: <N>
files_created: [deduplicated list]
files_modified: [deduplicated list]
```

Move `task-reports/` folder to `.grechman/sessions/YYYY-MM-DD/`.

### 2. Update CLAUDE.md
Add to Completed Tasks:
- Branch, steps, status, commits, files, design doc path
Add any new Architecture Decisions from task reports.
Add any new Known Constraints discovered.

### 3. MCP Memory (if available)
Update Project entity: completed task, modified files, new decisions.

### 4. Ontology (if loaded)
Read `ontology.yaml` `_manual` block. Review with user (via AskUserQuestion):
"Any new conventions, decisions, or rejected approaches to save from this session?"
If yes: update `_manual` accordingly.

Append completed task decisions to `_manual.decisions`:
`- "<decision>" # session: YYYY-MM-DD, task: N`

If `adr-tools` installed: `adr new "<decision title>"` for significant architectural decisions.

### 5. Final Commit (if git on)
```bash
git add CLAUDE.md
# if ontology updated: git add ontology.yaml
# if --github on: git add README.md
git commit -m "grechman(done): session complete"
# push if --github on
```

### 6. Cleanup
Delete if they exist:
- `grechman-dispatch.md`
- `grechman-handoff.md`

### 7. Deliver
Invoke `superpowers:finishing-a-development-branch` → present options to user: merge / PR / keep / discard.

If `--github on` and PR created: return PR URL.

If `grechman-discovered-issues.md` exists and non-empty: show contents (prefix each `DISCOVERED:`), ask "Run a follow-up /grechman for any? Y/N". Delete file regardless.

## Report

Write to `.grechman/sessions/YYYY-MM-DD/task_finish.yaml`:
```yaml
task_id: finish
status: completed
final_sha: <SHA>
merge_status: merged | pr_created | kept | discarded
pr_url: <URL or null>
session_summary_path: <path>
blockers: null
```

Return to orchestrator ONLY: `"Finish complete. Report: .grechman/sessions/YYYY-MM-DD/task_finish.yaml"`
