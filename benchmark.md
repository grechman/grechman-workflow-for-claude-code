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
| Cost Efficiency      | 7            | 0.12   |
| Usability            | 8            | 0.08   |
| Error Handling       | 8            | 0.12   |
| Autonomy             | 8            | 0.15   |
| Determinism          | 8            | 0.08   |
| Safety               | 9            | 0.12   |
| Traceability         | 9            | 0.08   |
| Scalability          | 8            | 0.10   |
| Extensibility        | 8            | 0.07   |
| Verification Quality | 8            | 0.08   |

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
| Cost Efficiency      | 7     | 0.12   | 0.84         |
| Usability            | 8     | 0.08   | 0.64         |
| Error Handling       | 8     | 0.12   | 0.96         |
| Autonomy             | 8     | 0.15   | 1.20         |
| Determinism          | 8     | 0.08   | 0.64         |
| Safety               | 9     | 0.12   | 1.08         |
| Traceability         | 9     | 0.08   | 0.72         |
| Scalability          | 8     | 0.10   | 0.80         |
| Extensibility        | 8     | 0.07   | 0.56         |
| Verification Quality | 8     | 0.08   | 0.64         |
| **TOTAL**            |       | **1.00** | **8.08**   |

## Rationale

**Cost Efficiency — 7**
Dispatch budget is explicit (15/25) and execution mode selection (RALPH_LOOP / SEQUENTIAL / PARALLEL) minimizes waste. The context overflow fix in `grechman-update.md` (structured YAML reports, ~500 token per task cap) directly addresses token inefficiency. Score is 7 rather than higher because actual dispatch efficiency is only verifiable at runtime — the budget controls are sound but not independently observable.

**Usability — 8**
`/grechman <task>` is straightforward with well-documented flags, defaults, and concrete usage examples. Fallback/resume lowers recovery friction significantly. Small deduction: two hard-required skills (`ralph-loop`, `superpowers:*`) add a one-time setup hurdle not handled automatically.

**Error Handling — 8**
Covers the main failure classes: iteration limit, merge conflicts, missing packages, unrecoverable rollbacks. Fallback file captures last stable SHA and remaining work; resume is a single command. Task-level `blockers` field gives structured per-task fault reporting. Not 9 because rollback itself (reverting to last stable SHA) is described but the mechanism isn't fully specified in the prompts.

**Autonomy — 8**
Plans, branches, fetches docs, implements, reviews, and commits without human input. One intentional pause: execution mode confirmation before running. This is a sensible checkpoint, not a weakness. `--pre-specified on` removes brainstorming if the operator already has a plan. Near-fully autonomous for most tasks.

**Determinism — 8**
Step ordering is fixed (Steps 0–7), branch naming is `grechman/<slug>`, commit format is `grechman(step N): <description>`, and three batch commits always occur. Execution mode is chosen by explicit rule, not randomly. Minor variability possible in LLM-driven brainstorming and planning output across runs.

**Safety — 9**
`main`/`master` is never touched. Commits are gated on verification. Security review covers XSS, SQL injection, exposed paths, auth logic, secrets, rate limiting. Clean working-tree check precedes branching. Depwire pre-edit dependency check prevents silent downstream breakage. No observable unsafe events in the design.

**Traceability — 9**
`CLAUDE.md` session log, structured commit messages, `docs/plans/*.md` design artifacts, ADR support, fallback file with last stable SHA, per-task YAML reports, and session summary YAML together make the session fully reconstructable. Score is 9 rather than 10 because log hygiene depends on agents consistently following instructions across all steps.

**Scalability — 8**
Three execution modes map to task complexity. Hard mode extends the iteration cap to 25. Context overflow fix keeps orchestrator memory flat across 14+ sequential tasks. `scale_factor` = 25 / 8 = 3.1×. Does not yet describe horizontal scaling (parallel orchestrators, multi-repo tasks) so 9–10 is not warranted.

**Extensibility — 8**
Optional skills and MCPs are checked at startup and used if present (graceful degradation). Pattern is clean: new capability = new MCP or skill, no core changes needed. Ontology system and depwire show the integration model works. Minor friction: no plugin manifest or formal extension API; extending requires editing prompt files directly.

**Verification Quality — 8**
Security review is mandatory before final commit (not skippable). Commits are blocked until verification passes. Depwire pre-edit hook catches dependency breakage before commit. Structured task reports surface blockers early. Does not enforce test-first development or automated test runs, which would push this toward 9–10.
