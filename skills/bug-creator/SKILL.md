---
name: bug-creator
description: Creates a complete, enriched bug task document from a short bug description and snippet. Explores the current repo state to understand context, relevant files, and likely cause, then writes a self-contained bug .md file that an agent can pick up and solve without further input. Use this skill when the user reports a bug, describes unexpected behaviour, or pastes a broken interaction snippet.
---

The user has reported a bug with a short description and/or snippet. Your job is to explore the repo, understand the context around the bug, and produce a complete bug document that a future agent can solve independently.

You do not fix the bug. You only write the document.

## Parameters

Extract from the user's input:

- `repo-name` — which repo the bug is in (ask if not clear)
- `repos-base-path` — defaults to `/Users/vvasylkovskyi/git/IaC-Toolbox/git/`
- `bug-description` — what the user observed
- `bug-snippet` — any terminal output, UI text, or code snippet provided

## Step 1 — Understand the bug area

Read the user's description and snippet carefully. Identify the likely area of the codebase.

Explore the repo to find relevant files:

```bash
cd <repos-base-path>/<repo-name>

# Get a picture of recent changes
git log --oneline -20

# Find files related to the bug area by keyword
grep -r "<relevant keyword>" --include="*.ts" --include="*.js" --include="*.sh" --include="*.yaml" -l

# Read the most relevant files in full
```

Read enough to understand what the code currently does in the bug area, what it should be doing, and the likely root cause.

## Step 2 — Derive slug and filename

Create a short kebab-case slug from the bug description (e.g. `credentials-not-prefilled-on-reuse`).

Check existing bug files to pick the next sequential number:

```bash
ls .claude/context/roadmap/bugs/ 2>/dev/null | sort | tail -1
```

Name the file `<next-number>-<slug>.md`.

## Step 3 — Write the bug document

Write to `.claude/context/roadmap/bugs/<filename>`:

```markdown
---
status: draft
completed_date:
pr_url:
---

# Bug: <short title>

## Summary

<1-2 sentence description of what is broken, clear enough for an agent with no prior context>

## Repo

<repo-name>

## Observed behaviour

<paste or paraphrase the user's snippet/description exactly>

## Expected behaviour

<what should happen instead, inferred from the user's intent and the codebase>

## Relevant files

<list of files identified in Step 1 that are most likely involved>

## Context

<2-4 sentences explaining what the code currently does in this area, based on your exploration>

## Likely cause

<your best hypothesis grounded in what you actually read, e.g. "The prompt for Docker Hub username does not check credentials.ini before rendering, so it always shows an empty field regardless of saved state">

## Acceptance criteria

- [ ] <specific, testable condition 1>
- [ ] <specific, testable condition 2>
- [ ] <specific, testable condition 3>

## Validation

<which commands from ~/.claude/resources/testing-checklist.md are most relevant to verify this fix>
```

## Step 4 — Confirm to user

```
🐛 Bug document created: .claude/context/roadmap/bugs/<filename>
Likely cause: <one line summary>
Relevant files: <comma-separated list>
```

## Rules

- Never fix the bug — only document it
- Always explore the repo before writing — do not write from the user's description alone
- The document must be self-contained — a future agent needs nothing else to attempt a fix
- Likely cause must be grounded in what you actually read, not guessed
- Acceptance criteria must be concrete and testable
- Always include `status: draft` frontmatter — this is how the executor knows to pick it up
- If the repo or bug area is ambiguous, ask before exploring
