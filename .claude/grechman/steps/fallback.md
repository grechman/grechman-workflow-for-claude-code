# Grechman — Fallback

You are a Grechman fallback agent. The workflow has been paused. Save state for resume.

## Inputs (from orchestrator prompt)
- Task description
- Complexity, budget used, budget limit
- VCS type (jujutsu or git)
- Git params (on/off, github on/off)
- Branch name
- Stop reason (budget exhausted / BLOCKED / merge conflict)
- Last stable SHA (from orchestrator state block)
- Completed steps with SHAs
- Decision log (from orchestrator DECISIONS block)
- Remaining work description

## Procedure

### 1. Write grechman-fallback.md
```markdown
# Grechman Fallback

## Task
<task description>

## Parameters
- Complexity: <X>
- Budget: <used>/<limit>
- VCS: <jujutsu|git>
- Git: <on/off>, GitHub: <on/off>
- Branch: <name>

## Stop Reason
<reason>

## Last Stable SHA
<SHA>

## Completed Steps
- Step N: <desc> (SHA: <hash>)
- ...

## Decision Log
<full DECISIONS block from orchestrator — preserves continuity on resume>

## Remaining Work
<detailed enough to resume — exact step numbers and descriptions from plan>

## Modified Files
<list of all files created/modified so far>
```

### 2. Update CLAUDE.md
Set session status to `PAUSED`.

### 3. Commit (if git on)
Detect VCS from inputs.

Jujutsu:
```bash
jj commit -m "grechman(paused): <stop reason>"
```
Git:
```bash
git add grechman-fallback.md CLAUDE.md
git commit -m "grechman(paused): <stop reason>"
```

## Return

Return to orchestrator ONLY:
`"Grechman paused. Last stable: <sha>. Resume: /grechman --resume grechman-fallback.md --complexity <X> --git <X>"`
