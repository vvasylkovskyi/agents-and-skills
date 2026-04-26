# Project-Autopick-Sequential

Inspect [IaC-Toolbox Project #1](https://github.com/orgs/IaC-Toolbox/projects/1) and autonomously work through ready issues one by one.

**Local repos:** `/Users/vvasylkovskyi/git/IaC-Toolbox/git/`

## Dedup guard (check before starting each issue)

- No open PR for issue (check with `gh pr list --repo <owner/repo> --search "closes #<issue>"`)
- Not assigned to anyone else
- Issue is in "Ready" or "Todo" status on project board

Skip issue if any check fails. Log reason and move to next.

## Workflow

**IMPORTANT: Work as a single agent with task list. Do NOT spawn sub-agents.**

1. Sync workspace: `git pull origin main` in all repos
2. Query Project #1 → classify: actionable/blocked/in-progress/stale
3. Run dedup guard on actionable items
4. Select up to 20 passing issues (ordered by priority)
5. **Create task list using TodoWrite** with all issues to be worked
6. **For each issue, work sequentially in this agent:**
   - Extract repo name from issue
   - Determine repo path: `/Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>`
   - Create branch: `issue-<number>-<slug>`
   - **Determine base branch:**
     - If this issue depends on changes from previous issue → use previous issue's branch as base
     - Otherwise → use `main`
   - Implement the issue following guidelines below
   - Run validation
   - Create PR with `gh pr create`
   - **Mark task complete in task list**
   - **Notify user in chat:** "✅ Completed issue #<number>: <title>. PR: <url>"
   - Move to next issue
7. Write memory: `memory/project-autopick/YYYY-MM-DD-autopick-sequential.md`
8. Report final results

**Rules:**

- **Work as single agent** - Do NOT use Agent tool or spawn sub-agents
- **Use TodoWrite** to create and track task list for all issues
- **Mark each task complete** as you finish each issue
- Use `gh api` or `gh project item-list` to query project board
- Hard cap: at most 20 issues per run
- Single issue failure = report error, continue with remaining issues
- Board unreachable = report + `HEARTBEAT_OK`
- Work directly on repo branches (no worktree isolation needed)
- Always create PR before moving to next issue
- Always notify user after each PR creation

## Implementation Guidelines for Each Issue

When working on each issue:

1. **Read and understand** the issue thoroughly
2. **Challenge the specification**: If code examples or patterns in the issue seem suboptimal, improve them
3. **Don't follow blindly**: Use your judgment - the issue is a guide, not gospel
4. **Verify against context**: Check context/architecture/, context/tech-stack/, and context/development/ for project conventions
5. **Follow the workflow**: See context/development/workflow.md for PR requirements
6. **Complete validation**: Run all tests from context/development/testing-checklist.md before creating PR

## Validation Requirements (run before creating PR)

**For CLI repository:**

- Run `npm run lint && npm run build && npm run format`
- Test wizard flow manually if UI changed
- Capture screenshots if UI changed

**For Raspberry Pi repository:**

- Run `ansible-lint` and `shellcheck`
- Test with `RPI_HOST=local`
- Verify Cloudflare URLs respond

## PR Creation

Create PR with:

- Clear summary of changes
- Testing methodology and results
- Screenshots (if UI)
- `Closes #<issue-number>`

Think critically. If the issue suggests an approach that conflicts with project architecture or best practices, deviate and explain why in the PR.

## Base Branch Selection Logic

```
previous_branch = null

for each issue:
  if issue depends on previous work:
    base_branch = previous_branch
  else:
    base_branch = "main"

  create branch from base_branch
  work on issue
  create PR with base = base_branch

  previous_branch = current_branch
```

## Report format

```
Project scan: IaC-Toolbox #1
Actionable: #<n> <title> — <why>
Blocked: #<n> <title> — <blocker>
In progress: #<n> <title> — <task/PR>

Working sequentially:
✅ #<n> <title> → PR created: <url>
❌ #<n> <title> → Failed: <error>
⏭️  #<n> <title> → Skipped: <dedup reason>

Completed: <count> issues
Failed: <count> issues
Remaining: <title or "none">
```

Append `HEARTBEAT_OK` if nothing worked and no actionable issues.
