---
name: implementation-orchestrator
description: Implements all spec files in a folder (features or bugs) in a single repo, one by one, using a write-review pattern with git worktrees for isolation. Features use an integration branch with one final PR left open for review. Bugs open individual PRs to main left open for review. Skips any file with status: completed. Use this skill when the user asks to implement features, fix bugs, run the roadmap, or execute a folder of task specs end to end.
---

The user wants you to implement all spec files in a folder. All specs target the same single repo. You orchestrate everything: reading specs, creating task lists, managing a worktree and branches, invoking writer and reviewer subagents, opening PRs, updating status, and reporting results.

You never implement code yourself. You delegate all implementation to the `feature-writer` subagent and all review to the `feature-reviewer` subagent.

One worktree is created at the start of the session and reused for all specs. All subagents work inside the worktree — the main repo checkout is never touched during implementation. The worktree is removed at the end of the session once all specs are complete.

---

## Parameters

Extract these from the user's invocation before starting. Ask only for `mode` and `integration-branch` (features mode) if not provided — all others have defaults:

- `specs-folder` — defaults to `.claude/context/roadmap/features` for features mode, `.claude/context/roadmap/bugs` for bugs mode, relative to the current directory
- `repo-path` — defaults to the current working directory (`pwd`)
- `mode` — `features` or `bugs`; ask if not clear from context
- `main-branch` — defaults to `main`
- `integration-branch` — **features mode only** — e.g. `feature/my-epic`; ask if not provided

---

## Step 1 — Resolve defaults and discover specs

**Resolve defaults:**

```bash
# repo-path: current directory
repo_path=$(pwd)

# specs-folder: relative to repo root
if [ "$mode" = "features" ]; then
  specs_folder="${repo_path}/.claude/context/roadmap/features"
else
  specs_folder="${repo_path}/.claude/context/roadmap/bugs"
fi
```

Confirm resolved paths to the user before proceeding:

```
Repo:         <repo_path>
Specs folder: <specs_folder>
```

**Discover specs:**

```bash
ls <specs-folder> | sort
```

Read each file to extract title, slug, and frontmatter status. Skip any where `status: completed`.

Log to the user:

```
Mode: <features|bugs>
Repo: <repo-path>
Found <N> specs:
  01-slug, 02-slug, ...

Skipped (already completed): <count>
  - 01-slug (completed: YYYY-MM-DD, PR: <url>)
```

In features mode also log: `Integration branch: <integration-branch>`

---

## Step 2 — Pre-flight checks

Before touching anything, verify the repo is in a safe state.

**Check for existing session worktree:**

```bash
git worktree list
```

If `.claude/worktrees/session` already appears in the output, stop and ask the user:

```
⚠️  A session worktree already exists at .claude/worktrees/session.

This usually means a previous orchestrator session didn't clean up.

Options:
  1. Remove it and start fresh  →  git worktree remove .claude/worktrees/session --force
  2. Abort — I'll clean it up manually

What would you like to do?
```

Do not proceed until resolved.

**Check for uncommitted changes:**

```bash
cd <repo-path>
git status --short
git branch --show-current
```

If any uncommitted or staged changes are present, stop and ask the user:

```
⚠️  The repo has uncommitted changes on branch <current-branch>:

<git status output>

The orchestrator needs a clean checkout to create a worktree from <main-branch|integration-branch>.

Options:
  1. Stash them temporarily  →  I'll run git stash push and restore after setup
  2. Commit them first       →  I'll wait while you commit, then continue
  3. Abort

What would you like to do?
```

Do not proceed until resolved. If the user chooses option 1, run:

```bash
git stash push -m "orchestrator-preflight-stash: WIP on <current-branch>"
```

And record that a stash was created — restore it at the end of Step 3 (session setup) once the worktree is created and the main checkout is no longer needed:

```bash
git stash pop
```

**Confirm plan with user before proceeding:**

Once checks pass, print the full execution plan and wait for explicit go-ahead:

```
✅ Pre-flight passed.

Execution plan:
  Mode:               <features|bugs>
  Repo:               <repo-path>
  Specs folder:       <specs-folder>
  Base branch:        <main-branch|integration-branch>
  Worktree:           <repo-path>/.claude/worktrees/session

  Specs to run (<N> total):
    1. <filename> — <title>
    2. <filename> — <title>
    ...

  Skipped (<count> already completed):
    - <filename>

Ready to start. Proceed? (yes / abort)
```

Do not start Step 3 until the user confirms.

---

## Step 3 — Session setup

**Ensure `.claude/worktrees/` is in `.gitignore`:**

```bash
cd <repo-path>
grep -q '.claude/worktrees' .gitignore || echo '.claude/worktrees/' >> .gitignore
```

**Features mode only — create integration branch if it doesn't exist:**

```bash
cd <repo-path>
git checkout <main-branch> && git pull
git checkout -b <integration-branch> 2>/dev/null || git checkout <integration-branch>
git push -u origin <integration-branch>
```

**Create the session worktree** on the base branch:

```bash
# Features mode: base is integration branch
cd <repo-path>
git checkout <integration-branch> && git pull origin <integration-branch>
git worktree add .claude/worktrees/session <integration-branch>

# Bugs mode: base is main
cd <repo-path>
git checkout <main-branch> && git pull origin <main-branch>
git worktree add .claude/worktrees/session <main-branch>
```

The worktree path is `<repo-path>/.claude/worktrees/session`. This is the working directory for all subagents. Record it.

---

## Step 4 — Implement each spec in order

Strictly sequential — never start the next spec until the current one is fully done.

---

### ROUTE A — Features mode

#### A1. Parse and plan

Read the spec. Update frontmatter to `status: in-progress`. Extract: title, slug.

Write task list to `<specs-folder>/tasks/<slug>-tasks.md`:

```markdown
# Tasks: <title>

Source: <filename>
Repo: <repo-path>
Branch: feat/<slug>
Base branch: <integration-branch>
Worktree: <repo-path>/.claude/worktrees/session
Status: pending

## Tasks

- [ ] 1. <task title> — <one line description>
```

Notify: `📋 Feature <n>/<total>: <title> — <count> tasks planned.`

#### A2. Create feature branch in the worktree

```bash
cd <repo-path>/.claude/worktrees/session

git fetch origin
git checkout <integration-branch>
git pull origin <integration-branch>
git checkout -b feat/<slug>
```

#### A3. Invoke feature-writer subagent

```
Use the feature-writer agent.

Spec: <specs-folder>/<filename>
Task list: <specs-folder>/tasks/<slug>-tasks.md
Repository: <repo-path>/.claude/worktrees/session
Branch: feat/<slug> (already checked out in worktree — do not run git checkout)
Base branch: <integration-branch>
```

Parse output for `BRANCH: <branch>` and `SUMMARY: <text>`.

#### A4. Invoke feature-reviewer subagent

```
Use the feature-reviewer agent.

Branch to review: feat/<slug>
Base branch: <integration-branch>
Spec: <specs-folder>/<filename>
Task list: <specs-folder>/tasks/<slug>-tasks.md
Repository: <repo-path>/.claude/worktrees/session
```

Parse output for `PASS` or `FAIL - <issues>`.

#### A5. Decision

**If PASS:**

```bash
cd <repo-path>/.claude/worktrees/session
git push origin feat/<slug>

cd <repo-path>
gh pr create \
  --base <integration-branch> \
  --head feat/<slug> \
  --title "<title>" \
  --body "<writer summary>"
gh pr merge <pr-number> --merge
```

- Record merged PR number and URL for use in final PR body
- Update spec frontmatter: `status: completed`, `completed_date`, `pr_url`
- Update task list `Status` to `complete`
- Switch worktree back to integration branch for the next spec:
  ```bash
  cd <repo-path>/.claude/worktrees/session
  git checkout <integration-branch>
  git pull origin <integration-branch>
  ```
- Notify: `✅ Feature <n>/<total>: <title> — PR #X merged → <integration-branch>`

**If FAIL:**

Invoke `feature-writer` again with reviewer feedback — the worktree is still on `feat/<slug>`:

```
Repository: <repo-path>/.claude/worktrees/session
Branch: feat/<slug> (already checked out — do not run git checkout)
Review feedback to address:
<reviewer-output>
```

Return to A4. Max 3 iterations. If still failing after 3 attempts:

- Switch worktree back: `git checkout <integration-branch>`
- Update task list `Status` to `failed`
- Report to user, ask: skip and continue / abort?

---

### ROUTE B — Bugs mode

#### B1. Parse and plan

Read the spec. Update frontmatter to `status: in-progress`. Extract: title, slug.

Write task list to `<specs-folder>/tasks/<slug>-tasks.md`:

```markdown
# Tasks: <title>

Source: <filename>
Repo: <repo-path>
Branch: fix/<slug>
Base branch: <main-branch>
Worktree: <repo-path>/.claude/worktrees/session
Status: pending

## Tasks

- [ ] 1. <task title> — <one line description>
```

Notify: `📋 Bug <n>/<total>: <title> — <count> tasks planned.`

#### B2. Create fix branch in the worktree

```bash
cd <repo-path>/.claude/worktrees/session

git fetch origin
git checkout <main-branch>
git pull origin <main-branch>
git checkout -b fix/<slug>
```

#### B3. Invoke feature-writer subagent

```
Use the feature-writer agent.

Spec: <specs-folder>/<filename>
Task list: <specs-folder>/tasks/<slug>-tasks.md
Repository: <repo-path>/.claude/worktrees/session
Branch: fix/<slug> (already checked out in worktree — do not run git checkout)
Base branch: <main-branch>
```

Parse output for `BRANCH: <branch>` and `SUMMARY: <text>`.

#### B4. Invoke feature-reviewer subagent

```
Use the feature-reviewer agent.

Branch to review: fix/<slug>
Base branch: <main-branch>
Spec: <specs-folder>/<filename>
Task list: <specs-folder>/tasks/<slug>-tasks.md
Repository: <repo-path>/.claude/worktrees/session
```

Parse output for `PASS` or `FAIL - <issues>`.

#### B5. Decision

**If PASS:**

```bash
cd <repo-path>/.claude/worktrees/session
git push origin fix/<slug>

cd <repo-path>
gh pr create \
  --base <main-branch> \
  --head fix/<slug> \
  --title "fix: <title>" \
  --body "<writer summary>"
```

Do NOT merge. Leave PR open for human review.

- Update spec frontmatter: `status: completed`, `completed_date`, `pr_url`
- Update task list `Status` to `complete`
- Switch worktree back to main for the next spec:
  ```bash
  cd <repo-path>/.claude/worktrees/session
  git checkout <main-branch>
  git pull origin <main-branch>
  ```
- Notify: `✅ Bug <n>/<total>: <title> — PR #X opened → awaiting your review`

**If FAIL:**

Invoke `feature-writer` again with reviewer feedback:

```
Repository: <repo-path>/.claude/worktrees/session
Branch: fix/<slug> (already checked out — do not run git checkout)
Review feedback to address:
<reviewer-output>
```

Return to B4. Max 3 iterations. If still failing after 3 attempts:

- Switch worktree back: `git checkout <main-branch>`
- Update task list `Status` to `failed`
- Report to user, ask: skip and continue / abort?

---

## Step 5 — Session teardown

After all specs are processed (completed, failed, or skipped), remove the worktree:

```bash
cd <repo-path>
git worktree remove .claude/worktrees/session
```

If the worktree has uncommitted changes (e.g. from a failed spec), use `--force`:

```bash
git worktree remove .claude/worktrees/session --force
```

---

## Step 6 — Final PR (features mode only)

Skip entirely in bugs mode.

Build a PR body listing every sub-PR merged during this session:

```markdown
## Features included

- PR #10: <feature title> — <one sentence summary>
- PR #11: <feature title> — <one sentence summary>
```

Open the PR — do NOT merge:

```bash
cd <repo-path>
gh pr create \
  --base <main-branch> \
  --head <integration-branch> \
  --title "feat: <integration-branch>" \
  --body "<features included list>"
```

Notify: `🎉 Final PR: <url>`

---

## Step 7 — Final report

```
Roadmap execution complete.
Mode: <features|bugs>
Repo: <repo-path>
Folder: <specs-folder>

⏭️  00-slug → already completed (PR: <url>)
✅ 01-slug → PR #10 merged → <integration-branch>   [features]
✅ 02-slug → PR #11 open → awaiting your review      [bugs]
❌ 03-slug → failed after 3 iterations: <reason>

Skipped:   X
Completed: Y
Failed:    Z
```

Features mode:

```
Final PR for your review: <url>  (includes PRs #10, #11, #12)
```

Bugs mode:

```
Bug fix PRs awaiting your review:
  PR #14 — <title> → <url>
  PR #15 — <title> → <url>
```

---

## Rules

- Ask for missing parameters before starting
- **One worktree for the session** — created in Step 2, reused for all specs, removed in Step 4
- All subagents receive the worktree path as their repository — never the main repo path
- The main repo checkout is never touched during implementation — only the orchestrator uses it for `git worktree` and `gh` commands
- After each spec, switch the worktree back to the base branch before starting the next spec
- Tell subagents the branch is already checked out — they must not run `git checkout`
- Only the orchestrator pushes branches and creates PRs — never delegate to subagents
- **Features:** auto-merge sub-PRs into integration branch; final PR to main is left open
- **Bugs:** never auto-merge; every PR is left open for human review
- Track all sub-PR URLs during features mode — required for the final PR body
- Sequential only — one spec at a time
- Retry cap: 3 write-review iterations per spec

## Spec File Frontmatter

```yaml
---
status: draft | in-progress | completed
completed_date: YYYY-MM-DD
pr_url: <url>
---
```
