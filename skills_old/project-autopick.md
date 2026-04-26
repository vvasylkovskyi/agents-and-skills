# Project-Autopick

Inspect [IaC-Toolbox Project #1](https://github.com/orgs/IaC-Toolbox/projects/1) and autonomously pick ready issues.

**Skills:** `project-issue-pickup`  
**Local repos:** `/Users/vvasylkovskyi/git/IaC-Toolbox/`

## Dedup guard (check before spawn)

- No open PR for issue (check with `gh pr list --repo <owner/repo> --search "closes #<issue>"`)
- Not assigned to anyone else
- Issue is in "Ready" or "Todo" status on project board

Skip issue if any check fails. Log reason.

## Workflow

1. Sync workspace: `git pull origin main`
2. Query Project #1 → classify: actionable/blocked/in-progress/stale
3. Run dedup guard on actionable items
4. Select up to 20 passing issues (hard cap)
5. For each issue, spawn Agent in background:
   - Extract repo name from issue
   - Determine repo path: `/Users/vvasylkovskyi/git/IaC-Toolbox/<repo-name>`
   - Create branch: `issue-<number>-<slug>`
   - Spawn Agent with detailed prompt including:
     - Repo path and issue URL
     - **Critical implementation guidelines** (challenge specs, don't follow blindly)
     - **Validation requirements** from context/development/
     - **Testing checklist** before PR creation
6. Write memory: `memory/project-autopick/YYYY-MM-DD-autopick.md`
7. Commit + push workspace
8. Report results

**Rules:**

- Use `gh api` or `gh project item-list` to query project board
- Hard cap: at most 20 new issue pickups per heartbeat run
- Don't spawn duplicates
- Single spawn failure = continue with remaining
- Board unreachable = report + `HEARTBEAT_OK`
- Use Agent tool with `run_in_background: true` for spawning
- Use `isolation: "worktree"` when spawning agents to work in isolated git worktrees
- **Agent prompt must include**: critical thinking directive, validation requirements, testing checklist

## Example Agent Spawn

```
Agent({
  description: "Implement issue #35",
  prompt: `Implement issue #35: Download Dialog with Loading Animation.

Repository: /Users/vvasylkovskyi/git/IaC-Toolbox/iac-toolbox-cli
Issue: https://github.com/IaC-Toolbox/iac-toolbox-cli/issues/35

## Critical Implementation Guidelines

1. **Read and understand** the issue thoroughly
2. **Challenge the specification**: If code examples or patterns in the issue seem suboptimal, improve them
3. **Don't follow blindly**: Use your judgment - the issue is a guide, not gospel
4. **Verify against context**: Check context/architecture/, context/tech-stack/, and context/development/ for project conventions
5. **Follow the workflow**: See context/development/workflow.md for PR requirements
6. **Complete validation**: Run all tests from context/development/testing-checklist.md before creating PR

## Required Before PR

For CLI repository:
- Run npm test && npm run lint && npm run build
- Test wizard flow manually
- Capture screenshots if UI changed

For Raspberry Pi repository:
- Run ansible-lint and shellcheck
- Test with RPI_HOST=local
- Verify Cloudflare URLs respond

## Deliverable

Create PR with:
- Clear summary of changes
- Testing methodology and results
- Screenshots (if UI)
- Closes #35

Think critically. If the issue suggests an approach that conflicts with project architecture or best practices, deviate and explain why in the PR.`,
  run_in_background: true,
  isolation: "worktree"
})
```

## Report format

```
Project scan: IaC-Toolbox #1
Actionable: #<n> <title> — <why>
Blocked: #<n> <title> — <blocker>
In progress: #<n> <title> — <task/PR>
Spawned: #<n> <title> → agent spawned in background
Skipped: #<n> <title> — <dedup reason>
Failed: #<n> <title> — <error>
Next: <title or "none">
```

Append `HEARTBEAT_OK` if nothing spawned and no actionable issues.
