---
name: issue-writing
description: Write and create GitHub issues for missing work discovered during repo tasks. Use when you notice a gap, bug, missing documentation, follow-up task, UX rough edge, or implementation debt that should be captured as a GitHub issue instead of being left as an implicit note. Includes a concrete issue template, required issue quality bar, and exact `gh` commands for creating and verifying issues.
---

# Issue Writing

Use this skill when a repo task reveals something that should be captured as a GitHub issue.

This skill is for durable follow-up work.

Use it to:
- record missing pieces found during implementation
- capture bugs discovered during testing
- note UX gaps, rough edges, or missing docs
- create follow-up tasks that are real enough to be actionable
- turn local notes into a GitHub issue Viktor can review and prioritize

Do not use this skill for:
- vague brainstorming with no actionable next step
- ephemeral observations that belong only in a task summary
- duplicate issues when the same problem is already tracked

## Core rule

If you discover a real missing piece during work, do not leave it buried in chat or in your own summary.

Create an issue when the missing piece is:
- actionable
- scoped enough to describe clearly
- valuable to track independently

## Issue quality bar

A good issue should let someone understand:
- what is wrong or missing
- why it matters
- what good looks like
- what constraints or context matter
- how to verify it is done

Write issues so they are useful later, not just accurate right now.

## Before creating an issue

Check these first:
- Is this already tracked?
- Is this actually a separate piece of work?
- Is the scope narrow enough to be actionable?
- Do I have enough context to write a useful title and body?

If the answer is no, gather context first.

## Required inputs

Determine:
- repo owner/name
- issue title
- concise problem statement
- why it matters
- proposed outcome or desired behavior
- acceptance criteria if they are clear
- labels if useful
- roadmap metadata location if the issue comes from `context/roadmap/`
- ticket metadata entry if the issue is part of a multi-ticket roadmap milestone

## Roadmap metadata workflow

When issues are derived from `context/roadmap/`, prefer a sidecar JSON file next to the markdown roadmap item.

Recommended shape:
- `milestone-name.md` — human-readable roadmap description
- `milestone-name.metadata.json` — structured ticket tracking metadata

The JSON may contain multiple ticket entries.

Each ticket entry should carry at least:
- `ticket_id`
- `title`
- `summary`
- `repo_url`
- `issue_number`
- `issue_url`
- `status`
- `source_sections`
- `acceptance_hints`

Interpretation rule:
- if `issue_number` is missing or null, the issue has not been created yet
- if `repo_url` is missing or null, the issue is not ready to be created yet

After creating an issue from roadmap metadata, update the JSON entry in place.

## AI responsibility for roadmap extraction

Do not leave roadmap decomposition to a naive parser.

When working from roadmap markdown, the LLM should:
1. read the roadmap markdown
2. read the relevant product context from `context/`
3. identify meaningful ticket candidates
4. infer the most appropriate repo from project architecture and feature description
5. collect the structured fields for each ticket
6. edit the sidecar metadata JSON directly
7. only then create GitHub issues from the resulting metadata

## Required ticket fields for roadmap extraction

For each ticket candidate, collect:
- `ticket_id` — stable slug for the ticket
- `title` — concise GitHub issue title
- `summary` — short but useful issue summary
- `repo_url` — target repo where work belongs
- `status` — usually `planned` before issue creation
- `source_sections` — headings or sections in the roadmap markdown that justify the ticket
- `acceptance_hints` — short checklist-like hints to seed issue acceptance criteria

## Metadata schema

When creating or updating `*.metadata.json`, use this top-level shape:

```json
{
  "roadmap_item": {
    "id": "milestone-slug",
    "title": "Human-readable title",
    "source_markdown": "milestone-file.md",
    "project_url": "https://github.com/orgs/IaC-Toolbox/projects/1",
    "status": "draft",
    "notes": "Each ticket entry represents a candidate issue derived from this roadmap milestone. Missing issue_number or repo_url means the ticket has not been created yet."
  },
  "tickets": [
    {
      "ticket_id": "stable-ticket-slug",
      "title": "Ticket title",
      "summary": "Short summary",
      "repo_url": "https://github.com/IaC-Toolbox/iac-toolbox-cli",
      "issue_number": null,
      "issue_url": null,
      "project_item_id": null,
      "status": "planned",
      "source_sections": ["Section heading"],
      "acceptance_hints": ["Hint 1", "Hint 2"]
    }
  ]
}
```

Interpretation rule:
- if `issue_number` is null or missing, the issue has not been created yet
- if `repo_url` is null or missing, repo assignment is still unresolved
- preserve existing `issue_number`, `issue_url`, and `project_item_id` values unless there is a clear reason to change them

## Repo inference guidance

Infer repo from project architecture, not guesswork.

Typical routing:
- CLI / wizard / terminal UX / npm / brew / config generation → `https://github.com/IaC-Toolbox/iac-toolbox-cli`
- Raspberry Pi / Ansible / playbooks / ARM64 / homelab services → `https://github.com/IaC-Toolbox/iac-toolbox-raspberrypi`
- docs / tutorials / guides / setup documentation → `https://github.com/IaC-Toolbox/iac-toolbox-project-docs`
- AWS / Terraform / cloud infrastructure → `https://github.com/IaC-Toolbox/iac-toolbox-project`

If repo assignment is genuinely unclear, leave `repo_url` null and report the ambiguity instead of inventing one.

## Title rules

Prefer short, concrete titles.

Good titles:
- `Wizard should validate missing environment variables before apply`
- `Document local Vault bootstrap and unseal recovery flow`
- `CLI should surface actionable error when backend config is missing`

Avoid titles like:
- `Fix stuff`
- `Improve UX`
- `Notes from testing`

## Recommended issue structure

Use this structure by default.

```markdown
## Summary
<one short paragraph describing the gap, bug, or missing piece>

## Problem
- <what happens today>
- <why this is incomplete, confusing, broken, or risky>

## Desired outcome
- <what should happen instead>
- <what behavior or artifact should exist>

## Context
- <where this was found>
- <relevant repo paths, commands, logs, or workflow context>
- <links to related PRs, issues, or commits if useful>

## Acceptance criteria
- [ ] <clear outcome 1>
- [ ] <clear outcome 2>
- [ ] <clear outcome 3>

## Notes
- <constraints, non-goals, implementation hints, or follow-up context>
```

## Template adaptation rule

Use the default structure above unless the issue is clearly one of these:
- bug report
- documentation gap
- technical debt / refactor follow-up
- feature follow-up discovered during implementation

Adapt wording, but keep the same information density.

## Bug-oriented variant

When the issue is a bug, prefer this body shape:

```markdown
## Summary
<short description of the bug>

## Current behavior
- <what happens>

## Expected behavior
- <what should happen>

## Impact
- <why this matters>

## Reproduction
1. <step 1>
2. <step 2>
3. <step 3>

## Acceptance criteria
- [ ] bug no longer reproduces
- [ ] expected behavior is implemented
- [ ] relevant validation or test coverage exists

## Notes
- <logs, environment details, constraints, or hypotheses>
```

## Documentation-gap variant

When the issue is missing documentation, prefer this body shape:

```markdown
## Summary
<what documentation is missing>

## Problem
- <what a user/operator cannot currently understand or do>
- <where they get stuck>

## Desired documentation
- <what document or section should exist>
- <what it should explain>

## Acceptance criteria
- [ ] the missing documentation exists in the repo
- [ ] the steps or concepts are accurate
- [ ] the doc is linked from the right place if needed

## Notes
- <commands, paths, screenshots, or related docs>
```

## Creation flow

Use `gh` CLI.

Prefer this sequence:
1. check for likely duplicates
2. draft the issue body in a temporary markdown file
3. create the issue with `gh issue create`
4. verify the created issue
5. report the issue link back in the task summary

## Duplicate check

Search before creating.

```bash
gh issue list --repo <owner>/<repo> --state all --search "<keywords>"
```

If useful, inspect matching issues:

```bash
gh issue view <number> --repo <owner>/<repo>
```

Do not create a duplicate if an open issue already covers the same work.

## Draft body pattern

Write the issue body to a temporary file first.

```bash
cat > /tmp/issue-body.md <<'EOF'
## Summary
...

## Problem
- ...

## Desired outcome
- ...

## Context
- ...

## Acceptance criteria
- [ ] ...
- [ ] ...

## Notes
- ...
EOF
```

## Create issue — use this exactly

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "<issue-title>" \
  --body-file /tmp/issue-body.md
```

With labels:

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "<issue-title>" \
  --body-file /tmp/issue-body.md \
  --label "<label-1>" \
  --label "<label-2>"
```

## Verify created issue

After creation, verify the final issue:

```bash
gh issue view <number> --repo <owner>/<repo>
```

If `gh issue create` returns a URL, use that to identify the created issue and inspect it.

## Reporting back

When you create an issue as part of a larger task, report:
- issue number and link
- short title
- why it was created
- whether it is blocking or follow-up work

## Writing style

- Prefer concrete language over abstract language
- Keep paragraphs short
- Use bullets where they reduce ambiguity
- Avoid filler and management-speak
- Do not over-specify implementation unless that detail matters
- Focus on the user-visible gap or operational problem

## Good issue checklist

Before creating, check:
- Is the title specific?
- Is the problem understandable without chat history?
- Does the issue explain why it matters?
- Are the acceptance criteria testable?
- Is there enough context to pick it up later?

## Failure reporting

If issue creation fails, report the exact layer:
- `gh` auth not configured
- repo not found
- duplicate issue already exists
- label does not exist
- network/API failure
- insufficient issue context to write a good issue

Be specific.

## Definition of done

This skill is done when:
- the issue is not a duplicate
- the title is clear
- the body is actionable
- the issue exists on GitHub
- the link is included in the task summary or user update
