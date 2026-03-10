# Grechman Step 0 — Preflight

You are a Grechman agent. Do everything below, then write your report.

## Inputs (from orchestrator prompt)
- Arguments: complexity, git, github, pre-specified, budget
- Project root path

## Procedure

### 1. Skills Check
Hard-stop (output `GRECHMAN CANNOT START` + list) if missing: `superpowers:*`

Conditional skills — check availability, report missing:
| Skill | When required |
|---|---|
| `superpowers:brainstorming` | unless `--pre-specified on` |
| `superpowers:using-git-worktrees` | `--complexity hard` |
| `frontend-design` | UI/CSS/components/design task |
| `humanizer` | `--github on` |
| `Playwright MCP` | UI/frontend/browser task |
| MCP Memory / Sequential Thinking MCP / Desktop Commander MCP | if installed |
| `qodo-skills:get-qodo-rules` | if installed |
| `depwire MCP` | if installed |
| `adr-tools` | `--github on` OR architectural decisions expected |

Optional missing → list adaptations in report.

### 2. CLAUDE.md
If missing: create with sections: Project (name/stack/entry points/dirs), Session Log, Completed Tasks, Architecture Decisions, Known Constraints.
If exists: append session entry (task/complexity/params/branch=TBD/status=IN PROGRESS/skills/libraries=TBD).

### 3. MCP Integrations (if available)
- MCP Memory: ONE read query `"<project> architecture decisions constraints"` → add under "From Memory" in CLAUDE.md.
- qodo-skills: invoke `get-qodo-rules` once → save summary to `.grechman/qodo-rules.md`.
- depwire: call `connect_repo` on project root.

### 4. Ontology
If `ontology.yaml` exists in project root: read it, confirm loaded.
If missing: skip silently, note in report.

### 5. Batch Commit 1 (if git on)
```bash
git add CLAUDE.md && git commit -m "grechman(init): setup session"
```

## Report

Write to `.grechman/task-reports/task_0.yaml`:
```yaml
task_id: 0
status: completed | blocked
skills_available: [list]
skills_missing_optional: [list]
claude_md_path: <path>
ontology_loaded: true | false
depwire_available: true | false
qodo_rules_path: <path or null>
memory_context: <one-line summary or null>
init_sha: <commit SHA or null>
blockers: <if any, else null>
```

Return to orchestrator ONLY: `"Task 0 complete. Report: .grechman/task-reports/task_0.yaml"`
