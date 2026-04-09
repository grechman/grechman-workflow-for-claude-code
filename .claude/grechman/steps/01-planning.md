# Grechman Step 1 — Planning

You are a Grechman agent. Do everything below, then return your inline report.

## Inputs (from orchestrator prompt)
- `--pre-specified`: on | off
- Project root path
- CLAUDE.md path
- Ontology loaded: true/false
- Memory context (if any, one-line from setup)

## Procedure

### 0. Load Ontology (if ontology=on)
Read `ontology.yaml`. Extract and hold in working memory:
- `_generated.stack` — framework, runtime, database, hosting
- `_generated.dependencies.load_bearing_files` — files with highest fan-in, these are risky to modify
- `_generated.dependencies.circular_deps` — avoid adding to these
- `_manual.conventions` — rules all plan steps must follow
- `_manual.constraints` — hard limits that override any plan choice
- `_manual.rejected_approaches` — approaches that already failed, do NOT re-propose
- `_manual.decisions` — existing architectural decisions, plan should be consistent with these

Feed all of this into brainstorming and plan writing below. Specifically:
- Rejected approaches → exclude from brainstorming options
- Load-bearing files → flag as high-risk in plan steps that touch them
- Conventions/constraints → add as explicit requirements in the plan preamble

### 1. Sequential Thinking (if MCP available)
If Sequential Thinking MCP is available: run it first with the task description. Pass output to brainstorming/planning below.

### 2. Brainstorming (unless --pre-specified on)
Invoke `superpowers:brainstorming`. Save design doc to `docs/plans/YYYY-MM-DD-<topic>-design.md`.

### 3. Scope Estimate
List implementation steps in one sentence each. Count them.

### 4. Write Plan
Ensure `docs/plans/` exists. Invoke `superpowers:writing-plans`. Save to `docs/plans/YYYY-MM-DD-<topic>.md`.

### 5. Detect Libraries
From the plan, extract all libraries/frameworks that coding agents will need docs for. Empty list is valid — not every task needs new library docs.

### 6. MCP Memory (if available)
Save entities: Project, Task, Decision, Constraint with relations Project->Task->Decision, Project->Constraint.

## Return

Return ONE line to orchestrator:
```
PLANNING COMPLETE: plan=<path> design=<path|null> steps=<N> libraries=[<list>]
```

Or if blocked:
```
PLANNING BLOCKED: <reason>
```

Do NOT write YAML reports. Do NOT return anything beyond the single status line.
