# Grechman — Automated Dev Workflow (medium/hard tasks only)

You are running the Grechman workflow. Parse all $ARGUMENTS before doing anything.

## Arguments from user:
$ARGUMENTS

---

## STEP 0 — PREFLIGHT

### 0a. Skills
Hard-stop (output ⛔ GRECHMAN CANNOT START + list + install instructions) if missing: `ralph-loop` · `superpowers:*`

Conditional:
| Skill | When |
|---|---|
| `superpowers:brainstorming` | unless `--pre-specified on` |
| `superpowers:using-git-worktrees` | hard complexity OR PARALLEL mode |
| `frontend-design` | UI/CSS/components/design |
| `humanizer` | `--github on` |
| `pr-review-toolkit:review-pr` | reviewing existing PR |
| `Playwright MCP` | UI/frontend/browser |
| MCP Memory / Sequential Thinking MCP / Desktop Commander MCP | if installed |
| `qodo-skills:get-qodo-rules` | if installed — run once in Step 0c, inject rules into all agent prompts |

Optional missing → list adaptations, Y/N prompt, proceed if Y.

### 0b. Arguments
| Param | Default | Values |
|---|---|---|
| `--complexity` | `medium` | `medium` / `hard` |
| `--git` | `on` | `on` / `off` |
| `--github` | `off` | `on` / `off` (needs `--git on` + remote) |
| `--pre-specified` | `off` | `on` — skip brainstorming |
| `--max-iterations-ralph` | medium=15 / hard=25 | override |
| `--resume` | off | path to grechman-fallback.md → skip to resume path in RULES |

If `--resume`: Step 0a only → resume path. If `--github on` + no remote: warn + set off.
Output: `Task: [X]. Complexity: [X]. Git: [X]. GitHub: [X]. Iterations: [N].`

### 0c. CLAUDE.md
If missing: create with sections Project (name/stack/entry points/dirs), Session Log, Completed Tasks, Architecture Decisions, Known Constraints. If exists: append session entry (task/complexity/params/branch=TBD/status=IN PROGRESS/skills/libraries=TBD).
If MCP Memory: ONE read query `"<project> architecture decisions constraints"` → add under "From Memory". (Read-only; Step 2 writes are fine.) Do NOT commit yet.
If `qodo-skills:get-qodo-rules`: invoke once → save rules summary; inject into KNOWLEDGE BLOCK in Step 4.

---

## STEP 1 — SCOPE ESTIMATE (runs before brainstorming)

Ask: "List implementation steps in one sentence each — rough only."
Rule: `steps × 1.5 ≤ max-iterations`. On fail: output ⚠️ with counts + options A (split now/later) / B (raise --max-iterations-ralph) / C (trim plan). Wait for choice.

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
3. `git checkout -b grechman/<slug>`; if hard: `superpowers:using-git-worktrees`.
4. After each verified commit: write `Last Stable SHA: <sha>` to CLAUDE.md (don't commit separately; included in next batch). Rollback SHA always from: `git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}'` — never from CLAUDE.md.
5. Commit rules: only after verification passes · never broken code · format `grechman(step N): <desc>` · push after each if `--github on` (non-blocking) · never push main/master.
6. **Batch commit 1:** `git add CLAUDE.md && git commit -m "grechman(init): setup session"`

---

## STEP 4 — CONTEXT7 / KNOWLEDGE

Per library: < 90 days old → use; version-pinned → always use; missing/stale → `resolve-library-id` → `query-docs` → append to knowledge.md (library, date queried, version, key APIs, gotchas). > 200 lines → archive unused to `knowledge-archive.md`.

Build KNOWLEDGE BLOCK: only entries for this task's libraries. Inject once into execution prompt — never re-read knowledge.md during execution.
Update CLAUDE.md: `Libraries: [list]`
**Batch commit 2 (skip if `--git off`):** `git add CLAUDE.md knowledge.md && git commit -m "grechman(knowledge): load library context"`

---

## STEP 5 — DISPATCH

### 5a. Choose mode — write grechman-dispatch.md
Analyze plan dependencies. Write grechman-dispatch.md: mode, preamble step list, epilogue step list, work packages (steps / files_create / files_modify / branch / depends_on).

**PARALLEL** (`superpowers:dispatching-parallel-agents`) — ALL must be true: 2+ groups with non-overlapping files · no inter-group deps · each group ≥ 3 steps · none touches CLAUDE.md/knowledge.md/package.json/lockfiles/schema/shared utils/.env/git config.
**SEQUENTIAL_SUBAGENTS** (`superpowers:subagent-driven-development`) when: 3+ ordered stages with distinct handoff artifacts (schema→models→API→tests) · OR 20+ tool calls accumulated in context · OR switching work modes.
**RALPH_LOOP** (default): tightly coupled steps · ≤ 8 steps · no natural decomposition · touches shared infra.

**Pre-dispatch file conflict check (PARALLEL only):** Before showing the dispatch prompt, extract every path in `files_create` and `files_modify` across ALL work packages. If any path appears in more than one WP: output `⛔ FILE CONFLICT IN DISPATCH` · list each conflicting path and which WPs claim it · force mode to SEQUENTIAL_SUBAGENTS or RALPH_LOOP · rewrite grechman-dispatch.md · do NOT proceed with PARALLEL. Only show dispatch prompt after this check passes.

Show and wait for Y before any code runs:
`⚙️ Mode: [X] · Preamble: [steps] · Packages: [WP-A: steps X–Y] · Epilogue: [steps] · Budget: ~N iters — Proceed? Y / N / adjust`

### 5b. Preamble (PARALLEL or SEQUENTIAL_SUBAGENTS only)
Run preamble steps in ralph-loop (`--max-iterations <max>`). Record `PREAMBLE_ITERS_USED`.
`git add grechman-dispatch.md && git commit -m "grechman(dispatch): manifest"` → `DISPATCH_SHA=$(git rev-parse HEAD)` → write `Dispatch SHA: $DISPATCH_SHA` to CLAUDE.md.
SEQUENTIAL_SUBAGENTS: exclude preamble steps from Phase 1's assignment.

### 5c. Create agent branches (PARALLEL only)
For each work package, in order:
1. Create branch: `git checkout -b grechman/<slug>-wp-<id> $DISPATCH_SHA && git checkout grechman/<slug>`
2. Create worktree: `git worktree add grechman-wp-<id>-worktree grechman/<slug>-wp-<id>`
3. Record worktree path in grechman-dispatch.md under the WP entry: `worktree: grechman-wp-<id>-worktree`

Each agent must run exclusively inside its assigned worktree directory. FORBIDDEN: any agent reading or writing paths outside its own worktree.
If DISPATCH_SHA out of scope: `grep 'Dispatch SHA:' CLAUDE.md | awk '{print $NF}'`

### 5d. Build agent prompts
Each prompt must include: agent ID + mode + DISPATCH_SHA + assigned branch · exact plan steps assigned · file lists (CREATE / MODIFY) · FORBIDDEN writes: CLAUDE.md, knowledge.md, grechman-dispatch.md, other agents' result files · "use KNOWLEDGE BLOCK below, do NOT read knowledge.md" · KNOWLEDGE BLOCK (scoped: include only entries whose library appears in the steps or files assigned to this WP; for each excluded entry add a comment line `# [LibraryX excluded — not referenced in this WP]` so the agent knows what was withheld).

Constraints: never commit broken code · format `grechman(step N)` · new package needed → write BLOCKED to result file, no install · no schema changes unless in steps · out-of-scope bugs → DISCOVERED_ISSUES, don't fix · run only modified-file tests.
Skills: TDD before any new code · systematic-debugging on failures · verification-before-completion every iteration.
Rollback: git stash → find last clean SHA → write FAILURE result → output `AGENT_BLOCKED: <reason>` (max 2 approaches per step).

Result file `grechman-result-[ID].md`:
- SUCCESS: WORK_PACKAGE / STATUS / COMMIT_SHA / FILES_CREATED / FILES_MODIFIED / TESTS_PASSING / VERIFICATION_OUTPUT (3 lines) / NEW_KNOWLEDGE / DISCOVERED_ISSUES
- FAILURE: WORK_PACKAGE / STATUS / FAILURE_REASON / LAST_CLEAN_COMMIT / PARTIAL_SAFE_TO_MERGE / FILES_CREATED_BEFORE_FAILURE

SEQUENTIAL_SUBAGENTS: **orchestrator** (not agent) writes `grechman-handoff.md` between phases using agent result file. Include: phase from/to, HEAD_COMMIT, completed steps (1 line each), key decisions (with file pointers), files changed, remaining steps (exact text), notes for next agent, known issues.

### 5e. Dispatch and wait
**PARALLEL:** `superpowers:dispatching-parallel-agents`. Wait for all result files. Timeout: 30 min or 50 polls → cascade all non-completed agents to FAILURE (synthesize result files) → proceed to failure routing.
**SEQUENTIAL_SUBAGENTS:** phase by phase; orchestrator writes handoff between phases. Timeout: 30 min/phase → FAILURE → Step F.
**RALPH_LOOP:** invoke `ralph-loop` with prompt from 5f.

### 5f. Ralph loop execution
`ralph-loop --completion-promise "GRECHMAN COMPLETE" --max-iterations <N from 0b>`

Pass: task + plan + KNOWLEDGE BLOCK + memory context (from Step 0) + these rules:

Skills: TDD before new features · systematic-debugging on any failure · verification-before-completion every iteration (no exceptions).

Per iteration:
1. Identify next step · consult KNOWLEDGE BLOCK
2. If new feature: TDD first
3. If new package needed: write to grechman-fallback.md + output `GRECHMAN BLOCKED: new package required` — do not install
4. Implement (minimal changes) · run tests for changed files
5. `superpowers:verification-before-completion` — mandatory
5a. If `code-simplifier` installed: run on files modified this iteration (post-verify only, pre-commit).
6. If fail: systematic-debugging + 1 fix + re-verify → if still fail: ROLLBACK (below)
7. Commit `grechman(step N): <desc>` · write Last Stable SHA to CLAUDE.md
8. If UI + Playwright MCP: navigate/interact/screenshot — infra failure: log+skip; UI bug: treat as verification failure (go back to 6)
9. Push if `--github on`
10. Every 10 iterations: evaluate `completed_steps / total_steps`. If `completed_steps < total_steps × 0.4` AND `iterations_remaining < iterations_used` → write fallback + output `GRECHMAN BLOCKED: budget rate insufficient (completed <X>/<Y> steps at iteration <N>)` + STOP. Do NOT continue burning budget on a trajectory that cannot complete.

Rollback: `STABLE=$(git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}')` · if empty → write fallback + `GRECHMAN BLOCKED: no stable commit` + STOP · else `git reset --hard $STABLE` → write fallback → `GRECHMAN BLOCKED: <reason>`. Max 2 approaches per step.

Stuck (design, not test): try exactly 1 alternative approach → if still blocked after that single attempt: write fallback + `GRECHMAN BLOCKED: design blocked on step N`. Ask about scope changes, not implementation. Never attempt a third approach on the same step.
Done → output `GRECHMAN COMPLETE` only when all steps committed · final verification passed · no hardcoded secrets · follows existing patterns · nothing beyond task scope.

### 5g. Consolidation (PARALLEL + SEQUENTIAL_SUBAGENTS)
All SUCCESS → merge each WP into session branch + verify. On conflict: `git merge --abort` → write to fallback → `GRECHMAN BLOCKED: merge conflict on <branch>`. After all merges: full integration verification.

Failure: one failed + independent → merge successes + retry failed as ralph-loop. One failed + dependent OR multiple failed → abandon all → `git reset --hard $DISPATCH_SHA` (read from CLAUDE.md if var gone) → write fallback → restart all in ralph-loop. Partial (PARTIAL_SAFE_TO_MERGE: true) → merge + add remaining steps to epilogue front.

After merges: write consolidated CLAUDE.md entry (WPs/SHAs/files) · merge NEW_KNOWLEDGE into knowledge.md · collect all DISCOVERED_ISSUES lines from every grechman-result-WP-*.md into `grechman-discovered-issues.md` (one issue per line, prefixed with source WP ID); skip if all result files have empty DISCOVERED_ISSUES.

Epilogue: PARALLEL budget = `max-iterations − PREAMBLE_ITERS_USED`; SEQUENTIAL_SUBAGENTS budget = `max-iterations − SUM(all phase iterations)`. Use `--max-iterations $EPILOGUE_BUDGET`. If ≤ 2: warn + ask before proceeding.

After epilogue: remove all worktrees (`git worktree remove grechman-wp-<id>-worktree --force` per WP) · delete grechman-dispatch.md · all grechman-result-WP-*.md · grechman-handoff.md.

---

## STEP F — FALLBACK

Triggers: max-iterations hit · BLOCKED · ROLLBACK · merge conflict.
Write/update `grechman-fallback.md`: task, complexity/iters used/limit, git params, branch, stop reason, last stable SHA (`git log --oneline --fixed-strings --grep="grechman(step" | head -1 | awk '{print $1}'`), completed steps with SHAs, remaining work in enough detail to resume, modified files.
Update CLAUDE.md `Status: PAUSED`. Output: `Grechman paused. Last stable: <sha>. Resume: /grechman --resume grechman-fallback.md --complexity <X> --git <X>`

---

## STEP 6 — FINAL REVIEW

**6a. Code review:** `superpowers:requesting-code-review` · diff branch vs main (`--git on`) or full working dir (`--git off`) · verify matches plan · flag scope creep if modified files > expected × 2.

**6b. Security** (skip irrelevant categories — check from `git diff --name-only`): frontend → XSS/exposed paths · DB/ORM → SQL injection/parameterized queries · auth → authz logic · API → input validation/rate limiting · all → hardcoded secrets/sensitive logs. HARD BLOCK: one targeted ralph-loop fix → re-check → if still failing: STOP + report to user (no second remediation, never "document and continue").

**6c. README (only if `--github on`):** update only sections affected by this task. `humanizer` if installed; else: no em dashes, no AI jargon, active voice, concrete examples.

**6d. Final commit:** update CLAUDE.md (Completed Tasks: branch/steps/status/commits/files/design doc; Architecture Decisions; Known Constraints). If MCP Memory: update Project entity (completed task, modified files, new decisions).
```bash
# skip if --git off
git add CLAUDE.md
# if --github on: git add README.md
git commit -m "grechman(done): session complete"  # push if --github on
```

**6e. Delivery:** `superpowers:finishing-a-development-branch` → merge / PR / keep / discard. Output PR URL if `--github on` and PR created. If `grechman-discovered-issues.md` exists and is non-empty: output its contents (each line prefixed `⚠️ DISCOVERED:`), then ask: "These issues were found out-of-scope during execution. Run a follow-up `/grechman` for any? Y/N" — delete the file after displaying regardless of answer.

---

## RULES

**Never:** skip Step 1 gate · commit broken code · use CLAUDE.md as rollback SHA source · re-read knowledge.md during execution · skip 6b security review · push to main/master · dispatch agents with overlapping file sets.
**Always:** dirty tree + branch collision checks before branching · create all agent branches before any dispatch · 3 batch commits (init / knowledge / done) · integration verify after all parallel merges.

**Resume (`--resume`):**
1. Step 0a only
2. Read grechman-fallback.md + CLAUDE.md
3. Per library entry in knowledge.md: check its `date queried` field individually. < 90 days → use as-is; stale or missing entry → re-query that library only; version-pinned → always use. Do not re-query entries that are still fresh.
4. MCP Memory query: `"<project> architecture decisions constraints"`
5. Build KNOWLEDGE BLOCK (include memory context from step 4)
6. Validate remaining steps vs remaining iterations (Step 1 rule)
7. Check `git log --fixed-strings --grep="grechman(knowledge)"` — if no commit: run Step 4 batch commit first
8. Skip to Step 5
