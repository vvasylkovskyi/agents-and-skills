---

name: pr_refresh_and_conflict_resolution
description: Use when an open PR branch is behind its base branch, needs a rebase, or hits merge conflicts. Handles branch refresh, conflict-resolution task routing, verification, and merge-readiness reporting.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# PR Refresh and Conflict Resolution

Use this skill when:

* an open PR branch is behind its base branch
* a rebase is needed before merge
* a rebase hits conflicts
* a previously green PR may be stale after another merge

## Routing

Use adjacent skills for their own concerns:

* `codex-orchestrated-dev` — top-level workflow and PR handoff
* `multi_agent_coordination` — worker spawn, tmux/mailbox, monitoring

If this skill needs a Codex worker, spawn it through `multi_agent_coordination`.

## Core policy

A PR is merge-ready only when:

* the branch is current against the latest base branch
* validation passed on the current branch state
* no unresolved conflicts remain

Never recommend merge based on old green checks after the base branch changed.

## Required inputs

Determine:

* repo path
* repo name
* branch name
* base branch, default `main`
* PR number if available
* validation commands
* Discord channel ID if a worker must be spawned

If the channel ID is required and not known, ask Viktor before spawning.

## Default refresh flow

Use rebase by default:

```bash
git fetch origin
git checkout <branch>
git rebase origin/<base-branch>
```

After a successful rebase:

```bash
# run validation commands
git push --force-with-lease
```

Use `--force-with-lease`, not plain `--force`.

## Outcomes

### Already current

* do not rewrite history unnecessarily
* verify validation is still valid for the current branch state
* report status clearly

### Rebase succeeds

* run validation
* if validation passes, push with `--force-with-lease`
* report refreshed status and validation result

### Rebase conflicts

* do not recommend merge
* do not continue broad feature work
* create or continue one dedicated conflict-resolution task
* report the blocker clearly

## Conflict-resolution task rules

Conflict resolution is its own task type.

Rules:

* use a narrow-scope task
* preserve intended PR behavior
* validate after resolving conflicts
* push only after validation, or report the blocker clearly

Only one active conflict task may exist per PR branch.
Do not spawn duplicates on repeated heartbeat checks.

Recommended task IDs:

* `rebase-<original-task-id>`
* `resolve-conflict-<pr-number>`
* `refresh-<branch-name>`

## Conflict-resolution prompt contract

Include only:

* repo path
* repo name
* branch name
* base branch
* PR number if available
* intended behavior to preserve
* validation commands
* definition of done

Recommended shape:

```text
Repo: /Users/vvasylkovskyi/git/IaC-Toolbox/<repo-name>
Branch: <branch-name>
Base: origin/<base-branch>
PR: #<number>

Task:
Rebase this branch onto the latest base branch and resolve conflicts carefully.

Requirements:
- Preserve the intended behavior already implemented in this PR
- Resolve conflicts intentionally against the latest base branch
- Run validation after resolving conflicts
- Push with --force-with-lease once ready
- Summarize conflicting files, resolutions, validation result, and final commit hash

Definition of done:
- branch rebased onto latest base branch
- conflicts resolved intentionally
- validation passed, or blocker clearly explained
- remote branch updated
- short summary produced
```

## Detection guidance

Use repo state, not guesswork.

Example:

```bash
git fetch origin
git checkout <branch>
git rev-list --left-right --count <branch>...origin/<base-branch>
```

Use PR mergeability/conflict signals too when available.

## Verification

Before declaring success:

* verify the branch is based on the latest base branch
* verify validation actually ran
* verify the push succeeded
* verify the PR is no longer stale or conflicted

Do not trust worker status alone.

## Failure reporting

Report the exact failed layer, for example:

* base branch unknown
* git fetch failed
* checkout failed
* rebase conflicted
* validation failed
* push failed
* duplicate conflict task already active
* missing channel ID prevented spawn

## Reporting

Report only meaningful changes:

* branch already current
* rebase succeeded
* validation passed after rebase
* conflicts detected
* conflict-resolution task spawned
* branch refreshed and merge-ready
* blocked, with exact reason

## Done criteria

Success:

* branch refreshed against latest base branch
* validation passed on refreshed state
* remote branch updated
* PR is conflict-free and merge-ready

Blocked:

* refresh or conflict resolution could not be completed
* blocker is clearly identified
* duplicate work was avoided
* next action is explicit
