# Grechman — Automated Dev Workflow (medium/hard tasks only)

You are a thin orchestrator. Dispatch agents for each step — never load heavy skills yourself. Agents read their own instruction files, invoke their own skills, and return structured inline reports. You track state, pass decision context, and make routing decisions.

## Arguments from user:
$ARGUMENTS

---

## Parse Arguments

| Param | Default | Values |
|---|---|---|
| `--complexity` | `medium` | `medium` / `hard` |
| `--git` | `on` | `on` / `off` |
| `--github` | `off` | `on` / `off` (needs `--git on` + remote) |
| `--pre-specified` | `off` | `on` — skip brainstorming |
| `--planning` | auto | `local` / `ultra` / `auto` |
| `--budget` | medium=15 / hard=25 | total Agent dispatches |
| `--ontology` | `off` | `on` — run `/grechman-ontology --diff` before planning |
| `--resume` | off | path to grechman-fallback.md |

`--planning auto` resolves to: `ultra` if complexity=hard, `local` if complexity=medium.

If `--github on` + no remote: warn + set off.

---

## State Block

Maintain this EXACTLY between every turn. Update after each agent return:

```
GRECHMAN: step=<name> | complexity=<X> | planning=<local|ultra> | git=<X> | github=<X> | ontology=<on|off> | budget=<used>/<total> | mode=<pending|ralph|sequential> | branch=<X> | vcs=<jujutsu|git> | steps=<done>/<total> | last_sha=<X>
```

---

## Decision Log

Maintain a running list of key decisions from coding agents. Append after each coding step returns. Pass the full log to the next coding agent as `decision_context`. Max 2-3 sentences per entry.

```
DECISIONS:
- Step N: "<what was done, key choices made, gotchas found>"
- Step M: "<what was done, key choices made, gotchas found>"
```

This prevents agents from re-discovering things or making contradictory choices.

---

## Step Sequence

| # | Step | Instruction file | Skip when |
|---|---|---|---|
| 0 | Setup | `.claude/grechman/steps/00-setup.md` | never |
| 0o | Ontology Refresh | inline (see below) | `--ontology off` OR `--resume` |
| 1 | Planning | `.claude/grechman/steps/01-planning.md` | `--resume` OR `planning=ultra` |
| 1U | Ultraplan Handoff | `.claude/grechman/steps/01u-ultraplan-handoff.md` | `planning=local` OR `--resume` |
| 2 | Dispatch | `.claude/grechman/steps/02-dispatch.md` | `--resume` (read existing dispatch.md) |
| 3…N | Coding | `.claude/grechman/steps/03-agent-contract.md` | — |
| R | Review | `.claude/grechman/steps/04-review.md` | — |
| F | Finish | `.claude/grechman/steps/05-finish.md` | — |

---

## Agent Dispatch Protocol

For each step, dispatch `Agent(general-purpose)` with this prompt structure:

```
Read your instructions: .claude/grechman/steps/<file>.md

[GRECHMAN STATE block]
[Step-specific context: paths, flags, fields from previous returns]
[DECISIONS block — for coding agents only]
```

Never include instruction file contents in the prompt — agent reads them itself.

Infrastructure agents (setup, planning, dispatch) return structured inline messages — parse them directly.
Coding agents write YAML to `.grechman/task-reports/task_<N>.yaml`. Read ONLY: `status`, `commits`, `decision`, `blockers`.
After reading a coding agent's YAML `decision` field, append it to the DECISIONS log.

---

## Step 0o — Ontology Refresh (if `--ontology on`)

After setup completes, before planning:
1. Run `/grechman-ontology --diff` (invokes the skill inline — not a subagent dispatch, does NOT count against budget)
2. This refreshes `_generated` block from current project state (stack, entities, depwire dependencies) while preserving `_manual`
3. After completion, set `ontology=on` in state block
4. Pass `ontology=on` to all subsequent agent dispatches

If `ontology.yaml` didn't exist before and `--ontology on`: run `/grechman-ontology --init` instead (will interview user).

---

## Ultraplan User Handoff (after step 1U returns)

When step 1U returns `ULTRAPLAN READY: prompt=<path>`:
1. Print the prompt file contents to the user.
2. `AskUserQuestion`: "Run ultraplan with the prompt above. When done, choose 'teleport back to terminal' and save the plan. Then give me the plan path."
   - **Plan is ready** — user provides plan path
   - **Switch to local planning** — fall back to step 1
3. Read the plan file. Extract step count and libraries list. Continue to Scope Gate.

---

## Scope Gate (after Planning or Ultraplan returns)

`step_count × 1.5 ≤ budget` — if fails: show counts + options:
- A: split task now/later
- B: raise --budget
- C: trim plan

Wait for user choice.

**Medium complexity + within budget:** auto-proceed. Print `Auto-proceeding: mode=[X] steps=[N] budget=[N]` and continue.
**Hard complexity OR budget tight:** ask user to confirm: `Mode: [X] · Steps: [N] · Budget: [N] — Proceed? Y/N/adjust`

---

## Execution Loop

### RALPH_LOOP (mode=ralph)

One agent per step. For each step:
1. Dispatch: `"Read 03-agent-contract.md. Plan: [path] → step [N]. Branch: [X]. VCS: [X]. Last SHA: [X]. Decision context: [DECISIONS block]"`
2. Read YAML → update state (last_sha, steps done) → append decision to DECISIONS
3. If `status: blocked` → one retry with approach hint + DECISIONS → if still blocked → dispatch fallback agent
4. If `status: partial` → merge remaining into next step prompt

### SEQUENTIAL_SUBAGENTS (mode=sequential)

One agent per phase. Phases defined in `grechman-dispatch.md`.

For each phase:
1. Write `grechman-handoff.md`: phase from/to · HEAD_COMMIT · remaining steps (exact from dispatch.md) · DECISIONS block
2. Dispatch: `"Read 03-agent-contract.md. Plan: [path] → phase [N], steps [X-Y]. Branch: [X]. VCS: [X]. Handoff: grechman-handoff.md. Dispatch SHA: [X]. Decision context: [DECISIONS block]"`
3. Read YAML → update state → append decisions
4. On failure:
   - Independent phase: merge successes, retry failed as RALPH
   - Dependent phase: reset to dispatch SHA (use detected VCS) → dispatch fallback agent
   - Partial (safe_to_merge=true): merge + prepend remaining to next phase

After all phases: full integration verification (dispatch one test agent).

---

## Budget & Fallback

Each Agent dispatch = 1 budget unit.
Budget exhausted + steps remain → dispatch agent with `.claude/grechman/steps/fallback.md`.
Agent returns BLOCKED twice for same step → dispatch agent with `.claude/grechman/steps/fallback.md`.

---

## Resume (`--resume`)

1. Step 0 (setup) — preflight only, skip VCS setup
2. Read grechman-fallback.md → extract state + DECISIONS
3. Dispatch dispatch agent (step 2) with re-check flag
4. Read existing grechman-dispatch.md → validate remaining steps vs budget
5. Continue execution loop from blocked step

---

## Post-Execution

After all coding steps complete:
1. Dispatch review agent (`04-review.md`) with `iteration=1, previous_issues=[]`
2. If `status=issues_remaining` → re-dispatch with `iteration+1` and `unresolved` as `previous_issues` (max 3)
3. If `status=recurring` → alert user immediately via `AskUserQuestion` with the recurring issue details, wait for guidance
4. If `status=pass` or `status=issues_fixed` → proceed to finish
5. After 3 iterations with unresolved issues → alert user, proceed to finish
6. Dispatch finish agent (`05-finish.md`)
7. Report to user: summary, PR URL if applicable, discovered issues
