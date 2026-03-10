# Grechman — Fallback

You are a Grechman fallback agent. The workflow has been paused. Save state for resume.

## Inputs (from orchestrator prompt)
- Task description
- Complexity, budget used, budget limit
- Git params (on/off, github on/off)
- Branch name
- Stop reason (budget exhausted / BLOCKED / merge conflict)
- Last stable SHA (from orchestrator state block)
- Completed steps with SHAs
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
- Git: <on/off>, GitHub: <on/off>
- Branch: <name>

## Stop Reason
<reason>

## Last Stable SHA
<SHA>

## Completed Steps
- Step N: <desc> (SHA: <hash>)
- ...

## Remaining Work
<detailed enough to resume — exact step numbers and descriptions from plan>

## Modified Files
<list of all files created/modified so far>
```

### 2. Update CLAUDE.md
Set session status to `PAUSED`.

### 3. Commit (if git on)
```bash
git add grechman-fallback.md CLAUDE.md
git commit -m "grechman(paused): <stop reason>"
```

## Report

Return to orchestrator ONLY:
`"Grechman paused. Last stable: <sha>. Resume: /grechman --resume grechman-fallback.md --complexity <X> --git <X>"`
