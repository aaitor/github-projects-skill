---
name: github-project-board
description: Manage GitHub Project Boards (Projects v2) via gh CLI. Read backlogs, claim work, update status, add comments, and communicate between agents through standardized board conventions.
---

# GitHub Project Board Skill

## When to Activate

Use this skill when:
- Checking or reading a project board / backlog / sprint / kanban
- Finding work assigned to you or available to pick up
- Updating issue status on a board
- Adding progress comments to issues
- Creating issues or sub-issues on a board
- Handing off work between agents
- Any mention of "board", "backlog", "sprint", "kanban", "project board"

## Board Discovery (Mandatory First Step)

Before any board operation, discover the board structure. All field IDs and option IDs are resolved dynamically — never hardcode them.

```bash
# 1. Get project metadata and README (workflow rules)
gh project view <NUMBER> --owner <OWNER> --format json

# 2. Get all fields with IDs and option values
gh project field-list <NUMBER> --owner <OWNER> --format json

# 3. Get all items with current field values
gh project item-list <NUMBER> --owner <OWNER> --format json --limit 100
```

From the `field-list` output, resolve:
- **Status field ID** — find the field where `name == "Status"`
- **Status option IDs** — map each option name ("Todo", "In progress", etc.) to its ID
- **Priority field ID** — find the field where `name == "Priority"`
- **Priority option IDs** — map each option name to its ID
- **Size field ID** — find the field where `name == "Size"`
- **Project ID** — from `gh project view` output, the `id` field (starts with `PVT_`)

Read the board README (`gh project view ... --format json | jq -r .readme`) to understand board-specific workflow rules. The README is authoritative for that board's conventions.

## Board Workflow

The standard 6-status lifecycle:

```
In Definition → Todo → In Progress → To Review → Done
                                    → Ready → Done
```

### Who controls what

| Transition | Who |
|---|---|
| In Definition → Todo | Human (issue is ready for work) |
| Todo → In Progress | Agent (claiming work) |
| In Progress → To Review | Agent (needs clarification or review) |
| In Progress → Ready | Agent (work complete, needs final approval) |
| To Review → Todo | Human (feedback given, back to backlog) |
| Ready → Done | Human (approved) |
| Ready → Todo | Human (rejected, needs rework) |

Agents should **never** move items to "In Definition" or "Done" — those are human-controlled states.

## Quick Command Reference

All commands use dynamically discovered IDs from the Board Discovery step.

### List backlog items by status

```bash
gh project item-list <NUMBER> --owner <OWNER> --format json --limit 100 \
  | jq '[.items[] | select(.status == "Todo")]'
```

### Move item to a new status

```bash
gh project item-edit \
  --project-id <PROJECT_ID> \
  --id <ITEM_ID> \
  --field-id <STATUS_FIELD_ID> \
  --single-select-option-id <TARGET_STATUS_OPTION_ID>
```

### Add a comment to an issue

```bash
gh issue comment <ISSUE_URL> --body "<structured comment>"
```

### Create an issue and add to board

```bash
# Create the issue
gh issue create --repo <OWNER>/<REPO> --title "<title>" --body "<body>"

# Add to board (use the returned issue URL)
gh project item-add <NUMBER> --owner <OWNER> --url <ISSUE_URL>
```

### Set priority or size

```bash
gh project item-edit \
  --project-id <PROJECT_ID> \
  --id <ITEM_ID> \
  --field-id <PRIORITY_FIELD_ID> \
  --single-select-option-id <PRIORITY_OPTION_ID>
```

### Create a sub-issue

```bash
gh issue create --repo <OWNER>/<REPO> \
  --title "<sub-issue title>" \
  --body "Parent: <PARENT_ISSUE_URL>"

# Then add to the board and set fields
```

### Open board in browser

```bash
gh project view <NUMBER> --owner <OWNER> --web
```

## Agent Comment Convention

When commenting on issues, use this structured format so other agents and humans can parse updates consistently:

```markdown
## Agent Update — <AGENT_NAME>

**Status**: Working | Blocked | Complete | Needs Review
**Action**: <brief description of what was done>

### Details

<fuller description of the work, findings, or changes>

### Blockers (if any)

- <blocker description>

### Next Steps

- <next action to take>

---
_Updated by `<AGENT_NAME>` at <ISO 8601 timestamp>_
```

## Agent Workflow Patterns

### Pick Up Work

1. **Discover** the board (see Board Discovery above)
2. **List** items with status "Todo"
3. **Sort** by priority (Critical > Very High > High > Average > Low > Very Low > Zero)
4. **Claim** the highest-priority item by moving it to "In Progress"
5. **Comment** on the issue with an "Agent Update" noting you've started

### Work Cycle

1. **Read** the issue description, comments, and attachments for requirements
2. **Implement** the requested work
3. **Comment** periodically with progress updates
4. **Move** to "To Review" (if you need feedback) or "Ready" (if work is complete)
5. **Comment** with a final summary of what was done

### Ask for Clarification

1. Move the issue to **"To Review"**
2. Add a comment with your question using the Agent Comment Convention
3. Set `**Status**: Blocked` in the comment
4. A human will review, answer in comments, and move back to "Todo"
5. Pick it up again from the backlog

### Submit for Approval

1. Move the issue to **"Ready"**
2. Add a comment with:
   - Summary of all changes made
   - Links to PRs or commits
   - Any caveats or known limitations
3. Set `**Status**: Complete` in the comment

### Agent Handoff

1. Complete your portion of work
2. Add a structured comment summarizing what was done and what remains
3. Move the issue back to **"Todo"** (or keep in "In Progress" if the next agent should continue immediately)
4. The next agent picks it up following the "Pick Up Work" pattern

## Quality Gate

Before marking any board operation as complete, verify:

- [ ] Status field was actually updated (re-read the item to confirm)
- [ ] Comment follows the Agent Comment Convention
- [ ] No stale "In Progress" items left behind (if you're done with an item, move it)
- [ ] Board README rules were followed

See `references/` for detailed command reference, GraphQL recipes, and expanded workflow patterns.
