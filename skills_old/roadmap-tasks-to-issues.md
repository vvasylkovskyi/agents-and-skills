# Roadmap Tasks To Github Issues

**Purpose:** Transfer context-rich task files to GitHub issues while preserving all context.

**Key principle:** Tasks in `context/tasks/` are the source of truth. GitHub issues are a public-facing mirror with all context preserved.

Use `issue-writing` skill at `skills/issue-writing/SKILL.md`. **Search-first approach** (never load full roadmap).

## Workflow

### 1. Search for unticketed tasks (minimal tokens)

```bash
jq -r '.milestones[].tasks[] | select(.status == "draft" and .issue_number == null) | "\(.task_id)|\(.file)|\(.repo)|\(.ticket_id)"' context/metadata/task-index.json
```

If no results: reply `HEARTBEAT_OK` and exit.

### 2. For each unticketed task found:

a. **Read the complete task file** from `context/tasks/`

Task files contain all necessary context:
- Description
- **Context** (architecture, tech stack, constraints)
- **Related Files** (discovered via Glob/Grep)
- **UI Specification** (for user-facing features)
- Acceptance Criteria
- Implementation Notes
- **Testing Strategy**
- Related Context Files

b. **Check for duplicates**:

```bash
gh issue list --repo IaC-Toolbox/<repo> --state all --search "<task title keywords>"
```

c. **Create GitHub issue - copy complete task content**:

**CRITICAL:** Copy the entire task file to the issue body. Use the task file as a template.

Simply map task markdown sections to issue body sections:
```markdown
## Summary
<from task Description>

## Context

### Architecture
<from task Context > Project Architecture section>

### Tech Stack
<from task Context > Technical Stack section>

### Constraints
<from task Context > Constraints section>

### Related Files
<from task Context > Related Files section>

## UI Specification
<from task UI Specification section - if applicable>
- Expected UI Flow
- Keyboard Navigation
- Loading States
- Error Handling

## Acceptance Criteria
<from task Acceptance Criteria - preserve ALL items>

## Implementation Notes
<from task Implementation Notes>

## Testing Strategy
<from task Testing Strategy>

## Related Context
<from task Related Context Files>
```

Write to `/tmp/issue-body.md` and create:
```bash
gh issue create \
  --repo IaC-Toolbox/<repo> \
  --title "<task title>" \
  --body-file /tmp/issue-body.md
```

d. **Update metadata** (both files):

- Update `task-index.json`: set `issue_number`, `issue_url`, `status: "created"`
- Update `context/metadata/milestone-*.metadata.json`: find matching `ticket_id` and update

e. **Commit changes**:

```bash
git add context/metadata/
git commit -m "roadmap: track issue #<N> for <task-id>"
git push
```

## Context Preservation Rules

**CRITICAL:** Copy the complete task file to the GitHub issue. No summarization, no reduction.

Tasks contain:
- Architecture context
- Tech stack details
- Constraints
- Related files
- UI specifications
- Testing strategy

**All sections must appear in the GitHub issue body.**

When agents pick up issues later, they should have the same context as if they read the task file directly.

## Example execution

```bash
# Step 1: Find unticketed
UNTICKETED=$(jq -r '.milestones[].tasks[] | select(.status == "draft" and .issue_number == null) | .task_id' context/metadata/task-index.json)

# If empty: HEARTBEAT_OK
if [[ -z "$UNTICKETED" ]]; then
  echo "HEARTBEAT_OK"
  exit 0
fi

# Step 2: Process each
for task_id in $UNTICKETED; do
  # Get task details
  TASK_FILE=$(jq -r ".milestones[].tasks[] | select(.task_id == \"$task_id\") | .file" context/metadata/task-index.json)
  REPO=$(jq -r ".milestones[].tasks[] | select(.task_id == \"$task_id\") | .repo" context/metadata/task-index.json)

  # Read COMPLETE task file - preserve all sections
  # Map task sections to issue body:
  #   - Description → Summary
  #   - Context (all sub-sections) → Context
  #   - UI Specification → UI Specification
  #   - Acceptance Criteria → Acceptance Criteria
  #   - Implementation Notes → Implementation Notes
  #   - Testing Strategy → Testing Strategy
  #   - Related Context Files → Related Context
  
  # Write to /tmp/issue-body.md
  # Create issue with gh issue create
  # Update both metadata files with issue_number and issue_url
done

# Step 3: Commit
git add context/metadata/ && git commit -m "roadmap: track new issues" && git push
```

## Quality check

Before marking done, verify:
- [ ] Issue body contains all task sections
- [ ] Architecture context preserved
- [ ] Tech stack details included
- [ ] UI specifications present (if applicable)
- [ ] Testing strategy included
- [ ] Related files listed
- [ ] No summarization or context loss

Reply `HEARTBEAT_OK` if no new issues created.
