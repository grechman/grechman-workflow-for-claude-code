# Grechman Workflow Benchmark

Rate each category **1–10** and compute the **weighted final score**.

| Score | Meaning                     |
| ----- | --------------------------- |
| 1–2   | Fails frequently            |
| 3–4   | Major issues                |
| 5–6   | Functional but inconsistent |
| 7–8   | Reliable                    |
| 9     | Highly optimized            |
| 10    | Exceptional                 |

---

# Definitions

```text
step = one planned implementation unit completed
dispatch = one agent execution call
unsafe_event = broken commit, corrupted repo state, unsafe branch operation
baseline_steps = 8
```

---

# Benchmark Table

| Category             | Score (1–10) | Weight |
| -------------------- | ------------ | ------ |
| Cost Efficiency      | 8            | 0.12   |
| Usability            | 8            | 0.08   |
| Error Handling       | 9            | 0.12   |
| Autonomy             | 8            | 0.15   |
| Determinism          | 8            | 0.08   |
| Safety               | 9            | 0.12   |
| Traceability         | 9            | 0.08   |
| Scalability          | 8            | 0.10   |
| Extensibility        | 8            | 0.07   |
| Verification Quality | 9            | 0.08   |

Total weight = **1.00**

---

# Category Evaluation

## Cost Efficiency

Evaluate how efficiently the workflow uses resources.

Measure:

* dispatches used vs dispatch budget
* unnecessary repeated steps or loops
* redundant tool calls or documentation queries
* token/context efficiency if observable

Metrics:

```text
dispatch_efficiency = steps_completed / dispatches_used
budget_ratio = dispatches_used / dispatch_budget
```

Scoring guidance:

* 1–3: budget exceeded or many wasted steps
* 4–6: moderate inefficiency
* 7–8: efficient with minimal waste
* 9–10: highly optimized resource usage

---

## Usability

Evaluate how easy the workflow is to operate.

Check:

* clarity of workflow instructions
* clarity of outputs and progress messages
* ease of configuring arguments
* operator understanding without external docs
* cognitive load required to run workflow

Scoring guidance:

* 1–3: confusing workflow
* 4–6: usable but requires frequent clarification
* 7–8: clear and manageable
* 9–10: very intuitive and self-explanatory

---

## Error Handling

Evaluate how the workflow manages failures.

Check:

* detection of errors or failed steps
* safe stop conditions
* rollback capability
* ability to continue via fallback or resume

Metric:

```text
recovery_rate = successful_recoveries / total_failures
```

Scoring guidance:

* 1–3: failures corrupt workflow state
* 4–6: errors handled inconsistently
* 7–8: reliable recovery behavior
* 9–10: robust fault isolation and recovery

---

## Autonomy

Evaluate how independently the workflow executes.

Check:

* number of manual interventions required
* ability to self-debug
* ability to make implementation decisions
* ability to proceed through full plan without help

Metric:

```text
autonomy_ratio = automated_steps / total_steps
```

Scoring guidance:

* 1–3: heavy human supervision
* 4–6: partial automation
* 7–8: mostly autonomous
* 9–10: near fully autonomous execution

---

## Determinism

Evaluate reproducibility of workflow behavior.

Check:

* consistency of step ordering
* reproducibility across multiple runs
* stability of decision logic
* absence of random branching behavior

Metric:

```text
determinism = identical_runs / total_runs
```

If only one run exists, estimate based on decision stability.

Scoring guidance:

* 1–3: highly inconsistent behavior
* 4–6: moderate variability
* 7–8: mostly consistent
* 9–10: highly deterministic execution

---

## Safety

Evaluate protection of repository and environment.

Check:

* prevention of committing broken code
* correct branch management
* safe rollback procedures
* protection of main/master branches
* absence of repository corruption

Metric:

```text
safety_score = max(1, 10 - unsafe_events)
```

Unsafe events include broken commits or corrupted repository state.

---

## Traceability

Evaluate ability to reconstruct the workflow session.

Check:

* commit messages clearly describe steps
* session logs exist
* planning artifacts exist
* fallback files exist when needed
* design decisions are documented

Scoring guidance:

* 1–3: poor auditability
* 4–6: partial record of workflow
* 7–8: clear session reconstruction possible
* 9–10: fully auditable process

---

## Scalability

Evaluate performance with increasing complexity.

Check:

* ability to handle larger plans
* stability with many steps
* ability to coordinate multi-phase execution
* performance with multi-agent workflows

Metric:

```text
scale_factor = steps_supported / baseline_steps
```

Scoring guidance:

* 1–3: breaks with complex tasks
* 4–6: struggles with large plans
* 7–8: handles complexity well
* 9–10: scales smoothly with task size

---

## Extensibility

Evaluate how easily the workflow integrates new tools.

Check:

* modular architecture
* ability to add new skills or agents
* ease of integrating external tools
* minimal modifications required for extensions

Scoring guidance:

* 1–3: difficult to extend
* 4–6: extensions require major changes
* 7–8: modular and adaptable
* 9–10: designed for seamless integration

---

## Verification Quality

Evaluate how well the workflow ensures code correctness.

Check:

* test-first development when applicable
* verification before committing changes
* integration checks
* security checks where relevant

Metric:

```text
verification_rate = verified_steps / total_steps
```

Scoring guidance:

* 1–3: verification often missing
* 4–6: basic checks only
* 7–8: strong verification practice
* 9–10: highly reliable correctness guarantees

---

# Final Score

```text
final_score = Σ(score × weight)
```

## Calculation

| Category             | Score | Weight | Contribution |
| -------------------- | ----- | ------ | ------------ |
| Cost Efficiency      | 8     | 0.12   | 0.96         |
| Usability            | 8     | 0.08   | 0.64         |
| Error Handling       | 9     | 0.12   | 1.08         |
| Autonomy             | 8     | 0.15   | 1.20         |
| Determinism          | 8     | 0.08   | 0.64         |
| Safety               | 9     | 0.12   | 1.08         |
| Traceability         | 9     | 0.08   | 0.72         |
| Scalability          | 8     | 0.10   | 0.80         |
| Extensibility        | 8     | 0.07   | 0.56         |
| Verification Quality | 9     | 0.08   | 0.72         |
| **TOTAL**            |       | **1.00** | **8.40**   |

## Rationale

**Cost Efficiency — 8**
The orchestrator is explicitly a thin dispatcher: agents return only a single-line acknowledgment and write structured YAML to `.grechman/task-reports/`; the orchestrator reads only the fields it needs. This keeps orchestrator context flat regardless of task count. Knowledge caching (90-day freshness, version-pin bypass, >200-line archiving) prevents redundant doc fetches. Budget gate (`step_count × 1.5 ≤ budget`) prevents over-commitment before coding starts. Not 9 because actual dispatch efficiency is only verifiable at runtime.

**Usability — 8**
`/grechman <task>` with sensible defaults requires minimal configuration. The orchestrator state block is printed between every turn for full visibility. Fallback gives the exact resume command. One friction point: the hard requirement on `superpowers:*` adds setup overhead; a missing skill causes a hard stop with no graceful degradation path.

**Error Handling — 9**
Two-strike rule is explicit: max 2 approaches per step, then rollback to last stable SHA via `git reset --hard` using `git log --grep` to find the SHA automatically. SEQUENTIAL mode adds two additional recovery paths: failed independent phases are retried as RALPH; failed dependent phases reset to `$DISPATCH_SHA`. `partial_safe_to_merge` flag allows salvaging partial work. Fallback agent commits state before stopping so resume is always clean. Missing packages block immediately rather than partially installing.

**Autonomy — 8**
The full workflow — plan, branch, fetch docs, implement, TDD, verify, review, commit, wrap up — executes without human input. Two intentional interactive checkpoints: (1) mode/step confirmation before coding, (2) ontology review in finish step. Both are deliberate design choices, not gaps. `--pre-specified on` and `--github on` reduce the interactive surface further.

**Determinism — 8**
Step ordering is fixed (Steps 0–F), branch naming is `grechman/<slug>`, commit format is `grechman(step N): <desc>`, and three batch commits (`init`, `knowledge`, `done`) always occur. Mode selection in Step 4 uses explicit rules (step count, domain count, phase boundaries). YAML report schema is enforced. Minor variability possible in LLM-generated plan content across runs.

**Safety — 9**
`main`/`master` is forbidden in the agent contract. Clean tree check and branch collision check run before branching. `superpowers:verification-before-completion` is mandatory before every commit. Security review is file-type scoped and blocks on unfixed issues — never "document and continue". Rollback to stable SHA is scripted, not ad-hoc. No observable unsafe events in the design.

**Traceability — 9**
Per-task YAML reports, `session_summary.yaml`, CLAUDE.md session log, structured commit messages, `docs/plans/` design docs, ADR support, and fallback file with last stable SHA together make any session fully reconstructable. Sessions are archived to `.grechman/sessions/YYYY-MM-DD/`. Discovered issues are collected into `grechman-discovered-issues.md` and surfaced to the user at finish. Score is 9 rather than 10 because log fidelity depends on agents consistently following the report schema.

**Scalability — 8**
SEQUENTIAL_SUBAGENTS mode handles multi-domain, multi-phase work with explicit handoff files between phases. Worktrees are used for hard complexity. Orchestrator context stays flat regardless of phase count because agents return minimal YAML. `scale_factor` = 25 / 8 = 3.1×. Does not address parallel orchestrators or multi-repo tasks, so 9–10 is not warranted.

**Extensibility — 8**
All non-core capabilities are optional: graceful degradation if depwire, Playwright, Sequential Thinking MCP, Memory MCP, qodo, humanizer, or adr-tools are absent. The pattern is clean — new capability = new MCP or skill added to the check table in Step 0. Ontology and depwire integrations demonstrate the model working end-to-end. Minor friction: no formal extension manifest; adding a capability requires editing prompt files directly.

**Verification Quality — 9**
TDD is mandated in the agent contract: write failing test → implement → verify pass via `superpowers:test-driven-development`. `superpowers:verification-before-completion` is mandatory before every commit. Security review checks XSS, SQL injection, parameterized queries, auth logic, rate limiting, and hardcoded secrets — scoped to actual file types changed. Depwire pre-edit call catches dependency breakage before code is written. Discovered issues across all tasks are aggregated and shown to the user. Not 10 because test suite scope is intentionally limited (changed files only, not full suite), so integration regressions outside the changed set are not caught.
