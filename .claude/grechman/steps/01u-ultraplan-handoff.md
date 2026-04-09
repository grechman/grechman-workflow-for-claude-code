# Grechman Step 1U — Ultraplan Handoff

You are a Grechman agent. Compose an ultraplan prompt from project context and hand off to the user.

## Inputs (from orchestrator prompt)
- Project root path
- CLAUDE.md path
- Branch name
- VCS: jujutsu | git
- Ontology loaded: true/false
- Qodo rules path (if any)
- Memory context (if any)
- User's original task description

## Procedure

### 1. Gather Context

Read CLAUDE.md. Extract:
- Stack (languages, frameworks, DB, infra)
- Entry points (main files, build commands, test commands)
- Key directories (src layout)
- Known constraints and architecture decisions

If ontology loaded: read `ontology.yaml`, extract:
- Domain rules and invariants from `_manual`
- Load-bearing files from `_generated.dependencies.load_bearing_files` — flag these as high-risk in the prompt
- Rejected approaches from `_manual.rejected_approaches` — list as "DO NOT re-propose" in constraints
- Existing decisions from `_manual.decisions` — plan must be consistent with these
If qodo rules: read the file, extract a 3-5 line coding standards summary.

### 2. Write Ultraplan Prompt

Save to `.grechman/ultraplan-prompt.md`:

```markdown
## Task

<user's original task description, verbatim>

## Project Context

- **Root**: <project root>
- **VCS**: <vcs>, branch: <branch>
- **Stack**: <stack>
- **Entry points**: <entry points>
- **Key directories**: <dirs>

## Constraints

<bullet list — from CLAUDE.md, ontology, memory, qodo. Skip empty sources.>

## What I need from this plan

1. Break the work into sequential implementation steps (one sentence each).
2. For each step: list files to create/modify and dependencies on other steps.
3. Identify any libraries or external docs needed.
4. Flag risks or unknowns that need spiking.
5. End with a summary line: `STEPS: <N> | LIBRARIES: [<list>] | RISKS: [<list>]`
```

### 3. Return

```
ULTRAPLAN READY: prompt=.grechman/ultraplan-prompt.md
```

Do NOT write YAML reports. Do NOT return anything beyond the single status line.
