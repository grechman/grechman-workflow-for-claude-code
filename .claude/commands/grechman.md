# Grechman — Automated Dev Workflow (medium/hard tasks only)

You are a thin orchestrator. Dispatch agents for each step — never load heavy skills yourself. Agents read their own instruction files, invoke their own skills, and return minimal YAML reports. You track state and make routing decisions.

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
| `--budget` | medium=15 / hard=25 | total Agent dispatches |
| `--resume` | off | path to grechman-fallback.md |

If `--github on` + no remote: warn + set off.

---

## State Block

Maintain this EXACTLY between every turn. Update after each agent return:

```
GRECHMAN: step=<name> | complexity=<X> | git=<X> | github=<X> | budget=<used>/<total> | mode=<pending|ralph|sequential> | branch=<X> | steps=<done>/<total> | last_sha=<X>
```

---

## Step Sequence

Dispatch agents in this order. Each agent reads its instruction file autonomously.

| # | Step | Instruction file | Skip when |
|---|---|---|---|
| 0 | Preflight | `grechman-steps/00-preflight.md` | never |
| 1 | Planning | `grechman-steps/01-planning.md` | `--resume` |
| 2 | Git setup | `grechman-steps/02-git.md` | `--git off` or `--resume` |
| 3 | Knowledge | `grechman-steps/03-knowledge.md` | never |
| 4 | Dispatch setup | `grechman-steps/04-dispatch-setup.md` | `--resume` (read existing dispatch.md) |
| 5…N | Coding steps | `grechman-steps/05-agent-contract.md` | — |
| R | Review | `grechman-steps/06-review.md` | — |
| F | Finish | `grechman-steps/07-finish.md` | — |

---

## Agent Dispatch Protocol

For each step, dispatch `Agent(general-purpose)` with this prompt structure:

```
Read your instructions: .claude/commands/grechman-steps/<file>.md

[GRECHMAN STATE block]
[Step-specific context: paths, flags, previous YAML fields needed]
```

Never include instruction file contents in the prompt — agent reads them itself.
Never carry agent output beyond what you record in state block.
Agents write results to `.grechman/task-reports/task_<N>.yaml`. Read ONLY: `status`, `commits`, `blockers`, and fields you need for next dispatch.

---

## Scope Gate (after Planning returns)

`step_count × 1.5 ≤ budget` — if fails: show counts + options:
- A: split task now/later
- B: raise --budget
- C: trim plan

Wait for user choice. Then ask user to confirm before coding starts:
`Mode: [X] · Steps: [N] · Budget: [N] — Proceed? Y/N/adjust`

---

## Execution Loop

### RALPH_LOOP (mode=ralph)

One agent per step. For each step:
1. Dispatch: `"Read 05-agent-contract.md. Plan: [path] → step [N]. Branch: [X]. Last SHA: [X]."`
2. Read YAML → update state (last_sha, steps done)
3. If `status: blocked` → one retry with approach hint → if still blocked → dispatch fallback agent
4. If `status: partial` → merge remaining into next step prompt

### SEQUENTIAL_SUBAGENTS (mode=sequential)

One agent per phase. Phases defined in `grechman-dispatch.md`.

For each phase:
1. Write `grechman-handoff.md`: phase from/to · HEAD_COMMIT · remaining steps (exact from dispatch.md) · files changed + decisions from previous YAML (if any)
2. Dispatch: `"Read 05-agent-contract.md. Plan: [path] → phase [N], steps [X-Y]. Branch: [X]. Handoff: grechman-handoff.md. Dispatch SHA: [X]."`
3. Read YAML → update state
4. On failure:
   - Independent phase: merge successes, retry failed as RALPH
   - Dependent phase: `git reset --hard $DISPATCH_SHA` → dispatch fallback agent
   - Partial (safe_to_merge=true): merge + prepend remaining to next phase

After all phases: full integration verification (dispatch one test agent).

---

## Budget & Fallback

Each Agent dispatch = 1 budget unit.
Budget exhausted + steps remain → dispatch agent with `grechman-steps/fallback.md`.
Agent returns BLOCKED twice for same step → dispatch agent with `grechman-steps/fallback.md`.

---

## Resume (`--resume`)

1. Step 0 (preflight) only
2. Read grechman-fallback.md → extract state
3. Dispatch knowledge agent (re-check freshness)
4. Read existing grechman-dispatch.md → validate remaining steps vs budget
5. Continue execution loop from blocked step

---

## Post-Execution

After all coding steps complete:
1. Dispatch review agent (`06-review.md`)
2. Dispatch finish agent (`07-finish.md`)
3. Report to user: summary, PR URL if applicable, discovered issues
