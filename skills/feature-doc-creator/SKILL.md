---
name: feature-doc-creator
description: Creates a complete, structured feature proposal document from a conversation or short description. Explores the current repo state to understand context, relevant files, and existing patterns, then writes a self-contained feature .md file that an agent or developer can pick up and implement without further input. Use this skill when the user wants to capture a feature idea, product requirement, CLI change, integration, or architectural decision as a formal document. Trigger whenever the user says "write a feature doc", "create a feature", "make a proposal", "document this feature", or when a feature has been discussed and the user wants to save it as a roadmap item.
---

The user wants to capture a feature as a formal document. Your job is to explore the repo, understand the context, and produce a complete feature document that a future agent or developer can implement independently.

You do not implement the feature. You only write the document.

## Parameters

Extract from the user's input or conversation history:

- `repo-name` — which repo the feature belongs to (ask if not clear)
- `repos-base-path` — defaults to `/Users/vvasylkovskyi/git/IaC-Toolbox/git/`
- `feature-description` — what the feature does, from the conversation or user input
- `feature-context` — any prior discussion, constraints, or related features already established

## Step 1 — Understand the feature area

Read the user's description and any conversation history carefully. Identify the area of the codebase this feature touches.

Explore the repo to find relevant files:

```bash
cd <repos-base-path>/<repo-name>

# Get a picture of recent changes
git log --oneline -20

# Find files related to the feature area by keyword
grep -r "<relevant keyword>" --include="*.ts" --include="*.js" --include="*.sh" --include="*.yml" --include="*.yaml" -l

# Check existing feature docs for style and structure reference
ls .claude/context/roadmap/features/ 2>/dev/null | sort | tail -5

# Read the most relevant source files in full
```

Read enough to understand what already exists, what the feature needs to integrate with, and any constraints from the current implementation.

## Step 2 — Derive slug and filename

Create a short kebab-case slug from the feature description (e.g. `cloudflare-tunnel-full-support`).

Check existing feature files to pick the next sequential number:

```bash
ls .claude/context/roadmap/features/ 2>/dev/null | sort | tail -1
```

Name the file `<next-number>-<slug>.md`.

## Step 3 — Write the feature document

Write to `.claude/context/roadmap/features/<filename>`.

Follow the document style established in the iac-toolbox project:

```markdown
---
status: draft
completed_date:
pr_url:
---

# <Integration or Feature Name> — <Short Title>

## Overview

<2-3 sentences describing what this feature does, why it exists, and what it enables.
Written for a developer who has no prior context on this conversation.>

---

## What Changes

| Area   | Change         |
| ------ | -------------- |
| <area> | <what changes> |

---

## CLI Wizard Changes ← include only if the feature involves CLI/wizard

### <Dialog or Step Name>

<Terminal UI mockup using the established clack/inquirer style:>

\`\`\`sh
◆ <Prompt title>
│ <subtitle or instruction>
│
│ ◉ <option>
│ ◯ <option>
└
\`\`\`

<Explanation of behaviour, validation, defaults>

---

## Output — iac-toolbox.yml ← include if feature writes config

\`\`\`yaml
<relevant yaml block>
\`\`\`

---

## Output — ~/.iac-toolbox/credentials ← include if feature writes secrets

\`\`\`ini
[default]
<key> = <value>
\`\`\`

---

## Ansible / Script Changes ← include if playbooks or scripts change

<Before/after snippets, new tasks, removed tasks>

---

## CLI Commands ← include if new CLI subcommands are added

### iac-toolbox <command>

<Description, what it resolves to internally>

\`\`\`bash
iac-toolbox <command>
\`\`\`

---

## Post-Install Screen ← include if there is a post-install UX step

\`\`\`sh
◆ <integration> installed successfully
│ <key outputs>
└
\`\`\`

---

## Security Notes ← include if the feature handles credentials or sensitive data

- <note>

---

## Out of Scope

- <item deferred to future milestone>
- <item intentionally excluded>
```

## Step 4 — Apply document style rules

These rules reflect the conventions established across existing iac-toolbox feature docs:

- **Terminal UI mockups** use clack-style box drawing characters (`◆`, `◇`, `◉`, `◯`, `│`, `└`) consistently
- **YAML examples** always include inline comments for injected values (e.g. `# injected by CLI at deploy time`)
- **Before/after patterns** are used for any Ansible role changes — never describe changes in prose alone
- **Cloudflare domain integration** — if the feature exposes a service, always include the three-state dialog pattern (Cloudflare enabled + confirmed, Cloudflare enabled + declined, Cloudflare not enabled + greyed out)
- **Coming soon integrations** are shown as `○ <name> — coming soon` and non-interactive in multi-select mockups
- **Sections are only included when relevant** — omit CLI, Ansible, credentials sections if not applicable
- **Out of Scope is always present** — explicitly list what is deferred, even if the list is short

## Step 5 — Confirm to user

```
✅ Feature document created: .claude/context/roadmap/features/<filename>
Summary: <one line describing what the feature does>
Touches: <comma-separated list of key areas — CLI, Ansible, credentials, etc.>
```

## Rules

- Never implement the feature — only document it
- Always explore the repo before writing — do not write from conversation alone
- The document must be self-contained — a future agent needs nothing else to attempt an implementation
- Terminal UI mockups must follow the established clack style — do not invent new patterns
- Cloudflare domain dialog pattern must be included for any feature that exposes a network service
- Always include `status: draft` frontmatter — this is how the executor knows to pick it up
- If the repo or feature area is ambiguous, ask before exploring
- If the feature was discussed at length in the conversation, extract decisions made during that discussion rather than asking the user to repeat them
