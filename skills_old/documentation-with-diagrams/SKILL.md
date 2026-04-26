---
name: documentation-with-diagrams
description: Supporting skill for writing docs-first plans, implementation proposals, architecture notes, and PR descriptions for repo work. Use when the top-level coding workflow has already decided that documentation is needed. Prefer concise practical prose and include a high-level diagram when system structure, data flow, boundaries, or deployment flow would benefit from visualization. Prefer Mermaid in markdown so GitHub renders it well.
---

# Documentation With Diagrams

Use this as a supporting skill for repo documentation.

## Scope

This skill owns documentation quality.

Use it for:
- docs-first plans
- architecture explanations
- PR descriptions
- implementation proposals
- operational runbooks

Do not treat this as the top-level coding workflow skill.

## Core style

- Keep documentation practical.
- Prefer short sections over long walls of text.
- Write for decision-making, implementation, and review.
- Optimize for GitHub PR readability.
- Avoid unnecessary filler.

## When to include a diagram

Include a high-level diagram when any of these are true:
- multiple components interact
- there is an architecture or deployment flow to explain
- request/response or data flow matters
- there are important boundaries between systems
- the written explanation is getting too text-heavy

Skip diagrams for very small or obvious changes.

## Diagram format preference

Prefer this order:
1. Mermaid markdown diagrams
2. Plain text diagrams

Use Mermaid when possible because GitHub PRs render it well.

## Diagram rules

- Keep diagrams high-level.
- Do not over-model every detail.
- Use clear labels.
- Match diagram names to the wording used in the document.
- Make sure the diagram supports the text instead of replacing it.
- After the diagram, explain the key takeaway in 1-3 short bullets if useful.

## GitHub PR rule

Use both places on purpose:
- write the durable technical document into a file in the repo for future reference
- put the actual proposal / review-oriented summary into the GitHub PR description

Reason:
- repo doc files preserve technical decisions long-term
- GitHub PR descriptions render Mermaid well and are easier for Viktor to review than changed-file diffs

## Reliable PR body update pattern

When updating a PR description, prefer a reliable API-based path instead of depending only on `gh pr edit`.

Prefer this pattern:
1. write the intended PR body to a temporary markdown file
2. use `gh api` with safe escaping to PATCH the PR body
3. verify the updated PR body afterwards

## Documentation quality bar

Before finishing, check:
- is the document easy to skim?
- does it explain the system at the right level?
- would a diagram reduce ambiguity?
- is the diagram simple enough to read in a PR?
- does the doc help Viktor decide go / no-go quickly?
