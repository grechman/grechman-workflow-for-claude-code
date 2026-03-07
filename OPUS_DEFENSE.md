# Opus Cost-Benefit Review — Defense

## Overview

The current `grechman.md` (commit `e6e04f6`) is the state after the 6 efficiency changes were applied and then partially rolled back via a correctness restoration pass. My job was to determine, for each of the 6 changes, whether the current file is correct, whether any remaining regression exists, and whether to keep, modify, or revert each change.

**Finding:** Four of the six regressions were already fixed in `e6e04f6`. One change (Change 2, `tools_used`) was consciously not restored and I agree with that decision. The batch depwire approach (Changes 1+5) is architecturally sound as written. No changes to `grechman.md` were required — the file is correct.

---

## Change 1+5 — Batch depwire + file exemptions

**Verdict: KEPT (as written in current file)**

**Reasoning:**

The batch approach calls depwire once for ALL files in the agent's assigned set before any edits begin. Because the plan specifies which files will be touched before dispatch, the agent has the full working-set dependency graph upfront. The mid-task discovery regression (file A's edit revealing dependency on file B) only matters if B was not already in the planned file set — which means it's a planning failure, not a depwire call frequency problem. For well-formed plans, the batch approach captures all necessary context.

The `*.yaml` over-broad exemption regression was the more serious issue. It was fixed: the current file explicitly exempts only "root-level tool/CI config YAMLs" and carves out "NOT schema/API/spec YAMLs (openapi.yaml, schema.yaml, etc.) which may have importers." This is precise and correct.

Token math: 6-file task saves ~6 depwire calls × ~300 tokens each = ~1,800 tokens saved. Recovery cost if a dependency is missed: potentially 5-10x a phase (~3,000-10,000 tokens). But the batch call covers the full planned file set, so the probability of a missed dependency is low (only unplanned file additions create the gap). E[tokens] for batch approach: positive.

**What changed (if modified):** Nothing. Already correct in current file.

---

## Change 2 — `tools_used` removed from task report YAML

**Verdict: KEPT REMOVED**

**Reasoning:**

The `tools_used` field is a post-hoc record, not an enforcement mechanism. An agent that skips `superpowers:verification-before-completion` and ships broken code is unlikely to honestly self-report the omission in `tools_used`. The actual enforcement is in the prompt rules: "TDD before any new feature code," "verification-before-completion — mandatory before commit," "`superpowers:systematic-debugging` on failure." These rules govern behavior at dispatch time. The YAML audit field governs only what gets written to a file after the fact.

The claim "git log already captures it" is wrong — git log shows commits, not tool invocations. But the counter-claim that restoring the field would prevent mistakes is also wrong: the field has zero causal power over agent behavior. Its only value is forensic, and a bad-faith agent will omit it from both the field and actual execution.

Token math: 100-200 tokens saved per task step across both RALPH_LOOP and SEQUENTIAL_SUBAGENTS schemas. At 8 steps per session, that's 800-1,600 tokens saved. Recovery cost if an agent skips verification: high, but the probability that `tools_used` in the YAML would have prevented the skip is near zero. E[tokens] for keeping removed: positive (small but real savings, negligible loss of protection).

**What changed (if modified):** Nothing. Not restored.

---

## Change 3 — Eliminate `grechman-result-[ID].md`, extend YAML

**Verdict: KEPT (regression already fixed in e6e04f6)**

**Reasoning:**

The two regressions in Change 3 were both corrected in the restoration commit:

1. `files_changed` (merged) was split back into `files_created` and `files_modified` in both the SEQUENTIAL_SUBAGENTS schema (lines 144-147) and the RALPH_LOOP schema (lines 216-219). The next-phase agent can now distinguish new files from modified files.

2. `verification_summary: one sentence` was replaced with `verification_output: | <2-3 lines: test runner output, pass/fail counts, any warnings — enough for orchestrator to diagnose silent failures>`. The orchestrator can now diagnose silent failures from actual test output rather than an opaque summary.

The result file elimination itself (~2 tool calls saved per phase) is correct to keep. The YAML now carries all necessary diagnostic information. The single-artifact approach reduces orchestrator complexity.

Token math: 2 tool calls saved per phase (one write, one read). At 3 phases: 6 tool calls × ~200 tokens = ~1,200 tokens saved. The regressions are fixed, so recovery cost exposure is near zero. E[tokens]: positive.

**What changed (if modified):** Nothing in current review. Both regressions already restored.

---

## Change 4 — Conditional handoff file

**Verdict: KEPT (regression already fixed in e6e04f6)**

**Reasoning:**

The regression — skipping `grechman-handoff.md` for "clean" phases and losing the current step list — was severe. If the plan was modified mid-session (partial phases, step prepending from line 175), the dispatch manifest becomes stale, and the next agent dispatched from the stale manifest would misunderstand its position in the plan.

The restoration fixed this by changing the condition to "Write `grechman-handoff.md` whenever remaining steps > 0 (i.e., between all non-final phases)." This is equivalent to "always write between phases" for any multi-phase work. The content is scaled: clean phases write minimal handoff (remaining steps + HEAD_COMMIT only); phases with new_knowledge, discovered_issues, or blockers write full handoff. This preserves the token savings for clean phases (no extra content) while guaranteeing the step list is always current.

Token math: Original "always write full handoff" cost ~300-500 tokens per phase for the write. Current approach writes minimal handoff for clean phases (~50-100 tokens) and full handoff when warranted. Savings: ~200-400 tokens per clean phase. Recovery cost of skipping entirely: entire next phase dispatched with wrong step context → potentially full phase wasted. E[tokens]: positive for current approach.

**What changed (if modified):** Nothing in current review. Regression already restored.

---

## Change 6 — Inline SHA in return signal, conditional YAML reads in RALPH_LOOP

**Verdict: KEPT (regression already fixed in e6e04f6)**

**Reasoning:**

The regression — orchestrator dispatching the next step on a `partial` without knowing what's partial — was fixed by the conditional read rule: "if status is `partial` or `blocked`, orchestrator MUST read the full YAML immediately before dispatching next step; otherwise full YAML is read in bulk during post-session aggregation only."

This is the correct safeguard. The orchestrator saves N YAML reads for normal `completed` steps (bulk read at session end) while forcing an immediate YAML read for the only cases where step details matter for the next dispatch decision.

The brittle string parsing concern is real but acceptable. The format `STEP COMPLETE: <sha> | <status>` is simple: split on ` | `, take last token. The status values are a closed set (`completed`, `partial`, `blocked`). No ambiguity.

Token math: 8-step RALPH_LOOP saves 7 YAML reads (reads only when partial/blocked, plus final aggregation). Each YAML read is ~2-3 tool calls × ~200 tokens = ~400-600 tokens. Savings: ~2,800-4,200 tokens per session. Recovery cost if orchestrator misses partial state: addressed by the conditional read requirement. E[tokens]: positive.

**What changed (if modified):** Nothing in current review. Regression already restored.

---

## Summary table

| Change | Verdict | File action |
|---|---|---|
| 1+5: Batch depwire + exemptions | KEPT | No change (already correct) |
| 2: Remove `tools_used` | KEPT REMOVED | No change (field stays absent) |
| 3: Single YAML artifact | KEPT (regressions fixed) | No change (already correct) |
| 4: Conditional handoff | KEPT (regression fixed) | No change (already correct) |
| 6: Inline SHA + conditional reads | KEPT (regression fixed) | No change (already correct) |

**No changes were made to `grechman.md`.** The file at `e6e04f6` is correct. The only open question is Change 2 (`tools_used`), where I recommend keeping it removed — documented above.
