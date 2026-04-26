---
name: design-doc-decomposer
description: Breaks down a complex design document (RFC, ADR, or architectural proposal) into a reviewed, ordered list of discrete implementable feature slices, then hands each slice to feature-doc-creator to generate individual feature docs. Use this skill when the user wants to bootstrap a new package or system from a design doc, says "break this design doc into features", "create feature docs from this RFC", or "decompose this proposal into implementable units". Always use this skill when the input is a multi-component design document rather than a single focused feature idea — do not try to decompose design docs inline without this skill.
---

# Design Doc Decomposer

Reads a design document and produces a human-reviewed breakdown of implementable feature slices, then generates one feature doc per slice using feature-doc-creator.

You do not implement anything. You produce a decomposition for human approval, then generate feature docs.

---

## Parameters

Extract from the user's input or conversation:

- `design-doc-path` — path to the design doc file, OR use the document already in context
- `repo-path` — absolute path to the current repo (the single target for all features); ask if not clear
- `reference-repo-path` — _(optional)_ path to a similar repo to draw patterns and conventions from; if provided, use it in Step 1b to inform how feature docs are written
- `features-output-path` — where to write feature docs; defaults to `<repo-path>/.claude/context/roadmap/features/`

---

## Step 1 — Read and internalize the design doc

If a file path is given, read it:

```bash
cat <design-doc-path>
```

If the document is already in the conversation context, work from it directly.

Identify the following from the document:

- **Goals** — what the system is trying to achieve
- **Components** — distinct technical areas (infra, CI/CD, runtime, dev tooling, integrations, etc.)
- **Dependencies** — which components must exist before others can be built
- **Non-goals** — explicitly deferred work, do not create features for these
- **Future work** — items flagged as future investigation, do not create features for these unless the user asks

### Step 1a — Explore the current repo

Get a picture of the existing codebase to understand what already exists vs. what needs to be built:

```bash
cd <repo-path>

# Repo structure
find . -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/.claude/*' \
  | sort | head -80

# Recent git history
git log --oneline -15

# Existing feature docs if any
ls .claude/outputs/roadmap/features/ 2>/dev/null | sort
```

Read any existing feature docs to understand numbering and style already in use.

### Step 1b — Explore the reference repo (if provided)

If a `reference-repo-path` was given, explore it to extract conventions and patterns to follow:

```bash
cd <reference-repo-path>

# Overall structure
find . -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/.claude/*' \
  | sort | head -80

# Existing feature docs for style reference
ls .claude/outputs/roadmap/features/ 2>/dev/null | sort | tail -5

# Read the most recent 2-3 feature docs to understand doc style
cat .claude/outputs/roadmap/features/<most-recent>.md
```

Note the patterns observed: file naming convention, section structure, level of detail, how dependencies are expressed. Pass these as style notes to feature-doc-creator in Step 4.

---

## Step 2 — Propose the feature breakdown

Output a structured decomposition for the user to review. Do not create any files yet.

Format:

```
## Proposed Feature Breakdown

<one sentence framing the decomposition rationale>

| # | Filename | Title | Depends on | Rationale for split |
|---|----------|-------|------------|---------------------|
| 1 | 01-repo-scaffold.md | Repository and package scaffolding | — | Must exist before any other work |
| 2 | 02-terraform-s3-infra.md | S3 bucket and Terraform setup | — | Independent of app code, can proceed in parallel |
| 3 | 03-... | ... | ... | ... |

Numbers are zero-padded (`01`, `02`, ...) and continue from the highest existing file in `<features-output-path>`. They are fixed once assigned — do not renumber after approval.

## What is excluded (and why)

- <item from Non-Goals section> — explicitly marked non-goal in design doc
- <future work item> — flagged as future work, no implementation spec yet

## Decomposition notes

<Any ambiguous splits, assumptions made, or items where you need the user's input>
```

### Decomposition rules

- **One feature = one independently mergeable PR.** If two things always ship together and have no value separately, they are one feature.
- **Infrastructure is always a separate feature** from application code that depends on it.
- **CI/CD pipeline setup** is its own feature — it is not part of the first feature that happens to use it.
- **Dev tooling** (preview server, local scripts) is separate from build/export pipeline.
- **Don't split by file** — split by capability boundary. Two files in the same logical unit are one feature.
- **Respect non-goals and future work** — do not create features for anything the design doc explicitly defers.
- **Flag ambiguity** — if you are unsure whether something is in or out of scope, say so rather than deciding silently.
- **Aim for 4–10 features** for a typical design doc. Fewer than 4 suggests under-decomposition; more than 10 suggests splitting too fine.

---

## Step 3 — Wait for human approval

Stop here. Present the breakdown and ask:

```
Does this decomposition look right?

You can:
- Approve it as-is to generate all feature docs
- Edit the table (merge items, split items, reorder, remove)
- Tell me what to change and I'll revise

Once approved, I'll generate one feature doc per row using feature-doc-creator.
```

Do not proceed until the user explicitly approves or provides edits.

If the user provides edits, revise the breakdown and present it again. Repeat until approved.

---

## Step 4 — Generate feature docs

Ensure the output directory exists:

```bash
mkdir -p <features-output-path>
```

**Determine the starting sequence number** before generating any docs. Check what already exists:

```bash
ls <features-output-path> 2>/dev/null | sort | tail -1
```

If files exist (e.g. `05-something.md`), the next number is `06`. If the directory is empty, start at `01`. Zero-pad to two digits: `01`, `02`, ..., `09`, `10`, `11`.

Assign each approved row a sequence number in order before invoking feature-doc-creator for any of them. The sequence number is fixed at assignment time — it does not change if a doc fails or is skipped.

For each approved row in the breakdown table, invoke feature-doc-creator in sequence.

Pass to feature-doc-creator for each slice:

- The **full design doc** as context (so it can explore relevant patterns and understand the system)
- The **specific slice description**: slug, title, rationale, dependencies
- The **assigned sequence number** and expected filename
- The `repo-path` and `features-output-path`
- Any **reference repo style notes** extracted in Step 1b (if a reference repo was provided)

Invoke it as:

```
Use the feature-doc-creator skill.

Repo path: <repo-path>
Features output path: <features-output-path>
Output filename: <NN>-<slug>.md  ← use exactly this filename, do not derive your own

Feature description:
  Title: <title>
  Slug: <slug>
  Depends on: <depends-on>
  Rationale: <rationale>

Design doc context (full document for reference):
<full design doc content>

<if reference repo was provided:>
Reference repo style notes (follow these conventions):
<style notes extracted from reference repo in Step 1b>

Scope constraint: Only document the capabilities described in this slice.
Do not include work from other slices even if referenced in the design doc.
Dependencies on other slices should be noted as assumptions, not re-specified.
```

After each feature doc is created, confirm to the user:

```
✅ <n>/<total>: <slug> → <features-output-path>/<filename>
```

---

## Step 5 — Final summary

After all feature docs are generated:

```
Decomposition complete.

Generated <N> feature docs in <features-output-path>:

  01-repo-scaffold.md             → Repository and package scaffolding
  02-terraform-s3-infra.md        → S3 bucket and Terraform setup
  ...

Suggested implementation order (based on dependencies):
  1. <slug> — no dependencies
  2. <slug> — no dependencies (parallel with 1)
  3. <slug> — requires 1
  ...

Excluded from decomposition:
  - <item> — <reason>

Next step: run implementation-orchestrator against <features-output-path> to implement these features.
```

---

## Rules

- **Never generate feature docs before human approval of the breakdown** — the decomposition is the hard part and requires human judgment
- **Never include non-goals or future work** as features unless the user explicitly asks
- **Always pass the full design doc** to feature-doc-creator as context — individual slices lose meaning without it
- **Scope constraint is mandatory** — feature-doc-creator will naturally expand scope; the constraint message prevents overlap between docs
- **Sequential generation only** — do not attempt parallel feature doc generation; ordering matters for dependency clarity
- **If the design doc is ambiguous**, surface the ambiguity in Step 2 rather than resolving it silently
- **Decomposition notes are required** — always explain assumptions, even if the breakdown seems obvious
- **Reference repo is read-only** — never modify it; only extract style patterns and conventions from it
- **If no reference repo is provided**, feature-doc-creator derives style from the current repo's existing feature docs (if any) or uses its defaults
