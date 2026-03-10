# Grechman Step 4 — Dispatch Setup

You are a Grechman agent. Do everything below, then write your report.

## Inputs (from orchestrator prompt)
- Plan path
- Step count
- Budget
- Knowledge block path

## Procedure

### 1. Analyze Dependencies
Read the plan. For each step, identify:
- Files it creates
- Files it modifies
- Which other steps it depends on

### 2. Choose Mode

**SEQUENTIAL_SUBAGENTS** — choose when ANY of:
- 3+ ordered stages with distinct handoff artifacts (schema → models → API → tests)
- Total steps > 8
- Crossing work domains (DB / API / UI / tests)

**RALPH_LOOP** (default) — choose when:
- Tightly coupled steps
- ≤ 8 steps
- Single work domain
- No natural phase boundary

### 3. Write grechman-dispatch.md

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

### 4. Preamble (SEQUENTIAL only)
If plan has setup steps (scaffolding, schema, env) that ALL phases depend on: mark them as Phase 0 in dispatch.md.

## Report

Write to `.grechman/task-reports/task_4.yaml`:
```yaml
task_id: 4
status: completed | blocked
mode: ralph | sequential
dispatch_path: grechman-dispatch.md
phase_count: <N or null>
step_count: <N>
preamble_steps: [list or null]
blockers: <if any, else null>
```

Return to orchestrator ONLY: `"Task 4 complete. Report: .grechman/task-reports/task_4.yaml"`
