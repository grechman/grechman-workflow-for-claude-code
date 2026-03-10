# Grechman Step 1 — Planning

You are a Grechman agent. Do everything below, then write your report.

## Inputs (from orchestrator prompt)
- `--pre-specified`: on | off
- Project root path
- CLAUDE.md path
- Ontology loaded: true/false
- Memory context (if any, one-line from preflight)

## Procedure

### 1. Sequential Thinking (if MCP available)
If Sequential Thinking MCP is available: run it first with the task description. Pass output to brainstorming/planning below.

### 2. Brainstorming (unless --pre-specified on)
Invoke `superpowers:brainstorming`. Save design doc to `docs/plans/YYYY-MM-DD-<topic>-design.md`.

### 3. Scope Estimate
List implementation steps in one sentence each. Count them. Write count to report — orchestrator will check budget gate.

### 4. Write Plan
Ensure `docs/plans/` exists. Invoke `superpowers:writing-plans`. Save to `docs/plans/YYYY-MM-DD-<topic>.md`.

### 5. Detect Libraries
From the plan, extract all libraries/frameworks that coding agents will need docs for. List in report.

### 6. MCP Memory (if available)
Save entities: Project, Task, Decision, Constraint with relations Project→Task→Decision, Project→Constraint.

## Report

Write to `.grechman/task-reports/task_1.yaml`:
```yaml
task_id: 1
status: completed | blocked
plan_path: <path>
design_doc_path: <path or null>
step_count: <N>
libraries_detected: [list]
blockers: <if any, else null>
```

Return to orchestrator ONLY: `"Task 1 complete. Report: .grechman/task-reports/task_1.yaml"`
