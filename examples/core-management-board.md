# Example: Core Management Board

End-to-end walkthrough using the "Core Management" board (project #7, owner: aaitor).

## Step 1: Discover Board

### Get project metadata

```bash
$ gh project view 7 --owner aaitor --format json
```

```json
{
  "id": "PVT_kwHOABpYtM4BSMQL",
  "number": 7,
  "title": "Core Management",
  "closed": false,
  "readme": "This Project Board allows to control the AI Agent tasks. Board Management:\n\n- New issues are created in the \"In Definition\" status. When they have enough details, and they are ready to be started they move to the \"Todo\" status\n- Issues in the \"Todo\" status are ready to be implemented...",
  "url": "https://github.com/users/aaitor/projects/7"
}
```

Key values:
- **Project ID**: `PVT_kwHOABpYtM4BSMQL`
- **Board README**: Describes the 6-status workflow with human/agent responsibilities

### Get fields and options

```bash
$ gh project field-list 7 --owner aaitor --format json
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

Resolved field map:

| Field | Field ID | Key Options |
|-------|----------|-------------|
| Status | `PVTSSF_...zFog` | Todo: `f75ad846`, In progress: `47fc9ee4`, To Review: `91a30e6b`, Ready: `d68422da` |
| Priority | `PVTSSF_...zFr8` | Critical: `a9105b3c`, Very High: `9fe5bcdf`, High: `419b8c12` |
| Size | `PVTSSF_...zFsA` | XL: `d1a5ff40`, L: `237d5f9e`, M: `329288a7`, S: `5ce5b392`, XS: `972adeea` |

## Step 2: List Todo Items

```bash
$ gh project item-list 7 --owner aaitor --format json --limit 100 \
    | jq '[.items[] | select(.status == "Todo")]'
```

```json
[
  {
    "id": "PVTI_example123",
    "title": "Implement auth middleware",
    "status": "Todo",
    "content": {
      "type": "Issue",
      "number": 42,
      "repository": "aaitor/core",
      "url": "https://github.com/aaitor/core/issues/42"
    }
  }
]
```

## Step 3: Claim an Issue

Move the highest-priority "Todo" item to "In Progress":

```bash
$ gh project item-edit \
    --project-id "PVT_kwHOABpYtM4BSMQL" \
    --id "PVTI_example123" \
    --field-id "PVTSSF_lAHOABpYtM4BSMQLzg_zFog" \
    --single-select-option-id "47fc9ee4"
```

## Step 4: Add Progress Comment

```bash
$ gh issue comment "https://github.com/aaitor/core/issues/42" --body "## Agent Update — Claude Code

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
$ gh issue comment "https://github.com/aaitor/core/issues/42" --body "## Agent Update — Claude Code

**Status**: Complete
**Action**: Auth middleware implemented, submitting for approval

### Summary

Implemented JWT-based authentication middleware following existing patterns in the codebase.

### Changes

- PR: https://github.com/aaitor/core/pull/43
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
$ gh project item-list 7 --owner aaitor --format json --limit 100 \
    | jq '.items[] | select(.id == "PVTI_example123") | .status'
"Ready"
```

The issue is now waiting for human approval. The human will either:
- Move to "Done" (approved)
- Move to "Todo" with feedback comments (needs rework)
