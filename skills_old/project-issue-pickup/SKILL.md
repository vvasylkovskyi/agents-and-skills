---
name: project-issue-pickup
description: Scan a GitHub Project board, classify issues as actionable/blocked/in-progress/stale, and autonomously work on ready issues. Use during heartbeats or periodic project triage. Works in parallel (spawn multiple agents) or sequential (work through issues one-by-one) mode.
---

# Project Issue Pickup

Use this skill to inspect a GitHub Project board and autonomously work on ready issues.

This skill is for project-board-driven triage and autonomous pickup with two modes:
- **Parallel mode**: Spawn multiple background agents (up to spawn cap)
- **Sequential mode**: Work through issues one-by-one in isolated worktrees

## Configuration

The heartbeat prompt that invokes this skill should specify:
- Project URL (e.g., `https://github.com/orgs/<org>/projects/<N>`)
- Repository base path (e.g., `/Users/username/git/<project>/`)
- Spawn cap (max issues per run)
- Execution mode (parallel or sequential)

## Core rule

Project scanning is not implementation.

This skill decides whether a project item is safe and worthwhile to start.
If a non-trivial coding task is selected, hand it off through the established
coding workflow.

## Execution context

This skill is typically used from an autonomous heartbeat.

In that context:
- Do not wait for user approval — work autonomously
- If an issue is ambiguous, skip it and record why; do not stall
- Notify user after each PR creation (sequential mode) or at end (parallel mode)

If used interactively, the user may request interactive approval for each issue.

## Spawn cap

The spawn cap is configured in the heartbeat prompt.
Typical values: 2-20 issues per run depending on project needs.

## Dedup guard

Before working on any issue, check all of the following:
- No open PR already covers this issue scope (use `gh pr list --search "closes #<issue>"`)
- Issue is not assigned to anyone else
- Issue is in "Ready" or "Todo" status on project board

If any check fails, skip the issue and log the reason in the report.

## Classifications

Apply these to every project item:

- **actionable** — clearly scoped, no blocker, no active work, ready to spawn
- **blocked** — depends on another issue, PR, or Viktor decision not yet made
- **in progress** — active PR or active Codex/OpenClaw task already exists
- **stale** — no recent activity, unclear if still relevant, needs Viktor review

## Project scan goals

A good scan answers:
- which items are ready to start?
- which items are blocked and why?
- which items are already in progress?
- which items need Viktor's attention before they can move?
- what is the single best next candidate if only one could be picked?

## Issue pickup policy

Before selecting an issue for pickup, confirm:
- The item maps to a real GitHub issue
- The issue has enough detail to act on without guessing
- The issue is not blocked by another issue, PR, or unresolved decision
- The issue passes the full dedup guard above
- The issue belongs to a repository in the configured base path

## Candidate heuristics

Good pickup candidates:
- Clearly worded title and body
- Scope is bounded enough for a single PR
- No active PR already covering the work
- No obvious blocker or dependency
- Repository ownership is unambiguous
- The next implementation step is concrete without architectural guesswork

Skip and flag as needs-clarification when:
- The issue title is vague or the body is missing
- The correct repository is unclear
- The issue depends on a design decision not yet made
- The scope would require multiple PRs to complete

## Scan sequence

1. Sync workspace: `git pull origin main` in all repositories
2. Query project items via `gh api` or `gh project item-list`
3. Identify items that map to real GitHub issues
4. Inspect linked issues and relevant repositories
5. Inspect open PRs for dedup
6. Classify all candidates (actionable/blocked/in-progress/stale)
7. Apply dedup guard to actionable candidates
8. Select up to spawn cap that pass all checks

**Parallel mode (spawn background agents):**
9. Spawn Agent for each using `run_in_background: true` and `isolation: "worktree"`
10. If one spawn fails, continue with remaining candidates
11. Write memory file and report results

**Sequential mode (work through one-by-one):**
9. For each issue:
   - Enter worktree with new branch
   - Implement issue following guidelines
   - Run validation/tests
   - Create PR
   - Exit worktree
   - Notify user
10. Write memory file and report results

## Query guidance

Use `gh project item-list` or `gh api` for project board queries.
If queries fail, report the exact failure and exit — do not pretend the board was scanned.

## Implementation guidelines

Each issue must be implemented with:
1. **Read and understand** the issue thoroughly
2. **Challenge the specification**: If code examples seem suboptimal, improve them
3. **Don't follow blindly**: Use judgment - the issue is a guide, not gospel
4. **Verify against project context**: Check architecture and conventions
5. **Complete validation**: Run all tests before creating PR

## Parallel mode agent spawn

Use the Agent tool with appropriate settings:

```
Agent({
  description: "Implement issue #<number>",
  prompt: "Implement issue #<number>: <title>. Repository: <repo-path>. Read the issue and implement the requested changes. Create a PR when complete.",
  run_in_background: true,
  isolation: "worktree"
})
```

## Sequential mode worktree pattern

Use EnterWorktree/ExitWorktree for each issue:

```
EnterWorktree({
  branch: "issue-<number>-<slug>",
  base_branch: "<main or previous-issue-branch>"
})

// Implement issue, run tests, create PR

ExitWorktree()
```

## Command-grounded failure rule

If pickup reaches worker handoff, the result must be grounded in a real command outcome.

That means:
- if spawn succeeds, report the spawned task id / branch / session
- if spawn fails, report the exact failing command and the literal stderr/stdout reason
- a failed spawn for one issue must not abort the whole run; continue with the next eligible issue unless the failure proves a global blocker affecting all remaining candidates
- do not speculate about repo-path, worktree, or infrastructure bugs unless a command result proves that exact failure

## Reporting format

Always emit a report, even when nothing is worked on.

**Parallel mode:**
```text
Project scan: <project-name>

Actionable: #<n> <title> — <why ready>
Blocked: #<n> <title> — <blocker>
In progress: #<n> <title> — <active PR>
Stale: #<n> <title> — <reason>
Needs clarification: #<n> <title> — <what is unclear>

Spawned: #<n> <title> → agent spawned in background
Skipped: #<n> <title> — <dedup reason>
Failed: #<n> <title> — <error>

Next: <title or "none">
```

**Sequential mode:**
```text
Project scan: <project-name>

Actionable: #<n> <title> — <why>
Blocked: #<n> <title> — <blocker>
In progress: #<n> <title> — <active PR>

Working sequentially:
✅ #<n> <title> → PR created: <url>
❌ #<n> <title> → Failed: <error>
⏭️  #<n> <title> → Skipped: <dedup reason>

Completed: <count> issues
Failed: <count> issues
Remaining: <title or "none">
```

Append `HEARTBEAT_OK` if nothing worked and no actionable issues exist.

## Failure reporting

Report the exact failure layer:
- Project query unavailable (`gh` auth, API endpoint)
- Repository not found
- Issue query failed
- PR query failed
- Dedup guard prevented work (which check failed)
- Implementation failed (tests, build, validation)
- PR creation failed

## Definition of done

This skill is complete when:
- The project board was inspected, or the exact query failure is reported
- Every item is classified
- The dedup guard ran on all actionable candidates
- Up to spawn cap issues were selected and worked on
- Each issue was implemented or spawn was attempted (parallel mode)
- A single issue failure did not abort the rest of the run
- A memory file was written
- The report was emitted