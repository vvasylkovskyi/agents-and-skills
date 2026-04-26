---
name: roadmap-to-tasks
description: Split a roadmap markdown file into individual task files (~200-500 tokens each) with metadata index. Use when a new milestone roadmap needs to be decomposed into trackable tasks for the heartbeat workflow.
agent_mode: plan_first
---

# Roadmap to Tasks

Use this skill to split a monolithic roadmap into discrete task files.

## Execution Mode

**Required:** Use structured planning before execution.

1. **Create a plan** with clear task breakdown:
   ```
   ⎿  ◼ Load context and understand project structure
      ◼ Read architecture docs
      ◼ Read tech stack docs
      ◼ Read testing requirements
   ⎿  ◼ Analyze roadmap and identify logical task boundaries
      ◼ Parse milestone markdown
      ◼ Identify feature sections
      ◼ Group related sections
   ⎿  ◼ Create enriched task files with full context
      ◼ Write task-001-*.md
      ◼ Write task-002-*.md
      ...
   ⎿  ◼ Update task-index.json with new tasks
   ⎿  ◼ Create milestone metadata file
   ```

2. **Execute incrementally**: Complete each sub-task before moving to next
3. **Track progress**: Mark tasks complete (✔) as you finish them
4. **Verify**: Ensure all tasks complete before reporting done

## When to use

- New roadmap markdown file needs decomposition
- Roadmap has been updated with new sections that need task extraction
- Consolidating related sections from roadmap into logical tasks

**Important:** This skill creates **context-rich** task files by pulling from:
- Roadmap content (what to build)
- Architecture docs (how components fit)
- Tech stack docs (technologies used)
- Constraints (limitations/requirements)
- Related files (existing code)
- Testing requirements (validation strategy)

The goal is to create tasks that agents can pick up and implement autonomously without needing to ask clarifying questions.

## Input requirements

You need:
- Path to roadmap file: `context/roadmap/milestone-*.md`
- Milestone ID (e.g., `milestone-0-raspberrypi-cli`)
- Target repo (e.g., `iac-toolbox-cli`)

## Output structure

Creates:
- `context/tasks/task-NNN-short-slug.md` — one per logical feature (~200-500 tokens)
- `context/metadata/task-index.json` — lightweight index
- `context/metadata/milestone-*.metadata.json` — detailed metadata

## Task extraction rules

One task per logical unit of work:
- Each task = one GitHub issue
- Scope: completable in 1-3 PRs
- Independent where possible
- Clear acceptance criteria
- **Rich context**: Include architecture, tech stack, constraints, related files
- **Actionable**: Agent can implement without asking for clarification
- **Complete**: Testing strategy and validation steps included

## Task file template

```markdown
# Task NNN: Title

**Milestone:** milestone-id  
**Status:** See metadata file  
**Repo:** https://github.com/IaC-Toolbox/repo-name  
**Tech Stack:** Node.js/TypeScript/Ink (or Ansible/Bash/Terraform)

## Description

<extracted from roadmap, 1-3 paragraphs>

## Context

### Project Architecture
<relevant details from context/architecture/>
- Repository boundaries and responsibilities
- Related components/services
- Integration points

### Technical Stack
<relevant details from context/tech-stack/>
- Technologies used
- Key libraries/frameworks
- Architecture patterns

### Constraints
<relevant details from context/constraints/>
- Platform requirements (ARM64/x64)
- Performance considerations
- Security requirements

### Related Files
<from grepping/globbing the target repo>
- `path/to/file.ts` - existing implementation
- `path/to/config.yml` - configuration schema
- `path/to/test.spec.ts` - test suite

## Acceptance Criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] UI matches specification (if applicable)

## UI Specification

<for user-facing features only>

### Expected UI Flow
```sh
◆  Step description
│  ● Option 1
│  ○ Option 2
└
```

### Keyboard Navigation
- Up/Down arrows: Navigate options
- Enter: Select and proceed
- Esc: Go back (if applicable)

### Loading States
```sh
│  ◜ Loading...
│  ◠ Loading...
│  ◝ Loading...
│  ◞ Loading...
```

### Error Handling
```sh
◇  Step failed
│  ✗ Error: descriptive message
│  
◆  Retry?
│  ● Yes
│  ○ No
```

## Implementation Notes

<optional: specific technical hints, edge cases, or gotchas>

## Testing Strategy

<from context/development/testing-checklist.md>
- Unit tests
- Integration tests
- Manual validation steps
- UI testing (if applicable): keyboard navigation, visual output

## Related Context Files

- `context/architecture/repositories.md`
- `context/tech-stack/frontend.md` (or backend.md)
- `context/development/conventions.md`
```

## Metadata structure

### task-index.json

```json
{
  "index_version": "1.0",
  "last_updated": "YYYY-MM-DD",
  "milestones": [
    {
      "milestone_id": "milestone-0-raspberrypi-cli",
      "title": "iac-toolbox cli",
      "metadata_file": "milestone-0-raspberrypi-cli.metadata.json",
      "roadmap_file": "../roadmap/milestone-0-raspberrypi-cli.md",
      "status": "draft",
      "tasks": [
        {
          "task_id": "task-001",
          "title": "ARM64 Validation",
          "file": "../tasks/task-001-arm64-validation.md",
          "ticket_id": "cli-validate-arm64-before-install",
          "status": "draft",
          "issue_number": null,
          "repo": "iac-toolbox-cli"
        }
      ]
    }
  ]
}
```

### milestone-*.metadata.json

```json
{
  "roadmap_item": {
    "id": "milestone-id",
    "title": "Milestone title",
    "source_markdown": "milestone-file.md",
    "project_url": "https://github.com/orgs/IaC-Toolbox/projects/1",
    "status": "draft"
  },
  "tickets": [
    {
      "ticket_id": "stable-slug",
      "title": "Issue title",
      "summary": "Short summary",
      "repo_url": "https://github.com/IaC-Toolbox/repo",
      "issue_number": null,
      "issue_url": null,
      "status": "draft",
      "source_sections": ["Section heading"],
      "acceptance_hints": ["Hint 1", "Hint 2"]
    }
  ]
}
```

## Workflow

**IMPORTANT:** Start by creating a plan using the structure shown in "Execution Mode" above. Show the plan, then execute incrementally.

### Phase 1: Load Context (Plan Item 1)

- Read `context/architecture/repositories.md` - understand repo boundaries
- Read `context/tech-stack/` - identify technologies for each repo
- Read `context/development/conventions.md` - code standards
- Read `context/development/testing-checklist.md` - testing requirements
- Read `context/constraints/` - platform/performance constraints

Mark this phase complete before proceeding.

### Phase 2: Analyze Roadmap (Plan Item 2)

- Load milestone markdown
- Identify logical feature boundaries (h2/h3 headers)
- Group related sections into coherent tasks
- Determine target repo for each task from architecture docs

Mark this phase complete before proceeding.

### Phase 3: Create Task Files (Plan Item 3)

For each identified task:
- Generate task ID: Sequential `task-001`, `task-002`, etc.
- Enrich with context:
  - Architecture details
  - Tech stack specifics
  - Constraints
  - Related files (via grep/glob)
  - UI specifications (if applicable)
  - Testing requirements
- Write individual task file
- Mark sub-task complete

### Phase 4: Update Metadata (Plan Items 4-5)

- Update/create `task-index.json`
- Create/update milestone metadata file
- Verify all roadmap content covered

### Phase 5: Verify

- Ensure all phases complete (all ✔)
- Verify no duplicate task IDs
- Confirm context completeness

## Task ID assignment

- Sequential numbering: `task-001`, `task-002`, `task-003`
- Stable slugs: `task-001-arm64-validation`
- Preserve existing IDs if updating

## Ticket ID generation

Format: `<repo>-<action>-<brief-description>`

Examples:
- `cli-validate-arm64-before-install`
- `cli-add-basic-keyboard-wizard-navigation`
- `cli-ask-for-target-folder-for-raspberry-pi-snippets`

## Status values

- `draft` — task file exists, no issue created
- `created` — GitHub issue created
- `existing` — issue existed before extraction

## Context enrichment guidelines

### Architecture context
From `context/architecture/`:
- Identify which repo owns the feature
- Note integration points with other repos
- Include relevant configuration schemas
- Link to related services/components

### Tech stack context
From `context/tech-stack/`:
- List primary technologies (e.g., Node.js, TypeScript, Ink for CLI)
- Note frameworks and libraries in use
- Include architecture patterns (e.g., React hooks for Ink)
- Reference package dependencies

### Constraints context
From `context/constraints/`:
- Platform requirements (ARM64, x64, macOS)
- Performance considerations
- Security requirements
- Backward compatibility needs

### File discovery
Use Glob/Grep on target repo:
```bash
# Find related implementation files
find /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo> -name "*.ts" -o -name "*.tsx"

# Find configuration schemas
grep -r "interface.*Config" /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo>

# Find existing tests
find /Users/vvasylkovskyi/git/IaC-Toolbox/git/<repo> -name "*.test.ts" -o -name "*.spec.ts"
```

Include discovered files in "Related Files" section.

### Testing strategy
From `context/development/testing-checklist.md`:
- Required validation steps
- Unit test patterns
- Integration test requirements
- Manual testing procedures

### UI/UX context (where applicable)
For user-facing features (especially CLI):
- Include exact terminal UI mockups from roadmap
- Specify keyboard navigation patterns (arrows, Enter, Esc)
- Define prompt structure and formatting
- Show loading animations or state transitions
- Include error message templates
- Reference UI library usage (e.g., Ink components for CLI)

Example for CLI task:
```sh
◆  Enter Raspberry Pi hostname
│  (Current: raspberrypi.local - press Enter to keep)
│  raspberrypi.local
└
```

For backend/infrastructure tasks:
- Configuration file formats
- API response schemas
- Log output examples
- Service status indicators

## Token budget considerations

While enriching context, aim for:
- **Roadmap content**: 200-300 tokens
- **Architecture context**: 100-150 tokens
- **Tech stack context**: 50-100 tokens
- **Constraints**: 50-100 tokens
- **Related files**: 50-100 tokens
- **UI specification**: 100-200 tokens (if applicable)
- **Testing strategy**: 50-100 tokens
- **Total per task**: 600-1000 tokens (with UI) or 500-850 tokens (without UI)

This is more than the original 200-500 tokens but provides significantly better context for agents, especially for user-facing features where exact UI specifications prevent ambiguity.

## Definition of done

- **Plan created and displayed** at start with clear phases
- **All plan items marked complete** (✔)
- All roadmap sections mapped to tasks
- Each task file is 600-1000 tokens (enriched with context)
- **Context sections populated** from `context/` directory
- **Related files discovered** via Glob/Grep
- **UI specifications included** for user-facing features (exact mockups, keyboard nav, loading states, errors)
- **Testing strategy included** from testing-checklist.md
- task-index.json updated
- milestone metadata file created/updated
- No duplicate task IDs
- Clear acceptance criteria in each task
- Agent can implement without asking for clarification
- **Progress tracked visibly** throughout execution
