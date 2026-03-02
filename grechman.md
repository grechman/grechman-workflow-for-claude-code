# Grechman — Automated Dev Workflow (medium/hard tasks only)

You are running the Grechman workflow. Parse all $ARGUMENTS before doing anything.

## Arguments from user:
$ARGUMENTS

---

## STEP 0 — PREFLIGHT

### 0a. Skills
Hard-stop (output ⛔ GRECHMAN CANNOT START + list + install instructions) if missing: `superpowers:*`

Conditional:
| Skill | When |
|---|---|
| `superpowers:brainstorming` | unless `--pre-specified on` |
| `superpowers:using-git-worktrees` | `--complexity hard` |
| `frontend-design` | UI/CSS/components/design |
| `humanizer` | `--github on` |
| `pr-review-toolkit:review-pr` | reviewing existing PR |
| `Playwright MCP` | UI/frontend/browser |
| MCP Memory / Sequential Thinking MCP / Desktop Commander MCP | if installed |
| `qodo-skills:get-qodo-rules` | if installed — run once in Step 0c, inject rules into all agent prompts |
| `depwire MCP` | dependency graph, impact analysis, entity scoping — if installed |
| `adr-tools` | `--github on` OR architectural decisions made during session — if installed |

Optional missing → list adaptations, Y/N prompt, proceed if Y.

### 0b. Arguments
| Param | Default | Values |
|---|---|---|
| `--complexity` | `medium` | `medium` / `hard` |
| `--git` | `on` | `on` / `off` |
| `--github` | `off` | `on` / `off` (needs `--git on` + remote) |
| `--pre-specified` | `off` | `on` — skip brainstorming |
| `--budget` | medium=15 / hard=25 | total Agent dispatches allowed |
| `--resume` | off | path to grechman-fallback.md → skip to resume path in RULES |

If `--resume`: Step 0a only → resume path. If `--github on` + no remote: warn + set off.
Output: `Task: [X]. Complexity: [X]. Git: [X]. GitHub: [X]. Budget: [N].`

### 0c. CLAUDE.md
If missing: create with sections Project (name/stack/entry points/dirs), Session Log, Completed Tasks, Architecture Decisions, Known Constraints. If exists: append session entry (task/complexity/params/branch=TBD/status=IN PROGRESS/skills/libraries=TBD).
If MCP Memory: ONE read query `"<project> architecture decisions constraints"` → add under "From Memory". (Read-only; Step 2 writes are fine.) Do NOT commit yet.
If `qodo-skills:get-qodo-rules`: invoke once → save rules summary; inject into KNOWLEDGE BLOCK in Step 4.

### 0d. Ontology
If `ontology.yaml` exists in project root: read it → store for Step 4 injection. Missing → skip silently.
If depwire MCP: call `connect_repo` on project root → available for Step 4 entity scoping.

---

## STEP 1 — SCOPE ESTIMATE

Ask: "List implementation steps in one sentence each — rough only."
Rule: `steps × 1.5 ≤ budget`. On fail: output ⚠️ with counts + options A (split now/later) / B (raise --budget) / C (trim plan). Wait for choice.

---

## STEP 2 — BRAINSTORMING + PLANNING

Ensure `docs/plans/` exists. If Sequential Thinking MCP: run first → pass output to brainstorming (or writing-plans if `--pre-specified on`).
If `--pre-specified off`: `superpowers:brainstorming` → save design doc to `docs/plans/YYYY-MM-DD-<topic>-design.md`.
`superpowers:writing-plans`.
If MCP Memory: save entities Project/Task/Decision/Constraint with relations Project→Task→Decision, Project→Constraint.

---

## STEP 3 — GIT SETUP (skip entirely if `--git off`)

1. `git status --porcelain` — non-empty → ⛔ stop, list files, ask to commit/stash (`--include-untracked` for untracked files).
2. `git show-ref --verify --quiet refs/heads/grechman/<slug>` — exists → warn, offer rename/delete/resume options.
3. `git checkout -b grechman/<slug>`; if `--complexity hard`: `superpowers:using-git-worktrees`.
4. Rollback SHA always from: `git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}'` — never from CLAUDE.md.
5. Commit rules: only after verification passes · never broken code · format `grechman(step N): <desc>` · push after each if `--github on` (non-blocking) · never push main/master.
6. **Batch commit 1:** `git add CLAUDE.md && git commit -m "grechman(init): setup session"`

---

## STEP 4 — KNOWLEDGE

Per library: < 90 days old → use; version-pinned → always use; missing/stale → `resolve-library-id` → `query-docs` → append to knowledge.md (library, date queried, version, key APIs, gotchas). > 200 lines → archive unused to `knowledge-archive.md`.

Build KNOWLEDGE BLOCK: only entries for this task's libraries. Inject into every agent prompt — agents must never re-read knowledge.md.
If `ontology.yaml` loaded (Step 0d): scope entities — if depwire: `get_file_context` for task files → include only referenced entities; else: all entities if <10, otherwise only entities whose names appear in task description/files. Append to KNOWLEDGE BLOCK as `# ONTOLOGY (scoped)`: full `_manual` block + scoped `_generated` entities. If depwire: also call `get_architecture_summary` → append as `# ARCHITECTURE GRAPH`.
Update CLAUDE.md: `Libraries: [list]`
**Batch commit 2 (skip if `--git off`):** `git add CLAUDE.md knowledge.md && git commit -m "grechman(knowledge): load library context"`

---

## STEP 5 — DISPATCH

### 5a. Choose mode — write grechman-dispatch.md

Analyze plan dependencies. Write grechman-dispatch.md: mode, phases/steps (steps / files_create / files_modify / depends_on).

**SEQUENTIAL_SUBAGENTS** (`superpowers:subagent-driven-development`): 3+ ordered stages with distinct handoff artifacts (schema→models→API→tests) · OR total steps > 8 · OR crossing work domains (DB / API / UI / tests).
**RALPH_LOOP** (default): tightly coupled steps · ≤ 8 steps · single work domain · no natural phase boundary.

Show and wait for Y before any code runs:
`⚙️ Mode: [X] · Phases/Steps: [list] · Budget: [N] — Proceed? Y / N / adjust`

### 5b. Preamble (SEQUENTIAL_SUBAGENTS only)
If plan has setup steps (scaffolding, schema, env) that all phases depend on: run them as Phase 0 using the agent contract below. Record `PREAMBLE_DISPATCHES_USED`.
`git add grechman-dispatch.md && git commit -m "grechman(dispatch): manifest"` → write `Dispatch SHA: $(git rev-parse HEAD)` to CLAUDE.md.

### 5c. Agent prompts + contract (SEQUENTIAL_SUBAGENTS)

Each phase prompt must include: phase ID + DISPATCH_SHA + branch · exact steps for this phase · file lists (CREATE / MODIFY) · "use KNOWLEDGE BLOCK below, do NOT read knowledge.md" · KNOWLEDGE BLOCK (scoped to libraries in this phase; add `# [LibraryX excluded]` for withheld entries).

**Agent contract (applies to all dispatched agents — no other constraints):**
- FORBIDDEN writes: CLAUDE.md, knowledge.md, grechman-dispatch.md, other phases' result files
- Commits: never broken code; format `grechman(step N)`; never push main/master
- Packages/schema: new package needed → BLOCKED + no install; no schema changes unless in assigned steps
- Bugs outside scope: log to DISCOVERED_ISSUES, do not fix; run only modified-file tests
- Skills: TDD before new code · `superpowers:systematic-debugging` on failure · `superpowers:verification-before-completion` every step
- If `code-simplifier` installed: run post-verify, pre-commit
- Ontology: if `ontology.yaml` in scope, after each commit append new reusable conventions to `_manual.conventions`; on BLOCKED append to `_manual.rejected_approaches`
- Stuck: max 2 approaches per step → write FAILURE result + `AGENT_BLOCKED: <reason>`

Result file `grechman-result-[ID].md`:
- SUCCESS: PHASE / STATUS / COMMIT_SHA / FILES_CREATED / FILES_MODIFIED / TESTS_PASSING / VERIFICATION_OUTPUT (3 lines) / NEW_KNOWLEDGE / DISCOVERED_ISSUES
- FAILURE: PHASE / STATUS / FAILURE_REASON / LAST_CLEAN_COMMIT / PARTIAL_SAFE_TO_MERGE / FILES_BEFORE_FAILURE

### 5d. Sequential dispatch + consolidation

Phase by phase. Between phases, orchestrator writes `grechman-handoff.md`: phase from/to · HEAD_COMMIT · completed steps (1 line each) · key decisions (with file pointers) · files changed · remaining steps (exact text) · notes for next agent · known issues. Timeout: 30 min/phase → FAILURE → Step F.

On all SUCCESS: full integration verification.
Failure — one failed + independent: merge successes, retry failed phase as RALPH_LOOP. One failed + dependent OR multiple failed: `git reset --hard $DISPATCH_SHA` → write fallback → restart all as RALPH_LOOP.
Partial (PARTIAL_SAFE_TO_MERGE: true): merge + prepend remaining steps to next phase.

After all phases: write consolidated CLAUDE.md entry · merge NEW_KNOWLEDGE into knowledge.md · collect all DISCOVERED_ISSUES lines into `grechman-discovered-issues.md` (prefixed with source phase ID); skip if all empty.

Epilogue budget = `--budget − PREAMBLE_DISPATCHES_USED − phases_used`. If ≤ 2: warn + ask. After epilogue: delete grechman-dispatch.md · all grechman-result-*.md · grechman-handoff.md.

### 5e. Agent iteration loop (RALPH_LOOP)

Each step = one `Agent(general-purpose)` dispatch. Orchestrator loops until all steps done or budget exhausted.

**Build step prompt for each dispatch:**
- Exact step to implement (from plan)
- Branch + Last Stable SHA: `git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}'`
- Files created/modified so far this session
- KNOWLEDGE BLOCK (full)
- Per-step agent rules and return protocol below

**Per-step agent rules:**
- Use KNOWLEDGE BLOCK only — do NOT read knowledge.md
- TDD before any new feature code
- New package needed → return `GRECHMAN BLOCKED: new package required` (write reason to grechman-fallback.md first)
- Implement (minimal changes) · run tests for changed files only
- `superpowers:verification-before-completion` — mandatory before commit
- If `code-simplifier` installed: run post-verify, pre-commit
- If fail: `superpowers:systematic-debugging` + 1 fix + re-verify → if still fail: ROLLBACK
- Commit `grechman(step N): <desc>` → return `STEP COMPLETE: <sha>`
- If UI + Playwright MCP: navigate/interact/screenshot — infra failure: log+skip; UI bug: verification failure → back to debugging
- If `ontology.yaml` loaded: if this commit established a non-obvious reusable convention → append to `_manual.conventions`: `- "<one sentence>" # step N, YYYY-MM-DD` (conservative — only genuinely cross-session patterns)
- Max 2 approaches per step; if stuck: write grechman-fallback.md + append to `_manual.rejected_approaches` if ontology loaded → return `GRECHMAN BLOCKED: design blocked on step N`

**Return protocol:**
- `STEP COMPLETE: <sha>` — orchestrator records SHA to CLAUDE.md as Last Stable SHA, dispatches next step
- `GRECHMAN BLOCKED: <reason>` — orchestrator writes fallback → Step F
- `GRECHMAN COMPLETE` — all steps committed, final verification passed → Step 6

**Rollback (agent executes if verification fails after both approaches):**
`STABLE=$(git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}')`
Empty → return `GRECHMAN BLOCKED: no stable commit`
Else → `git reset --hard $STABLE` → return `GRECHMAN BLOCKED: <reason>`

**Budget exhausted:** if steps remain after `--budget` dispatches → orchestrator writes fallback → Step F.

---

## STEP F — FALLBACK

Triggers: budget exhausted · BLOCKED · ROLLBACK · merge conflict.
Write/update `grechman-fallback.md`: task · complexity/budget used/limit · git params · branch · stop reason · last stable SHA (`git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}'`) · completed steps with SHAs · remaining work in enough detail to resume · modified files.
Update CLAUDE.md `Status: PAUSED`. Output: `Grechman paused. Last stable: <sha>. Resume: /grechman --resume grechman-fallback.md --complexity <X> --git <X>`

---

## STEP 6 — FINAL REVIEW

**6a. Code review:** `superpowers:requesting-code-review` · diff branch vs main (`--git on`) or full working dir (`--git off`) · verify matches plan · flag scope creep if modified files > expected × 2.

**6b. Security** (skip irrelevant — check from `git diff --name-only`): frontend → XSS/exposed paths · DB/ORM → SQL injection/parameterized queries · auth → authz logic · API → input validation/rate limiting · all → hardcoded secrets/sensitive logs. HARD BLOCK: one targeted fix (dispatch as RALPH_LOOP single step) → re-check → if still failing: STOP + report to user (no second remediation, never "document and continue").

**6c. README (only if `--github on`):** update only sections affected by this task. `humanizer` if installed; else: no em dashes, no AI jargon, active voice, concrete examples.

**6d. Final commit:** update CLAUDE.md (Completed Tasks: branch/steps/status/commits/files/design doc; Architecture Decisions; Known Constraints). If MCP Memory: update Project entity (completed task, modified files, new decisions). If `ontology.yaml` loaded: review `_manual` block with user — any new conventions, decisions, or rejected approaches worth preserving from this session? If `adr-tools` installed: `adr new "<decision title>"` for each significant architectural decision; add ADR number to `_manual.decisions`. Commit ontology.yaml if updated.
```bash
# skip if --git off
git add CLAUDE.md
# if ontology updated: git add ontology.yaml
# if --github on: git add README.md
git commit -m "grechman(done): session complete"  # push if --github on
```

**6e. Delivery:** `superpowers:finishing-a-development-branch` → merge / PR / keep / discard. Output PR URL if `--github on` and PR created. If `grechman-discovered-issues.md` exists and non-empty: output contents (each line prefixed `⚠️ DISCOVERED:`), ask "Run a follow-up `/grechman` for any? Y/N" — delete file regardless of answer.

---

## RULES

**Never:** skip Step 1 gate · commit broken code · use CLAUDE.md as rollback SHA source · let agents re-read knowledge.md · skip 6b security review · push to main/master · dispatch agents with overlapping file sets.
**Always:** dirty tree + branch collision checks before branching · 3 batch commits (init / knowledge / done) · integration verify after all sequential phases.

**Resume (`--resume`):**
1. Step 0a only
2. Read grechman-fallback.md + CLAUDE.md
3. Per library in knowledge.md: check `date queried` individually — < 90 days → use; stale/missing → re-query; version-pinned → always use.
4. MCP Memory query: `"<project> architecture decisions constraints"`
5. Build KNOWLEDGE BLOCK (include memory context)
6. Validate remaining steps vs remaining budget (Step 1 rule)
7. Check `git log --fixed-strings --grep="grechman(knowledge)"` — if no commit: run Step 4 batch commit first
8. Skip to Step 5

---

## GLOSSARY

- **step** — a numbered unit of the plan; one commit per step on success
- **phase** — a group of steps assigned to one Agent dispatch (SEQUENTIAL_SUBAGENTS)
- **budget** — total Agent dispatches allowed (`--budget`); 1 per step in RALPH_LOOP, 1 per phase in SEQUENTIAL_SUBAGENTS
- **approach** — a distinct implementation strategy for a step; max 2 before BLOCKED
- **agent** — a Claude instance dispatched via the Agent tool with fresh context
- **orchestrator** — the main session that dispatches agents, reads results, manages state
- **KNOWLEDGE BLOCK** — compiled context (libraries + ontology + architecture) injected into every agent prompt
