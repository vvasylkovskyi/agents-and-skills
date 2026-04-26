# Feature-Tasks-Sequential

Work through numbered task files in a feature folder sequentially, implementing each one.

**Usage:** Invoke with feature folder path as parameter.

**Local repos:** `/Users/vvasylkovskyi/git/IaC-Toolbox/git/`

**⚠️ Context Window Requirements:** This workflow assumes a very large context window for the agent. Each task builds upon the previous one, so all code changes, implementation details, and decisions from prior tasks must remain in context. Not suitable for models with limited context or for feature sets with many large tasks.

## Input

Feature folder containing numbered task markdown files:
- `01-task-name.md`
- `02-another-task.md`
- `03-final-task.md`

Each task file should specify:
- Task description
- Target repository
- Acceptance criteria
- Any dependencies

## Workflow

**IMPORTANT: Work as a single agent with task list. Do NOT spawn sub-agents.**

1. Sync workspace: `git pull origin main` in all repos
2. Read all task files from feature folder (sorted numerically)
3. Validate each task file has required information
4. **Create task list using TodoWrite** with all tasks to be worked
5. **For each task, work sequentially in this agent:**
   - Extract repo name from task file
   - Determine repo path: `/Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>`
   - Create branch: `feature-<folder-name>-task-<number>`
   - **Determine base branch:**
     - If task depends on previous task → use previous task's branch as base
     - Otherwise → use `main`
   - Implement the task following guidelines below
   - Run validation
   - Create PR with `gh pr create`
   - **Mark task complete in task list**
   - **Notify user in chat:** "✅ Completed task <number>: <title>. PR: <url>"
   - Move to next task
6. Write memory: `memory/feature-tasks/YYYY-MM-DD-<folder-name>.md`
7. Report final results

**Rules:**

- **Work as single agent** - Do NOT use Agent tool or spawn sub-agents
- **Use TodoWrite** to create and track task list for all tasks
- **Mark each task complete** as you finish each task
- Process tasks in numerical order (01, 02, 03, etc.)
- Single task failure = report error, continue with remaining tasks
- Work directly on repo branches (no worktree isolation needed)
- Always create PR before moving to next task
- Always notify user after each PR creation

## Implementation Guidelines for Each Task

When working on each task:

1. **Read and understand** the task file thoroughly
2. **Challenge the specification**: If code examples or patterns in the task seem suboptimal, improve them
3. **Don't follow blindly**: Use your judgment - the task is a guide, not gospel
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
- Reference to task file: `Implements <folder-name>/task-<number>`
- Testing methodology and results
- Screenshots (if UI)

Think critically. If the task suggests an approach that conflicts with project architecture or best practices, deviate and explain why in the PR.

## Base Branch Selection Logic

```
previous_branch = null

for each task:
  if task depends on previous work:
    base_branch = previous_branch
  else:
    base_branch = "main"

  create branch from base_branch
  work on task
  create PR with base = base_branch

  previous_branch = current_branch
```

## Report format

```
Feature folder: <folder-path>
Tasks found: <count>

Working sequentially:
✅ Task <n>: <title> → PR created: <url>
❌ Task <n>: <title> → Failed: <error>

Completed: <count> tasks
Failed: <count> tasks
Remaining: <title or "none">
```
