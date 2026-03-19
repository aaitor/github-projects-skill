# gh CLI Command Reference for GitHub Projects v2

Complete command reference for managing GitHub Projects v2 boards via the `gh` CLI.

## Board Discovery

### List all projects for an owner

```bash
gh project list --owner <OWNER> --format json
```

Output includes `number`, `title`, `id` for each project.

### View project metadata and README

```bash
gh project view <NUMBER> --owner <OWNER> --format json
```

Key fields:
- `id` — the project node ID (e.g., `PVT_kwHOABpYtM4BSMQL`), required for `item-edit`
- `readme` — board workflow rules (agents must read this)
- `title`, `url`, `closed`

Extract just the README:

```bash
gh project view <NUMBER> --owner <OWNER> --format json | jq -r '.readme'
```

### List all fields with IDs and options

```bash
gh project field-list <NUMBER> --owner <OWNER> --format json
```

Output structure:

```json
{
  "fields": [
    {
      "id": "PVTSSF_...",
      "name": "Status",
      "type": "ProjectV2SingleSelectField",
      "options": [
        {"id": "f75ad846", "name": "Todo"},
        {"id": "47fc9ee4", "name": "In progress"}
      ]
    }
  ]
}
```

### Field Resolution Pattern

To resolve a field name to its ID and option IDs:

```bash
# Get Status field ID
STATUS_FIELD_ID=$(gh project field-list <NUMBER> --owner <OWNER> --format json \
  | jq -r '.fields[] | select(.name == "Status") | .id')

# Get "In progress" option ID
IN_PROGRESS_ID=$(gh project field-list <NUMBER> --owner <OWNER> --format json \
  | jq -r '.fields[] | select(.name == "Status") | .options[] | select(.name == "In progress") | .id')

# Get Priority field ID
PRIORITY_FIELD_ID=$(gh project field-list <NUMBER> --owner <OWNER> --format json \
  | jq -r '.fields[] | select(.name == "Priority") | .id')
```

### List all board items

```bash
gh project item-list <NUMBER> --owner <OWNER> --format json --limit 100
```

Output structure:

```json
{
  "items": [
    {
      "id": "PVTI_...",
      "title": "Issue title",
      "status": "Todo",
      "content": {
        "type": "Issue",
        "number": 42,
        "repository": "owner/repo",
        "url": "https://github.com/owner/repo/issues/42",
        "body": "Issue description..."
      }
    }
  ]
}
```

Filter by status:

```bash
gh project item-list <NUMBER> --owner <OWNER> --format json --limit 100 \
  | jq '[.items[] | select(.status == "Todo")]'
```

## Item Management

### Add an existing issue to the board

```bash
gh project item-add <NUMBER> --owner <OWNER> --url <ISSUE_URL>
```

Returns the new item ID.

### Create a draft item (no linked issue)

```bash
gh project item-create <NUMBER> --owner <OWNER> --title "<title>" --body "<body>"
```

### Edit an item's field value

```bash
# Set a SingleSelect field (Status, Priority, Size)
gh project item-edit \
  --project-id <PROJECT_ID> \
  --id <ITEM_ID> \
  --field-id <FIELD_ID> \
  --single-select-option-id <OPTION_ID>

# Set a text field
gh project item-edit \
  --project-id <PROJECT_ID> \
  --id <ITEM_ID> \
  --field-id <FIELD_ID> \
  --text "<value>"

# Set a number field
gh project item-edit \
  --project-id <PROJECT_ID> \
  --id <ITEM_ID> \
  --field-id <FIELD_ID> \
  --number <value>

# Set a date field
gh project item-edit \
  --project-id <PROJECT_ID> \
  --id <ITEM_ID> \
  --field-id <FIELD_ID> \
  --date "2026-03-19"
```

### Archive an item

```bash
gh project item-archive <NUMBER> --owner <OWNER> --id <ITEM_ID>
```

### Delete an item from the board

```bash
gh project item-delete <NUMBER> --owner <OWNER> --id <ITEM_ID>
```

## Issue Operations

### Create an issue

```bash
gh issue create --repo <OWNER>/<REPO> \
  --title "<title>" \
  --body "<body>" \
  --label "<label1>,<label2>" \
  --assignee "<username>"
```

### Add a comment

```bash
gh issue comment <ISSUE_URL> --body "<comment body>"

# Or by number
gh issue comment <NUMBER> --repo <OWNER>/<REPO> --body "<comment body>"
```

### Edit an issue

```bash
gh issue edit <ISSUE_URL> --title "<new title>" --body "<new body>"

# Add labels
gh issue edit <ISSUE_URL> --add-label "bug,urgent"

# Assign
gh issue edit <ISSUE_URL> --add-assignee "<username>"
```

### List issues in a repo

```bash
gh issue list --repo <OWNER>/<REPO> --state open --json number,title,labels,assignees
```

### View issue details

```bash
gh issue view <ISSUE_URL> --json title,body,comments,labels,assignees,state
```

## Gotchas

1. **`--project-id` vs `--owner` + `<NUMBER>`**: `item-edit` requires `--project-id` (the node ID like `PVT_...`), while `item-list`, `field-list`, `view` use `--owner` + `<NUMBER>`.

2. **Item ID vs Issue number**: Board item IDs (`PVTI_...`) are different from issue numbers. Use `item-list` to find the board item ID for an issue.

3. **`--limit` default is 30**: Always pass `--limit 100` (or higher) to `item-list` to avoid missing items.

4. **JSON output**: Always use `--format json` for machine-readable output. The default table format is not parseable.

5. **Iteration fields**: Iteration fields use `--iteration-id` instead of `--single-select-option-id`.
