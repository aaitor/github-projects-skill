# Walkthrough: An Agent Working a Board

End-to-end walkthrough of an agent picking up and completing work on a board.
It uses a fictional board — owner `octo-org`, project #1, repo `octo-org/webapp` —
so you can map every command to your own board by swapping the owner and number.

> The field and option IDs below (`PVT_…`, `f75ad846`, …) are illustrative. Yours
> will differ — always resolve them at runtime with the discovery commands in Step 1.
> Never hardcode them.

## Step 1: Discover the Board

### Project metadata and README

```bash
$ gh project view 1 --owner octo-org --format json
```

```json
{
  "id": "PVT_kwHOABpYtM4BSMQL",
  "number": 1,
  "title": "Engineering Backlog",
  "closed": false,
  "readme": "This board tracks AI-agent work.\n\n- New issues start in \"In Definition\". When fully specified they move to \"Todo\".\n- \"Todo\" items are ready to be implemented...",
  "url": "https://github.com/orgs/octo-org/projects/1"
}
```

Key values to capture:

- **Project ID**: `PVT_kwHOABpYtM4BSMQL` — required for every `item-edit`.
- **Board README**: the authoritative description of this board's workflow rules.

### Fields and options

```bash
$ gh project field-list 1 --owner octo-org --format json
```

```json
{
  "fields": [
    {"id": "PVTF_lAHOABpYtM4BSMQLzg_zFoY", "name": "Title", "type": "ProjectV2Field"},
    {"id": "PVTF_lAHOABpYtM4BSMQLzg_zFoc", "name": "Assignees", "type": "ProjectV2Field"},
    {
      "id": "PVTSSF_lAHOABpYtM4BSMQLzg_zFog",
      "name": "Status",
      "type": "ProjectV2SingleSelectField",
      "options": [
        {"id": "c527c777", "name": "In Definition"},
        {"id": "f75ad846", "name": "Todo"},
        {"id": "47fc9ee4", "name": "In progress"},
        {"id": "91a30e6b", "name": "To Review"},
        {"id": "d68422da", "name": "Ready"},
        {"id": "98236657", "name": "Done"}
      ]
    },
    {
      "id": "PVTSSF_lAHOABpYtM4BSMQLzg_zFr8",
      "name": "Priority",
      "type": "ProjectV2SingleSelectField",
      "options": [
        {"id": "a9105b3c", "name": "Critical"},
        {"id": "9fe5bcdf", "name": "Very High"},
        {"id": "419b8c12", "name": "High"},
        {"id": "b15e811d", "name": "Average"},
        {"id": "aced06ae", "name": "Low"},
        {"id": "1439d9b0", "name": "Very Low"},
        {"id": "890fd880", "name": "Zero"}
      ]
    },
    {
      "id": "PVTSSF_lAHOABpYtM4BSMQLzg_zFsA",
      "name": "Size",
      "type": "ProjectV2SingleSelectField",
      "options": [
        {"id": "d1a5ff40", "name": "XL"},
        {"id": "237d5f9e", "name": "L"},
        {"id": "329288a7", "name": "M"},
        {"id": "5ce5b392", "name": "S"},
        {"id": "972adeea", "name": "XS"}
      ]
    }
  ]
}
```

Resolved field map for this session:

| Field | Field ID | Key options |
|-------|----------|-------------|
| Status | `PVTSSF_…zFog` | Todo: `f75ad846`, In progress: `47fc9ee4`, To Review: `91a30e6b`, Ready: `d68422da` |
| Priority | `PVTSSF_…zFr8` | Critical: `a9105b3c`, Very High: `9fe5bcdf`, High: `419b8c12` |
| Size | `PVTSSF_…zFsA` | XL: `d1a5ff40`, L: `237d5f9e`, M: `329288a7`, S: `5ce5b392`, XS: `972adeea` |

## Step 2: List Todo Items

### All Todo items (no label filter)

```bash
$ gh project item-list 1 --owner octo-org --format json --limit 100 \
    | jq '[.items[] | select(.status == "Todo")]'
```

### Todo items scoped to a label (e.g. a "research" agent)

```bash
$ gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    organization(login: $owner) {
      projectV2(number: $number) {
        items(first: 100) {
          nodes {
            id
            fieldValueByName(name: "Status") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
            content {
              ... on Issue {
                number
                title
                url
                labels(first: 20) { nodes { name } }
              }
            }
          }
        }
      }
    }
  }
' -f owner="octo-org" -F number=1 \
  --jq '.data.organization.projectV2.items.nodes[]
    | select(.content.labels.nodes[]?.name == "research")
    | select(.fieldValueByName.name == "Todo")
    | {id, title: .content.title, number: .content.number}'
```

> Use `organization(login: …)` for org-owned boards and `user(login: …)` for
> user-owned boards.

Example output:

```json
[
  {
    "id": "PVTI_example123",
    "title": "Implement auth middleware",
    "status": "Todo",
    "content": {
      "type": "Issue",
      "number": 42,
      "repository": "octo-org/webapp",
      "url": "https://github.com/octo-org/webapp/issues/42"
    }
  }
]
```

## Step 3: Claim an Issue

Move the highest-priority "Todo" item to "In progress":

```bash
$ gh project item-edit \
    --project-id "PVT_kwHOABpYtM4BSMQL" \
    --id "PVTI_example123" \
    --field-id "PVTSSF_lAHOABpYtM4BSMQLzg_zFog" \
    --single-select-option-id "47fc9ee4"
```

## Step 4: Add a Progress Comment

```bash
$ gh issue comment "https://github.com/octo-org/webapp/issues/42" --body "## Agent Update — Claude Code

**Status**: Working
**Action**: Picked up from backlog, starting implementation

### Details

Reviewing issue requirements and related code. Will implement auth middleware as described.

### Next Steps

- Analyze existing middleware patterns in the codebase
- Implement the auth middleware
- Write tests
- Submit PR

---
_Updated by \`Claude Code\` at 2026-03-19T10:30:00Z_"
```

## Step 5: Submit for Approval

After completing the work:

```bash
# Move to "Ready"
$ gh project item-edit \
    --project-id "PVT_kwHOABpYtM4BSMQL" \
    --id "PVTI_example123" \
    --field-id "PVTSSF_lAHOABpYtM4BSMQLzg_zFog" \
    --single-select-option-id "d68422da"

# Add completion comment
$ gh issue comment "https://github.com/octo-org/webapp/issues/42" --body "## Agent Update — Claude Code

**Status**: Complete
**Action**: Auth middleware implemented, submitting for approval

### Summary

Implemented JWT-based authentication middleware following existing patterns in the codebase.

### Changes

- PR: https://github.com/octo-org/webapp/pull/43
- Added \`src/middleware/auth.ts\` — JWT validation middleware
- Added \`src/middleware/auth.test.ts\` — Unit tests (92% coverage)
- Updated \`src/routes/index.ts\` — Applied middleware to protected routes

### Testing

- All unit tests passing
- Integration tests verified against test JWT tokens
- Manually tested with curl against local dev server

### Known Limitations

- Token refresh not implemented (out of scope for this issue)

---
_Updated by \`Claude Code\` at 2026-03-19T14:45:00Z_"
```

## Step 6: Verify

Confirm the status was updated:

```bash
$ gh project item-list 1 --owner octo-org --format json --limit 100 \
    | jq '.items[] | select(.id == "PVTI_example123") | .status'
"Ready"
```

The issue is now waiting for human approval. The human will either:

- Move it to "Done" (approved), or
- Move it to "Todo" with feedback comments (needs rework).
