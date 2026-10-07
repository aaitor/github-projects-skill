# Agent Workflow Patterns

Detailed playbooks for common agent workflows on GitHub Project Boards. All patterns start with board discovery.

## Prerequisites

Every pattern begins with:

```bash
# Store these for the session
PROJECT_NUMBER=<NUMBER>
OWNER=<OWNER>
AGENT_LABEL=""  # Optional: set to a label name (e.g., "research") to scope this agent

# Discover board
PROJECT_JSON=$(gh project view $PROJECT_NUMBER --owner $OWNER --format json)
PROJECT_ID=$(echo "$PROJECT_JSON" | jq -r '.id')
BOARD_README=$(echo "$PROJECT_JSON" | jq -r '.readme')

# Discover fields
FIELDS_JSON=$(gh project field-list $PROJECT_NUMBER --owner $OWNER --format json)

# Resolve Status field
STATUS_FIELD_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Status") | .id')
TODO_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Status") | .options[] | select(.name == "Todo") | .id')
IN_PROGRESS_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Status") | .options[] | select(.name == "In progress") | .id')
TO_REVIEW_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Status") | .options[] | select(.name == "To Review") | .id')
TO_APPROVE_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Status") | .options[] | select(.name == "Ready") | .id')

# Resolve Priority field
PRIORITY_FIELD_ID=$(echo "$FIELDS_JSON" | jq -r '.fields[] | select(.name == "Priority") | .id')

# Get items
ITEMS_JSON=$(gh project item-list $PROJECT_NUMBER --owner $OWNER --format json --limit 100)
```

## Pattern 1: Pick Up Work

**Goal**: Find and claim the highest-priority available item from the backlog.

### Steps

1. **Filter items by "Todo" status** (and optionally by label):

```bash
# All Todo items (no label filter)
echo "$ITEMS_JSON" | jq '[.items[] | select(.status == "Todo")]'
```

If the agent has a label scope (`AGENT_LABEL` is set), use GraphQL to filter by label (see `graphql-recipes.md` recipe #6):

```bash
# Todo items with a specific label (via GraphQL)
gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    user(login: $owner) {
      projectV2(number: $number) {
        items(first: 100) {
          nodes {
            id
            fieldValueByName(name: "Status") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
            fieldValueByName(name: "Priority") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
            content {
              ... on Issue {
                number
                title
                url
                labels(first: 20) {
                  nodes { name }
                }
              }
            }
          }
        }
      }
    }
  }
' -f owner="$OWNER" -F number=$PROJECT_NUMBER \
  --jq ".data.user.projectV2.items.nodes[]
    | select(.content.labels.nodes[]?.name == \"$AGENT_LABEL\")
    | select(.fieldValueByName.name == \"Todo\")
    | {id, title: .content.title, number: .content.number, priority: (.fieldValueByName // {} | .name)}"
```

2. **Read board README** to check for any priority rules or constraints:

```bash
echo "$BOARD_README"
```

3. **Sort by priority** (Critical first). The priority order is:
   - Critical > Very High > High > Average > Low > Very Low > Zero

The GraphQL query above returns priority directly. For the CLI approach, use GraphQL recipe #2 for priority-aware sorting, or read each issue to check.

4. **Claim the item** by moving to "In Progress":

```bash
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "<ITEM_ID>" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$IN_PROGRESS_ID"
```

5. **Comment on the issue**:

```bash
gh issue comment "<ISSUE_URL>" --body "## Agent Update — <AGENT_NAME>

**Status**: Working
**Action**: Picked up from backlog

### Details

Starting work on this issue. Priority: <PRIORITY>.

### Next Steps

- Review requirements in issue description
- Begin implementation

---
_Updated by \`<AGENT_NAME>\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

## Pattern 2: Work Cycle

**Goal**: Implement work on a claimed issue with progress tracking.

### Steps

1. **Read the full issue** for requirements:

```bash
gh issue view "<ISSUE_URL>" --json title,body,comments,labels,assignees
```

2. **Implement** the requested changes.

3. **Post progress comments** at meaningful milestones:

```bash
gh issue comment "<ISSUE_URL>" --body "## Agent Update — <AGENT_NAME>

**Status**: Working
**Action**: <milestone description>

### Details

<what was accomplished in this phase>

### Next Steps

- <remaining work>

---
_Updated by \`<AGENT_NAME>\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

4. **Move to final status** when done:

If work is complete and ready for final approval:
```bash
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "<ITEM_ID>" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$TO_APPROVE_ID"
```

If work needs review or feedback:
```bash
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "<ITEM_ID>" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$TO_REVIEW_ID"
```

## Pattern 3: Ask for Clarification

**Goal**: Request human input when requirements are unclear or a decision is needed.

### Steps

1. **Move to "To Review"**:

```bash
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "<ITEM_ID>" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$TO_REVIEW_ID"
```

2. **Post a question comment**:

```bash
gh issue comment "<ISSUE_URL>" --body "## Agent Update — <AGENT_NAME>

**Status**: Blocked
**Action**: Requesting clarification

### Details

<context about what you've found or attempted>

### Questions

1. <specific question>
2. <specific question>

### Next Steps

- Waiting for human feedback
- Will resume once questions are answered

---
_Updated by \`<AGENT_NAME>\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

3. **Wait**: A human will review, add comments with answers, and move the issue back to "Todo".

4. **Pick up again**: Follow Pattern 1 to reclaim the issue from the backlog.

## Pattern 4: Submit for Approval

**Goal**: Present completed work for final human validation.

### Steps

1. **Move to "Ready"**:

```bash
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "<ITEM_ID>" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$TO_APPROVE_ID"
```

2. **Post a completion summary**:

```bash
gh issue comment "<ISSUE_URL>" --body "## Agent Update — <AGENT_NAME>

**Status**: Complete
**Action**: Submitting for approval

### Summary

<comprehensive summary of all changes made>

### Changes

- <change 1 with PR/commit link>
- <change 2 with PR/commit link>

### Testing

- <how the changes were tested>

### Known Limitations

- <any caveats or edge cases>

---
_Updated by \`<AGENT_NAME>\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

## Pattern 5: Sub-issue Decomposition

**Goal**: Break a large issue into smaller, trackable sub-issues.

### Steps

1. **Analyze the parent issue** for discrete work units:

```bash
gh issue view "<PARENT_ISSUE_URL>" --json title,body,comments
```

2. **Create sub-issues** (add `--label` to route to a specific agent):

```bash
# gh issue create prints the new issue URL to stdout — capture it directly.
SUB_URL=$(gh issue create --repo <OWNER>/<REPO> \
  --title "<parent title> — <sub-task description>" \
  --label "<LABEL>" \
  --body "Parent: <PARENT_ISSUE_URL>

## Scope

<specific scope of this sub-issue>

## Acceptance Criteria

- [ ] <criterion 1>
- [ ] <criterion 2>")
```

3. **Add each sub-issue to the board**:

```bash
gh project item-add $PROJECT_NUMBER --owner $OWNER --url "$SUB_URL"
```

4. **Set priority and size** on each sub-issue:

```bash
# Get the new item ID
NEW_ITEM_ID=$(gh project item-list $PROJECT_NUMBER --owner $OWNER --format json --limit 100 \
  | jq -r ".items[] | select(.content.url == \"$SUB_URL\") | .id")

# Set priority
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "$NEW_ITEM_ID" \
  --field-id "$PRIORITY_FIELD_ID" \
  --single-select-option-id "<PRIORITY_OPTION_ID>"
```

5. **Comment on the parent issue** with a summary:

```bash
gh issue comment "<PARENT_ISSUE_URL>" --body "## Agent Update — <AGENT_NAME>

**Status**: Complete
**Action**: Decomposed into sub-issues

### Sub-issues Created

- [ ] <sub-issue 1 URL> — <brief description>
- [ ] <sub-issue 2 URL> — <brief description>
- [ ] <sub-issue 3 URL> — <brief description>

### Next Steps

- Sub-issues added to board with priorities set
- Each can be picked up independently

---
_Updated by \`<AGENT_NAME>\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

## Pattern 6: Agent Handoff

**Goal**: Transfer work from one agent to another with full context.

### Steps

1. **Document what was completed and what remains**:

```bash
gh issue comment "<ISSUE_URL>" --body "## Agent Update — <CURRENT_AGENT>

**Status**: Complete (partial)
**Action**: Handing off to next agent

### Completed Work

- <completed item 1>
- <completed item 2>

### Remaining Work

- <remaining item 1>
- <remaining item 2>

### Context for Next Agent

<important context, decisions made, gotchas encountered>

### Files Modified

- \`path/to/file1\` — <what changed>
- \`path/to/file2\` — <what changed>

---
_Updated by \`<CURRENT_AGENT>\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

2. **Move to "Todo"** so the next agent can pick it up:

```bash
gh project item-edit \
  --project-id "$PROJECT_ID" \
  --id "<ITEM_ID>" \
  --field-id "$STATUS_FIELD_ID" \
  --single-select-option-id "$TODO_ID"
```

3. The next agent follows **Pattern 1** to claim the issue.
