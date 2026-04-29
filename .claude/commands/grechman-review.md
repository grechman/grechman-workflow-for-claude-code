---
allowed-tools: Bash(git:*), Bash(jj:*), Bash(ls:*), Bash(mkdir:*), Bash(test:*), Bash(stat:*), Bash(date:*), Bash(rg:*), Bash(find:*), Bash(readlink:*), Bash(cat:*), Bash(awk:*), Bash(sort:*), Bash(head:*), Bash(tail:*), Bash(echo:*), Bash(realpath:*), Read, Write, Edit, Agent, AskUserQuestion, TaskCreate, TaskUpdate, TaskList, TaskGet
description: Multi-agent review of a commit range or branch. Default is research-only (3 parallel agents write reports for you to read). Pass --fixall to apply fixes from the most recent research run.
disable-model-invocation: false
---

# /grechman-review — Multi-Agent Branch / Range Review

Two-step workflow:

1. `/grechman-review <scope description or range>` — **research only**. Three
   parallel agents (security, correctness, architecture) write YAML reports.
   You read them.
2. `/grechman-review --fixall` — **apply fixes** from the most recent research
   run. One confirmation up front, then editor + resolver loop. Resolver
   validates each commit and reverts breakers.

You are the orchestrator. Thin dispatcher. The 5 specialist subagents do the work.

---

## Arguments

$ARGUMENTS

### Flags

| Flag | Default | Meaning |
|---|---|---|
| `--fixall` | off | Skip research. Apply fixes from the most recent review. |
| `--since <ref>` | — | Review everything from `<ref>` to HEAD. `<ref>` is any git-resolvable sha/branch/tag. |
| `--range <A..B>` | — | Explicit git range (two-point). |
| `--last <N>` | — | Review the last N commits (`HEAD~N..HEAD`). |
| `--base <branch>` | auto | Diff HEAD against this branch. Used when nothing else given. |
| `--fail-on <sev>` | off | Exit nonzero if residual ≥ severity (critical\|high\|medium). |
| `--max-fixes <N>` | 50 in --fixall, no cap otherwise | Cap on fixes applied. |
| `--timeout-research <N>` | — | Seconds per research agent. Omit to wait indefinitely. |

**Positional argument — a single git ref (sha, tag, or branch):**
`/grechman-review b4b4b4` is equivalent to `/grechman-review --since b4b4b4`.
This is the common case: "review everything I've done since my last review".

**Free-form text (not a git ref)** is treated as a scope description and
natural-language-resolved into a range (see R.3 below).

---

## Mode selection

Parse `$ARGUMENTS`:

1. If the arg list contains `--fixall` → **FIX MODE** (jump to Stage F).
2. Otherwise → **RESEARCH MODE** (Stages 0–2, then 5).

No mode does both. `--fixall` + free-form scope is an error
(`REVIEW BLOCKED: --fixall applies the latest research; do not pass a new scope`).

---

## State block

Maintain this EXACTLY between every turn:

```
GRECHMAN-REVIEW: mode=<research|fix> | stage=<preflight|resolve|research|plan|apply|done> | scope=<range|branch-vs-base> | head=<sha> | vcs=<git|jj> | files=<N> | research=<S:pending,C:pending,A:pending> | fixes=<applied>/<planned> | reverted=<N> | escalated=<N>
```

---

# RESEARCH MODE

## Stage 0 — Preflight

1. **Detect VCS**
   ```
   command -v jj >/dev/null 2>&1 && echo "jj" || echo "git"
   ```

2. **Require clean working tree** (same rule as before; dirty → BLOCKED).

3. **Record current HEAD sha** as `head_sha`.

## Stage R — Resolve scope

Scope priority (first match wins):

### R.1 Explicit `--range A..B`
Use verbatim. Validate with `git rev-parse A..B` — if invalid, BLOCKED.
Store `scope_type=range`, `base_ref = A`, `head_ref = B`.

### R.2 Explicit `--since <ref>` OR positional single ref
If `--since X` is passed, OR if $ARGUMENTS contains exactly one token that
`git rev-parse --verify` resolves (a sha, tag, or branch), use it as the
startpoint and HEAD as the endpoint.

- Resolve `<ref>` with `git rev-parse --verify <ref>` → `since_sha`
- Range = `<since_sha>..HEAD`. This **excludes** `since_sha` itself (it's
  the last commit you already reviewed) and includes everything after it.
- Store `scope_type=since`, `base_ref=<since_sha>`, `head_ref=HEAD`.

### R.3 Explicit `--last N`
Range = `HEAD~N..HEAD`. `base_ref=HEAD~N`, `head_ref=HEAD`.

### R.4 Explicit `--base <branch>`
Validate branch exists. `base_ref=<branch>`, `head_ref=HEAD`.
Store `scope_type=branch-vs-base`.

### R.5 Free-form text (not a git ref) — natural-language resolve

Identify the user's free-form string (everything in $ARGUMENTS that isn't a
known flag). Example: `"changes made on m06-m12"`.

Run the following resolution algorithm (use Bash, parse output inline):

**Step R.5.a — Extract candidate identifiers.**
Tokenize the free-form text. Keep tokens that look like git refs:
- `\b[mv]\d+\b` (milestone / version style: m06, v12)
- `\b[a-f0-9]{7,40}\b` (sha-like)
- `\b[A-Z]+-\d+\b` (ticket-style: FOO-123)
- Tokens containing `/` that look like branch paths

Also split tokens on `-` if the token itself looks range-like
(`m06-m12` → `m06`, `m12`).

**Step R.5.b — Resolve each candidate to a commit.**
For each token, try in this order:
1. `git rev-parse --verify "<tok>" 2>/dev/null` — is it a ref / sha / tag?
2. `git tag -l "<tok>"` — exact tag match?
3. `git log --all --oneline --grep="^\s*<tok>\b" -1` — subject starts with the token?
4. `git log --all --oneline --grep="\b<tok>\b" -1` — subject contains the token?

Record the first match per token with its commit sha and subject.

**Step R.5.c — Pick the range.**
- If ≥ 2 tokens resolved:
  - Order them by commit date (`git log --format='%H %ct' <sha1> <sha2> ...`)
  - Let `start_sha` = earliest, `end_sha` = latest
  - Range = `<start_sha>^..<end_sha>` (includes `start_sha` itself).
- If exactly 1 token resolved via methods 3 or 4 (grep match):
  - Treat it as the startpoint. Range = `<that_sha>..HEAD`.
- If 0 tokens resolved: go to R.6 (don't silently fall back — ask user).

**Step R.5.d — Confirm with the user.**
Show:
```
Interpreted "<free-form text>" as:
  range: <abbrev_start>..<abbrev_end>
  commits: <N> commits
  files changed: <M> files
  example subjects:
    - <short sha> <subject of earliest commit>
    - <short sha> <subject of latest commit>
```

Ask via `AskUserQuestion`: proceed / override with explicit range / cancel.

### R.6 NL resolution failed — ask, don't guess

If free-form text was passed but no tokens resolved via git, do NOT silently
default. `AskUserQuestion` with options:
- "Use `--last 30`" (review last 30 commits)
- "Specify a since-ref" → follow-up prompt asking for a sha/tag
- "Review current branch vs main" (the old default)
- "Cancel" → BLOCKED

### R.7 Fallback — branch vs main

Only if $ARGUMENTS was EMPTY (no free-form, no flags):

- Determine base branch: try `main`, `master`, `develop`. First that exists wins.
- If current branch == base: BLOCKED (`on base branch with no scope — pass a sha, --since, --range, or --last N`)
- Scope = `<base>..HEAD`. Store `scope_type=branch-vs-base`.

### After resolution

Compute `changed_files`:
- Range: `git diff --name-only <start_sha>^ <end_sha>` (or the resolved range)
- Branch: `git diff --name-only <base>...HEAD`

If 0 files → `REVIEW BLOCKED: no file changes in scope`.

Snapshot:
- `base_sha`
- `head_sha` (always current HEAD — research runs read-only so this doesn't matter much, but fix mode uses it)
- `changed_files`

## Stage 1 — Research dir + dispatch 3 agents

1. `TS=$(date -u +%Y%m%dT%H%M%SZ)`
2. `mkdir -p .grechman/review/$TS/reports .grechman/review/$TS/fixes .grechman/review/$TS/logs`
3. Write `<review_dir>/preflight.yaml`:
   ```yaml
   timestamp: <ISO>
   mode: research
   scope_type: range | branch-vs-base
   scope_description: <free-form text verbatim, or null>
   range: <start_sha>..<end_sha>
   base_branch: <branch or null>
   head_sha: <sha>
   vcs: <git|jj>
   changed_files: [list]
   flags: {fail_on: <sev|null>, timeout_research: <N|null>}
   ```
4. Update pointer: `echo "<absolute review_dir>" > .grechman/review/.latest`

5. **Dispatch three agents IN PARALLEL** — three `Agent` tool calls in one message,
   each with `run_in_background: true`:

```
subagent_type: grechman-<security|correctness|architecture>

Review the diff between <base_ref> and <head_ref> for
<security|correctness|architecture> issues.

Inputs:
- base_ref: <start_sha or branch name>
- head_ref: <end_sha or "HEAD">
- changed_files: <N files — see .grechman/review/<ts>/preflight.yaml for full list>
- review_dir: <absolute path>
- vcs: <git|jj>
- scope_description: <free-form text or null — this is a HINT about what the
  user cares about, not a restriction on what you review>

Follow your system prompt. Read-only.
Write your report to <review_dir>/reports/<agent>.yaml.
Return one line:
- REPORT WRITTEN: <path> | findings=<N>
- REPORT BLOCKED: <reason>
```

Update state: `research=<S:running,C:running,A:running>`.

## Stage 2 — Poll until complete

Every 10 s, check `test -f <review_dir>/reports/<agent>.yaml` for each.

If `--timeout-research N` was passed, cap the wait at N seconds per agent.
Otherwise wait indefinitely — an agent will finish eventually, and a diff of
any realistic size will not take "forever".

Update state as each completes. If `--timeout-research` was set AND all 3 time
out → `REVIEW BLOCKED`.

## Stage 5 — Research summary

Read the three report files. Write `<review_dir>/summary.md` with:
- Scope (range, files)
- Findings table by agent × severity
- Top findings: list first 5 from each report with file:line + description
- Considered-and-rejected paragraphs from each report

Print a condensed version to the user. End with:

```
REVIEW COMPLETE (research only).

Reports:   .grechman/review/<ts>/reports/{security,correctness,architecture}.yaml
Summary:   .grechman/review/<ts>/summary.md

To apply fixes:  /grechman-review --fixall
To review again with different scope:  /grechman-review --range A..B
```

If `--fail-on <sev>` is set: exit nonzero when residual findings exist at that severity.

State: `stage=done`.

---

# FIX MODE (--fixall)

## Stage F.0 — Load latest review

1. `test -f .grechman/review/.latest` or BLOCKED: "no previous review — run /grechman-review first"
2. `review_dir = cat .grechman/review/.latest`
3. `test -d "$review_dir"` or BLOCKED: "latest review dir missing — run /grechman-review again"
4. Verify reports exist: security.yaml, correctness.yaml, architecture.yaml. Any missing is OK — we'll just plan from what's there.
5. Detect VCS (same as research mode).
6. Require clean working tree. Dirty → BLOCKED.
7. Snapshot current HEAD as `last_stable_sha`.
8. Read `preflight.yaml` for original scope context.

## Stage F.1 — Plan

Dispatch `@grechman-editor` in PLAN mode (foreground):

```
subagent_type: grechman-editor
mode: PLAN

Inputs:
- review_dir: <abs>
- reports: [security.yaml, correctness.yaml, architecture.yaml] (any that exist)
- last_stable_sha: <sha>
- max_fixes: <flag or 50 default>
- vcs: <git|jj>

Follow your system prompt. Produce <review_dir>/fix-plan.yaml.
Return: PLAN WRITTEN: <path> | total=<N> | conflicts=<M> | escalated=<K>
```

Read the fix plan.

## Stage F.2 — Confirmation gate

Print a concrete summary:

```
FIX PLAN (from review <ts>):
  total fixes:  <N>
  by severity:  critical=X, high=Y, medium=Z, low=W
  by agent:     security=A, correctness=B, architecture=C
  conflicts:    <M>  (resolver arbitrates)
  escalated:    <K>  (human — left as residual)
  first 5 fixes:
    - F1 (S2 high sql-injection)    <file>:<line>    — parameterize query
    - F2 (C4 high null-deref)       <file>:<line>    — guard missing key
    - ...
```

Ask via `AskUserQuestion`:
- "Apply all N fixes now" (default)
- "Apply with a lower cap" → prompt for new max, then continue
- "Cancel — don't apply anything"

If cancel: `REVIEW ABORTED: no fixes applied` and exit.

## Stage F.3 — Apply loop

Same as Stage 4 in the original design. For each fix in order:

1. `@grechman-editor` (ACT mode) applies one fix = one commit.
   - If `FIX SKIPPED`: log, skip, continue.
   - If `FIX PENDING ARBITRATION`: dispatch resolver in ARBITRATE mode,
     then re-dispatch editor with chosen candidate.
2. `@grechman-resolver` validates the commit on 6 axes. Executes revert itself if needed.
3. Act on verdict: ACCEPT / REVERT / ESCALATE. Update state + decision log.

Stop conditions:
- All fixes processed.
- 3 consecutive REVERTs → BLOCKED with pointer to logs.
- `--max-fixes` cap reached.

## Stage F.4 — Summary

Update `<review_dir>/summary.md` with fix outcome (applied / reverted / escalated).
Print condensed to user. Exit based on `--fail-on`.

State: `stage=done`.

---

## Guardrails

- Refuses on dirty working tree (both modes).
- Never force-push, never delete branches.
- Fix mode NEVER re-runs research (that's the whole point of the split).
- Research mode NEVER edits code.
- The `.grechman/review/.latest` pointer is the single source of truth for "what --fixall will act on". If a user wants to apply a specific older review, they have to point the file at that dir manually (documented as an escape hatch).
- On any return format mismatch from a subagent: treat as failure and escalate.
- If a resolver REVERT fails to actually revert (git rev-parse disagrees), stop everything with explicit SHAs and a manual-recovery note.

## Compatibility

Independent from `/grechman`. Can be run mid-session or between sessions.
Review dirs accumulate in `.grechman/review/<ts>/` — safe to gitignore the whole
`.grechman/` directory if you haven't already.
