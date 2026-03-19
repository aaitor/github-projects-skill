# GitHub Project Board Skill

A skill that enables AI agents (Claude Code, OpenClaw, etc.) to interact with GitHub Project Boards (Projects v2) via the `gh` CLI. Agents can read backlogs, claim work, update status, add structured comments, and hand off between each other through standardized board conventions.

## Why

GitHub Projects v2 provides a structured way to manage work. This skill gives AI agents the vocabulary and workflow to participate in that process — picking up issues, reporting progress, and coordinating with humans and other agents through a shared board.

## Prerequisites

- **`gh` CLI** installed and authenticated
- **`project` scope** enabled: `gh auth refresh -s project`
- A GitHub Projects v2 board with standard fields (Status, Priority, Size)

Verify access:

```bash
gh project list --owner <OWNER>
```

## Installation

### Claude Code

Symlink or copy the skill and references into your Claude Code skills directory:

```bash
# Symlink (recommended — stays in sync with repo updates)
ln -s /path/to/github-projects-skill ~/.claude/skills/github-project-board

# Or copy
cp -r /path/to/github-projects-skill ~/.claude/skills/github-project-board
```

The skill will appear as `github-project-board` in your available skills.

### Other Agents

Reference `SKILL.md` content as system prompt or context. The `references/` directory contains detailed command references and workflow patterns that can be included as additional context.

## Board Setup

Your GitHub Projects v2 board should have these standard fields:

| Field | Type | Options |
|-------|------|---------|
| **Status** | SingleSelect | In Definition, Todo, In progress, To Review, Ready, Done |
| **Priority** | SingleSelect | Critical, Very High, High, Average, Low, Very Low, Zero |
| **Size** | SingleSelect | XL, L, M, S, XS |

Add a **README** to your board (via board settings) describing your workflow rules. Agents read this README at runtime to understand board-specific conventions.

## Quick Start

Once installed, an agent can discover and work with any board:

```bash
# Discover board structure
gh project view 7 --owner aaitor --format json
gh project field-list 7 --owner aaitor --format json

# List Todo items
gh project item-list 7 --owner aaitor --format json | jq '.items[] | select(.status == "Todo")'

# Claim an issue (move to "In Progress")
gh project item-edit --project-id <PROJECT_ID> --id <ITEM_ID> --field-id <STATUS_FIELD_ID> --single-select-option-id <IN_PROGRESS_OPTION_ID>

# Add a progress comment
gh issue comment <ISSUE_URL> --body "## Agent Update — Claude Code
**Status**: Working
**Action**: Started implementation
### Details
Picked up from backlog, beginning work.
---
_Updated by \`Claude Code\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

See `examples/core-management-board.md` for a full walkthrough with real output.

## Repo Structure

```
├── README.md                           # This file
├── SKILL.md                            # The skill definition (install this)
├── references/
│   ├── gh-cli-commands.md              # Full command reference with examples
│   ├── graphql-recipes.md              # Advanced queries (assignee filter, bulk ops)
│   └── agent-workflow-patterns.md      # Step-by-step agent playbooks
├── examples/
│   └── core-management-board.md        # Walkthrough using a real board
└── LICENSE                             # MIT license
```

## License

MIT
