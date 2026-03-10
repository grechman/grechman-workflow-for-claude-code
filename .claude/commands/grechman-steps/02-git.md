# Grechman Step 2 — Git Setup

You are a Grechman agent. Do everything below, then write your report.

## Inputs (from orchestrator prompt)
- Git: on (this step is skipped if off)
- GitHub: on | off
- Complexity: medium | hard
- Branch slug (from task name)

## Procedure

### 1. Clean Tree Check
```bash
git status --porcelain
```
If non-empty → report BLOCKED with file list. User must commit/stash first.

### 2. Branch Collision Check
```bash
git show-ref --verify --quiet refs/heads/grechman/<slug>
```
If exists → report BLOCKED. Orchestrator will ask user: rename / delete / resume.

### 3. Create Branch
```bash
git checkout -b grechman/<slug>
```

### 4. Worktree (complexity=hard only)
If `--complexity hard`: invoke `superpowers:using-git-worktrees` for isolated workspace.

## Report

Write to `.grechman/task-reports/task_2.yaml`:
```yaml
task_id: 2
status: completed | blocked
branch: <branch name>
init_sha: <SHA of current HEAD>
worktree_path: <path or null>
blockers: <if any, else null>
```

Return to orchestrator ONLY: `"Task 2 complete. Report: .grechman/task-reports/task_2.yaml"`
