# Board Setup Guide

How to create a GitHub Projects v2 board from scratch that works with this skill.
Every command uses the `gh` CLI, so the whole board can be provisioned without
touching the web UI.

## 0. Prerequisites

```bash
gh auth login                 # if not already authenticated
gh auth refresh -s project    # grant the 'project' scope
```

Pick an **owner**. A board can be owned by a user or an organization:

- User board:    `--owner <your-username>`
- Org board:     `--owner <org-name>`

The rest of this guide uses `<OWNER>` as a placeholder.

## 1. Create the Project

```bash
gh project create --owner <OWNER> --title "Engineering Backlog"
```

This prints the new project's number and URL. Capture the **number** — every other
command needs it:

```bash
PROJECT_NUMBER=<NUMBER>   # from the create output
OWNER=<OWNER>
```

## 2. Add the Status Field

The skill's workflow is built around a six-stage `Status` field. Create it with all
six options in a single command (order matters — it sets the column order on the board):

```bash
gh project field-create $PROJECT_NUMBER --owner $OWNER \
  --name "Status" \
  --data-type SINGLE_SELECT \
  --single-select-options "In Definition,Todo,In progress,To Review,Ready,Done"
```

| Option | Meaning | Who moves items here |
|--------|---------|----------------------|
| **In Definition** | Idea captured, not yet ready to work | Human |
| **Todo** | Fully specified, ready to pick up | Human |
| **In progress** | An agent is actively working it | Agent |
| **To Review** | Needs human feedback / a decision | Agent |
| **Ready** | Work complete, awaiting approval | Agent |
| **Done** | Approved and closed | Human |

> GitHub Projects boards already ship with a default `Status` field (Todo / In Progress /
> Done). If yours exists, edit it in the web UI to match these six options rather than
> creating a duplicate — the CLI cannot yet add options to an existing single-select field.

## 3. Add the Priority Field

```bash
gh project field-create $PROJECT_NUMBER --owner $OWNER \
  --name "Priority" \
  --data-type SINGLE_SELECT \
  --single-select-options "Critical,Very High,High,Average,Low,Very Low,Zero"
```

Agents sort the backlog by this field, highest first:
`Critical > Very High > High > Average > Low > Very Low > Zero`.

## 4. Add the Size Field

```bash
gh project field-create $PROJECT_NUMBER --owner $OWNER \
  --name "Size" \
  --data-type SINGLE_SELECT \
  --single-select-options "XL,L,M,S,XS"
```

`Size` is informational (effort estimate). The skill reads it but does not require it.

## 5. Write the Board README

The board README is the **single most important setup step**. Agents read it at
runtime and treat it as authoritative for that board's conventions. Put your rules
there — anything board-specific that the skill can't know in advance.

```bash
gh project edit $PROJECT_NUMBER --owner $OWNER --readme "$(cat <<'EOF'
# How this board works

This board coordinates AI-agent and human work.

## Status lifecycle

- **In Definition** — new issues land here. A human fleshes out the details.
- **Todo** — ready to be worked. Agents pick up the highest-priority Todo item.
- **In progress** — an agent is actively working the item.
- **To Review** — the agent needs human input; see the latest comment.
- **Ready** — work is complete and awaiting human approval.
- **Done** — approved and closed. Humans only.

## Rules for agents

- Only move items between: Todo → In progress → (To Review | Ready).
- Never move items to "In Definition" or "Done" — those are human-controlled.
- Always leave an "Agent Update" comment when you change status.
- Sort the backlog by Priority, highest first.

## Multi-agent scoping (optional)

Agents may be scoped by issue label (e.g. `research`, `coding`, `docs`).
An agent with no label scope may pick up any Todo item.
EOF
)"
```

## 6. Verify

```bash
# Confirm the fields and options
gh project field-list $PROJECT_NUMBER --owner $OWNER --format json \
  | jq '.fields[] | {name, options: (.options // [] | map(.name))}'

# Confirm the README
gh project view $PROJECT_NUMBER --owner $OWNER --format json | jq -r '.readme'
```

Expected field output:

```json
{"name": "Status", "options": ["In Definition","Todo","In progress","To Review","Ready","Done"]}
{"name": "Priority", "options": ["Critical","Very High","High","Average","Low","Very Low","Zero"]}
{"name": "Size", "options": ["XL","L","M","S","XS"]}
```

## 7. Add Work to the Board

```bash
# Create an issue and add it to the board.
# gh issue create prints the new issue URL to stdout, so capture it directly.
ISSUE_URL=$(gh issue create --repo <OWNER>/<REPO> \
  --title "Implement auth middleware" \
  --body "Add JWT validation to protected routes.")

gh project item-add $PROJECT_NUMBER --owner $OWNER --url "$ISSUE_URL"
```

New items default to the first Status option (`In Definition`). Move them to `Todo`
once they're ready for an agent to pick up. From here, point an agent at the board and
follow [`examples/walkthrough.md`](../examples/walkthrough.md).

## Customizing the Workflow

The skill resolves all field and option IDs dynamically, so you can rename or reorder
options without changing any code — the agent reads whatever is on the board. If you
change the **meaning** of the workflow (e.g. add a "Blocked" status or drop "To Review"),
document it in the board README so agents behave accordingly.
