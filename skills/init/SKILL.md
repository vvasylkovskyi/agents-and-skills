---
name: init
description: Initializes a repo for autopilot-workflow by discovering validation commands, writing CLAUDE.md, creating roadmap folders, extracting agent context files (workflow.md, testing-checklist.md), and updating settings.json with allowed tools. Use this skill when setting up a new repo or when CLAUDE.md needs to be generated or refreshed. Trigger when the user says "init repo", "set up autopilot workflow", "initialize context", or when starting work in a new repository.
---

# Repo Initialization

One-command setup for autopilot-workflow. This skill:

1. Inspects the repo to discover validation commands
2. Writes or updates `CLAUDE.md` with discovered commands and conventions
3. Creates `.claude/context/roadmap/features/` and `.claude/context/roadmap/bugs/` directories
4. Updates `.gitignore` to exclude roadmap directories
5. Extracts agent-consumable files: `.claude/context/workflow.md` and `.claude/context/testing-checklist.md`
6. Updates `.claude/settings.json` with allowed tools and paths

After running this skill, the repo is ready for `/autopilot-workflow:design-doc-decomposer` and `/autopilot-workflow:implementation-orchestrator`.

---

## Parameters

No parameters required. The current working directory is the repo root.

---

## Step 1 — Discover repo type and validation commands

Inspect the repo to identify the tech stack:

```bash
# Package managers and build tools
ls package.json yarn.lock pnpm-lock.yaml bun.lockb Gemfile Cargo.toml go.mod requirements.txt pyproject.toml mix.exs 2>/dev/null

# Package manager scripts
cat package.json 2>/dev/null | grep -E '"scripts"' -A 30
cat Makefile 2>/dev/null

# CI configuration
cat .github/workflows/*.yml 2>/dev/null | head -100
cat .buildkite/pipeline.yml 2>/dev/null | head -60

# Existing docs
cat README.md 2>/dev/null | head -80
```

Based on what you find, determine validation commands using this logic:

### Node / pnpm

If `pnpm-lock.yaml` exists: `pnpm lint`, `pnpm build`, `pnpm test`, `pnpm typecheck` (whichever scripts exist in package.json)

### Node / npm or yarn

Same priority list but with `npm run` or `yarn` prefix.

### Ansible

If `playbooks/` exists: `ansible-lint playbooks/`

### Shell scripts

If `scripts/*.sh` exists: `shellcheck scripts/*.sh`

### Python

If `pyproject.toml` exists: `pytest`, `ruff check .`, `black --check .`, `mypy .` (whichever are configured)

### Go

If `go.mod` exists: `go build ./...`, `go test ./...`, `go vet ./...`

### Terraform

If `.tf` files exist: `terraform fmt -check`, `terraform validate`

### Make

If `Makefile` has `test`, `lint`, or `check` targets, use those.

---

## Step 2 — Write or update CLAUDE.md

Check if CLAUDE.md exists:

```bash
cat CLAUDE.md 2>/dev/null
```

**If CLAUDE.md exists:** Preserve all existing content. Only add missing sections.

**If CLAUDE.md does not exist:** Write it from scratch using this template:

````markdown
# Development Conventions

## Branch Naming

- `feat/<description>` — new features
- `fix/<description>` — bug fixes
- `docs/<description>` — documentation
- `refactor/<description>` — code refactoring
- `issue-<number>-<slug>` — issue-driven work

## Commit Messages

Format: `type: description`

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

Example: `feat: add ARM64 validation check`

Include `Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>` for AI-assisted commits.

# Validation Commands

Run in order. Stop and fix before continuing if any command fails.

```bash
<discovered command 1>
<discovered command 2>
```

# Testing Checklist

All commands must exit 0. A non-zero exit code is a hard failure — do not commit, do not mark PASS.

- [ ] Branch follows naming convention
- [ ] Commits follow format (`type: description`)
- [ ] No merge conflicts with base branch
- [ ] All validation commands pass
````

**If no validation commands were discovered**, write:

```markdown
# Validation Commands

> ⚠️ No validation commands were auto-detected for this repo.
> Add the commands that should run before every commit here.
```

---

## Step 3 — Extract workflow.md and testing-checklist.md

From the CLAUDE.md you just wrote or updated, extract two agent-consumable files.

Create the context directory:

```bash
mkdir -p .claude/context
```

**Extract `.claude/context/workflow.md`:**

From CLAUDE.md, extract branch naming, commit message format, and general conventions.

Write to `.claude/context/workflow.md`:

```markdown
# Workflow

> Auto-extracted from CLAUDE.md on <date>. Do not edit manually — re-run /autopilot-workflow:init to refresh.

## Branch Naming

<extracted content>

## Commit Messages

<extracted content>

## General Conventions

<extracted content, or "No additional conventions specified." if absent>
```

**Extract `.claude/context/testing-checklist.md`:**

From CLAUDE.md, extract validation commands and hard-gate rules.

Write to `.claude/context/testing-checklist.md`:

````markdown
# Testing Checklist

> Auto-extracted from CLAUDE.md on <date>. Do not edit manually — re-run /autopilot-workflow:init to refresh.

## Hard Rules

All commands must exit 0. Non-zero exit is a hard failure — do not commit, do not mark PASS.

## Validation Commands

Run in order. Stop and fix before continuing if any command fails.

```bash
<command 1>
<command 2>
```

## Acceptance Checklist

- [ ] <item 1>
- [ ] <item 2>
````

---

## Step 4 — Create roadmap folders and update .gitignore

Create directories for feature and bug specs:

```bash
mkdir -p .claude/context/roadmap/features .claude/context/roadmap/bugs
```

Update `.gitignore` to exclude these folders:

```bash
cat .gitignore 2>/dev/null
```

If `.claude/context/roadmap/` is not already ignored, append:

```
# Autopilot workflow roadmap directories
.claude/context/roadmap/features/
.claude/context/roadmap/bugs/
```

**Important:** Only add if not already present. Do not duplicate entries.

---

## Step 5 — Update .claude/settings.json

Extract validation commands and add them to `allowedTools` so Claude Code does not prompt during agent sessions.

Read current settings:

```bash
cat .claude/settings.json 2>/dev/null
```

**Merge strategy:**

- If `.claude/settings.json` does not exist, create it from the base template below
- If it exists, read current `allowedTools` and add new entries that aren't present
- Wrap each command as `Bash(<command> *)` — e.g. `pnpm test` → `"Bash(pnpm *)"`
- Never remove existing entries

**Base template:**

```json
{
  "allowedTools": [
    "Bash(git worktree *)",
    "Bash(git checkout *)",
    "Bash(git stash *)",
    "Bash(git stash push *)",
    "Bash(git stash pop)",
    "Bash(git fetch *)",
    "Bash(git pull *)",
    "Bash(git push *)",
    "Bash(git branch *)",
    "Bash(git add *)",
    "Bash(git commit *)",
    "Bash(git status *)",
    "Bash(git diff *)",
    "Bash(git log *)",
    "Bash(git merge *)",
    "Bash(gh pr create *)",
    "Bash(gh pr merge *)",
    "Bash(gh pr list *)",
    "Bash(mkdir -p *)",
    "Bash(ls *)",
    "Bash(cat *)",
    "Bash(pwd)",
    "Bash(grep -q *)",
    "Bash(echo * >> *)",
    "Bash(cd .claude/worktrees/session && git *)",
    "Bash(cd .claude/worktrees/session && git *; git *)",
    "Bash(cd .claude/worktrees/session && git * 2>/dev/null; git *)",
    "Bash(pnpm *)"
  ],
  "allowedPaths": [
    ".claude/context/workflow.md",
    ".claude/context/testing-checklist.md",
    ".claude/context/roadmap/features",
    ".claude/context/roadmap/features/**",
    ".claude/context/roadmap/features/tasks",
    ".claude/context/roadmap/features/tasks/**",
    ".claude/context/roadmap/bugs",
    ".claude/context/roadmap/bugs/**",
    ".claude/context/roadmap/bugs/tasks",
    ".claude/context/roadmap/bugs/tasks/**",
    ".claude/worktrees/**",
    ".gitignore",
    "CLAUDE.md"
  ]
}
```

Write the merged result to `.claude/settings.json`.

---

## Step 6 — Confirm to the user

```
✅ Repo initialized for autopilot-workflow

  CLAUDE.md                       — validation commands and conventions
  .claude/context/workflow.md             — branch naming, commit format
  .claude/context/testing-checklist.md    — validation commands, hard gates
  .claude/context/roadmap/features/       — created for feature specs
  .claude/context/roadmap/bugs/           — created for bug specs
  .gitignore                      — updated to exclude roadmap directories
  .claude/settings.json           — allowedTools and allowedPaths configured

Detected stack: <e.g. Node/pnpm, Python/pytest, Go>
Validation commands added:
  <command 1>
  <command 2>

Validation commands added to allowedTools:
  Bash(pnpm *)
  Bash(pytest *)
  ...

Ready for:
  /autopilot-workflow:design-doc-decomposer
  /autopilot-workflow:implementation-orchestrator
```

---

## Rules

- **Never overwrite existing CLAUDE.md content** — only add missing sections
- **Never invent commands** — only use commands evidenced by repo files
- **Use exact command names** from package.json scripts, Makefile targets, etc.
- **Never remove existing settings.json entries** — only add
- **Be a faithful transcriber** — copy commands exactly as written
- **Date the output** in extracted files so agents can tell if files are stale
