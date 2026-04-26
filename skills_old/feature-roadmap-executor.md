# Feature-Roadmap-Executor

Reads all feature files in a folder, groups them by target repo, implements them one by one using a write-review pattern, opens one PR per feature into that repo's integration branch, then opens one final PR per repo into main.

## Invocation

```
Run the feature-roadmap-executor workflow.
Features folder: context/roadmap/features/
Repos base path: /Users/vvasylkovskyi/git/IaC-Toolbox/git/
Main branch: main
```

## Branch strategy

```
<repo-a>/main
 └── feature/all-features          ← integration branch per repo (created once each)
      ├── feature/01-<slug>        ← per-feature branch, cut from integration branch
      └── feature/03-<slug>

<repo-b>/main
 └── feature/all-features
      ├── feature/02-<slug>
      └── ...
```

- Each repo gets its own `feature/all-features` integration branch, created the first time a feature targets that repo.
- Every feature branch is cut from its repo's `feature/all-features` (not main), so it always starts from the latest merged work in that repo.
- Each feature opens a PR into its repo's `feature/all-features`.
- Orchestrator merges each feature PR immediately after PASS.
- At the end, one final PR per repo: `feature/all-features` → `main`.

---

## Workflow

### 1. Discover and group features

List all feature files sorted by filename:

```bash
ls context/roadmap/features/ | sort
```

Read each file to extract the target repo. Build a map:

```
01-auth.md          → repo: api-server
02-payments.md      → repo: api-server
03-dashboard.md     → repo: frontend
...
```

Log to user:

```
Found 9 features across N repos:
  api-server: 01-auth, 02-payments, ...
  frontend:   03-dashboard, ...
```

---

### 2. Setup integration branches (once per repo)

For each unique repo discovered:

```bash
cd /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
git checkout main && git pull
git checkout -b feature/all-features
git push -u origin feature/all-features
```

Skip this step for a repo if `feature/all-features` already exists.

---

### 3. For each feature file (in order 01 → 09):

#### A. Parse & plan

Read the feature file. Extract:

- Target repo name
- Feature title
- Slug from filename (`01-auth.md` → `01-auth`)
- Full repo path: `/Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>`

Write a task list to disk at `context/roadmap/features/tasks/<slug>-tasks.md`:

```markdown
# Tasks: <feature title>

Source: <feature-filename>
Repo: <repo-name>
Branch: feature/<slug>
Status: pending

## Tasks

- [ ] 1. <task title> — <one line description>
- [ ] 2. <task title> — <one line description>
```

Notify user: `📋 Feature <n>/9: <title> (repo: <repo-name>) — <count> tasks planned.`

#### B. Cut feature branch from integration branch

```bash
cd /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
git checkout feature/all-features
git pull origin feature/all-features
git checkout -b feature/<slug>
git push -u origin feature/<slug>
```

#### C. Writer phase

Invoke the `feature-writer` subagent:

```
Use the feature-writer agent.

Feature spec: context/roadmap/features/<feature-filename>
Task list: context/roadmap/features/tasks/<slug>-tasks.md
Repository: /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
Branch: feature/<slug> (already exists — check it out, do not recreate)
Base branch: feature/all-features
```

Expected output: `BRANCH: feature/<slug>` and `SUMMARY: <text>`

#### D. Reviewer phase

Invoke the `feature-reviewer` subagent:

```
Use the feature-reviewer agent.

Branch to review: feature/<slug>
Base branch: feature/all-features
Feature spec: context/roadmap/features/<feature-filename>
Task list: context/roadmap/features/tasks/<slug>-tasks.md
Repository: /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
```

Expected output: `PASS` or `FAIL - <issues>`

#### E. Decision

**If PASS:**

```bash
cd /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
git push origin feature/<slug>
gh pr create \
  --repo <repo-name> \
  --base feature/all-features \
  --head feature/<slug> \
  --title "<feature title>" \
  --body "<writer summary>"
gh pr merge <pr-number> --merge --delete-branch
git checkout feature/all-features && git pull origin feature/all-features
```

- Update task list status to `complete`
- Notify: `✅ Feature <n>/9: <title> (repo: <repo-name>) — PR #X merged → feature/all-features`
- Move to next feature

**If FAIL:**

- Invoke `feature-writer` again on the same branch with reviewer feedback:
  ```
  Branch: feature/<slug> (already exists — check it out, do not recreate)
  Review feedback to address:
  <reviewer-output>
  ```
- Return to step D
- **Max 3 iterations** — if still failing after 3 attempts:
  - Update task list status to `failed`
  - Report to user, ask: skip and continue / abort all?

---

### 4. Final PRs (one per repo)

After all features are processed, for each repo that had at least one successful feature:

```bash
gh pr create \
  --repo <repo-name> \
  --base main \
  --head feature/all-features \
  --title "feat: roadmap features for <repo-name>" \
  --body "<list of merged feature PRs and their summaries>"
```

Notify: `🎉 Final PR for <repo-name>: <url>`

---

### 5. Final report

```
Roadmap execution complete.
Features folder: context/roadmap/features/

✅ 01-auth         (api-server)  → PR #10 merged, final PR #15
✅ 02-payments     (api-server)  → PR #11 merged, final PR #15
✅ 03-dashboard    (frontend)    → PR #12 merged, final PR #16
❌ 04-notifications (frontend)   → failed after 3 iterations: <reason>
...

Completed: X/9 features
Failed:    Y/9 features

Final PRs for review:
  api-server → PR #15: <url>
  frontend   → PR #16: <url>
```

---

## Rules

- **Read the feature file before any git work** — repo is determined by the feature spec, not assumed
- **One `feature/all-features` per repo** — created once, shared across all features targeting that repo
- **Every feature branch is cut from its repo's `feature/all-features`** — never from main
- **Every feature PR targets its repo's `feature/all-features`** — never main
- **Orchestrator merges each feature PR immediately after PASS** — keeps integration branch current
- **Writer never pushes or creates PRs** — only commits locally
- **Reviewer is read-only** — never modifies files or commits
- **Sequential only** — one feature at a time
- **Retry cap: 3 iterations per feature**
- **One final PR per repo to main** — the only things that reach main

## Memory format

```markdown
# Roadmap Execution

Date: YYYY-MM-DD

## Feature <n>: <title>

- Repo: <repo-name>
- Branch: feature/<slug>
- Feature PR: <url>
- Writer iterations: <count>
- Review result: PASS / FAIL
- Notes: <any issues>

## Final PRs

- <repo-name>: <url>
```
