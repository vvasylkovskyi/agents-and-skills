# Feature-Tasks-Sequential-With-Reviewer

**Inherits from:** [feature-tasks-sequential.md](feature-tasks-sequential.md)

All base workflow rules apply, with the addition of a two-agent write-review pattern for each task.

## Prerequisites

Two subagent files must be present in `.claude/agents/`:

- `feature-writer.md`
- `feature-reviewer.md`

## Feature Status Tracking

Each task file should have frontmatter for completion tracking:

```markdown
---
status: draft | in-progress | completed
completed_date: YYYY-MM-DD (only if completed)
pr_urls: []
---
```

**Before processing tasks:**
- Skip any task file where `status: completed`
- Update task file to `status: in-progress` when starting

**After all tasks completed successfully:**
- Update task file frontmatter:
  - `status: completed`
  - `completed_date: YYYY-MM-DD`
  - `pr_urls: [<all PR URLs>]`

## Enhanced Workflow

Replaces step 5 from base workflow:

5. **For each task, use write-review pattern:**

   **A. Status Check:**
   - Read task file frontmatter
   - If `status: completed` → skip to next task
   - If `status: draft` → update to `status: in-progress`

   **B. Writer Phase:**

   Invoke the `feature-writer` subagent with a prompt containing:
   - Path to the task file
   - Repository path: `/Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>`
   - Branch to create: `feature-<folder-name>-task-<number>`
   - Base branch: `<main or previous-task-branch>`

   Example prompt:

   ```
   Use the feature-writer agent.

   Task file: <task-file-path>
   Repository: /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
   Branch: feature-<folder-name>-task-<number>
   Base branch: <main or previous-task-branch>
   ```

   Expected output from writer: branch name + summary of changes made.

   **C. Reviewer Phase:**

   Extract the branch name from the writer's output, then invoke the `feature-reviewer` subagent with:
   - The branch name
   - The base branch name
   - Path to the task file (for acceptance criteria)
   - Repository path

   Example prompt:

   ```
   Use the feature-reviewer agent.

   Branch to review: <branch-name>
   Base branch: <base-branch>
   Task spec: <task-file-path>
   Repository: /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
   ```

   Expected output from reviewer: `PASS` or `FAIL - <list of specific issues>`.

   **D. Decision Phase:**
   - If reviewer returns **PASS**:
     - Run `gh pr create` to open the PR
     - **Update task file frontmatter:**
       - `status: completed`
       - `completed_date: 2026-04-20` (today's date)
       - `pr_urls: [<PR URL>]`
     - Mark task complete in task list
     - Notify user: `✅ Completed task <number>: <title>. PR: <url>`
     - Move to next task

   - If reviewer returns **FAIL**:
     - Invoke the `feature-writer` subagent again on the same branch with the reviewer's feedback included in the prompt:

       ```
       Use the feature-writer agent.

       Branch: <branch-name> (already exists — check it out, do not recreate)
       Repository: /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo-name>
       Task file: <task-file-path>

       Review feedback to address:
       <reviewer-output>
       ```

     - Return to Reviewer Phase (step B)
     - **Max 3 iterations** — if still failing after 3 attempts:
       - Report failure details to user
       - Ask whether to: skip task, continue with known issues, or abort

   **E. After PR Created:**
   - Log to memory: task number, PR URL, iterations needed
   - Update task list
   - Continue to next task

## Rules

- **Parent agent orchestrates** — invokes writer and reviewer subagents, parses their output, decides next action
- **Writer subagent** — implements and commits; never creates PRs
- **Reviewer subagent** — read-only; inspects diffs and task spec; never modifies files
- **Communication via output** — writer outputs branch name + summary; reviewer outputs PASS or FAIL
- **Retry cap enforced** — 3 write-review iterations maximum per task

## Memory Format

```markdown
# Feature: <folder-name>

Date: YYYY-MM-DD

## Task <number>: <title>

- Writer iterations: <count>
- Review result: PASS/FAIL
- PR: <url>
- Notes: <any issues encountered>
```

## Report Format

```
Feature folder: <folder-path>
Tasks found: <count>
Tasks skipped (already completed): <count>

Working sequentially with review:
⏭️  Task <n>: <title> → Already completed (PR: <url from frontmatter>)
✅ Task <n>: <title> → PR: <url> (reviewed, <iterations> iteration(s))
❌ Task <n>: <title> → Failed after <iterations> review cycles: <reason>
⚠️  Task <n>: <title> → PR: <url> (created with known issues, user approved)

Newly completed: <count> tasks
Failed: <count> tasks
Avg review iterations: <number>
```

## When to Use This vs Base Workflow

**Use this workflow when:**

- Tasks have many acceptance criteria
- High quality bar is required
- Context window can handle subagent spawning overhead

**Use base feature-tasks-sequential.md when:**

- Tasks are simple or straightforward
- Speed matters more than review depth
- Context window is constrained
