# GitHub Project Board Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A skill that lets AI agents (Claude Code, OpenClaw, or any agent with shell access)
drive **GitHub Project Boards (Projects v2)** through the `gh` CLI. Agents read
backlogs, claim work, update status, post structured progress comments, and hand off to
each other through a shared set of board conventions.

## Why

GitHub Projects v2 is a structured place to manage work. This skill gives an agent the
vocabulary and workflow to participate in that process the way a teammate would — pick up
the highest-priority issue, report progress, ask for clarification, and submit finished
work for approval. Multiple agents can share one board without colliding.

## Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [The Board Model](#the-board-model)
- [Multi-Agent Setup](#multi-agent-setup)
- [Documentation](#documentation)
- [Repo Structure](#repo-structure)
- [Contributing](#contributing)
- [License](#license)

## Prerequisites

- **`gh` CLI** installed and authenticated.
- **`project` scope** enabled: `gh auth refresh -s project`.
- A GitHub Projects v2 board (see [Board Setup Guide](docs/board-setup.md) to create one).

Verify access:

```bash
gh project list --owner <OWNER>
```

## Quick Start

Once the skill is installed, an agent can discover and work any board. Replace `<OWNER>`
and `<NUMBER>` with your board's owner and project number.

```bash
# Discover board structure (field and option IDs are resolved at runtime, never hardcoded)
gh project view <NUMBER> --owner <OWNER> --format json
gh project field-list <NUMBER> --owner <OWNER> --format json

# List Todo items
gh project item-list <NUMBER> --owner <OWNER> --format json --limit 100 \
  | jq '[.items[] | select(.status == "Todo")]'

# Claim an issue (move to "In progress")
gh project item-edit --project-id <PROJECT_ID> --id <ITEM_ID> \
  --field-id <STATUS_FIELD_ID> --single-select-option-id <IN_PROGRESS_OPTION_ID>

# Post a structured progress comment
gh issue comment <ISSUE_URL> --body "## Agent Update — Claude Code
**Status**: Working
**Action**: Started implementation
---
_Updated by \`Claude Code\` at $(date -u +%Y-%m-%dT%H:%M:%SZ)_"
```

For the full end-to-end story, see [`examples/walkthrough.md`](examples/walkthrough.md).

## Installation

The skill is plain Markdown, so "installing" it means making `SKILL.md` and `references/`
available to your agent. The two most common paths:

```bash
# Claude Code — symlink (stays in sync with repo updates)
git clone https://github.com/aaitor/github-projects-skill.git
ln -s "$(pwd)/github-projects-skill" ~/.claude/skills/github-project-board

# Any other agent — load the content as context
cat SKILL.md references/*.md > github-project-board-context.md
```

Recipes for copy installs, project-scoped skills, generic LLM agents, remote/SSH agents,
and CI are in [`examples/installation.md`](examples/installation.md).

## The Board Model

The skill expects three single-select fields. The [Board Setup Guide](docs/board-setup.md)
creates them from scratch with `gh`.

| Field | Type | Options |
|-------|------|---------|
| **Status** | SingleSelect | In Definition, Todo, In progress, To Review, Ready, Done |
| **Priority** | SingleSelect | Critical, Very High, High, Average, Low, Very Low, Zero |
| **Size** | SingleSelect | XL, L, M, S, XS |

The standard status lifecycle, and who owns each transition:

```
In Definition ──▶ Todo ──▶ In progress ──▶ To Review ──▶ Todo   (feedback)
                                        └─▶ Ready ──────▶ Done   (approved)
   (human)      (human)    (agent)         (agent)       (human)
```

Agents only move items between `Todo → In progress → (To Review | Ready)`. `In Definition`
and `Done` are human-controlled. Add a **README to your board** (board settings, or
`gh project edit --readme`) describing any board-specific rules — agents read it at runtime
and treat it as authoritative.

## Multi-Agent Setup

Run several independent agents on one board by scoping each to a **label**:

1. Create labels: `gh label create research --repo <OWNER>/<REPO>`
2. Label issues: `gh issue edit <ISSUE_URL> --add-label research`
3. Tell each agent its label scope — it only picks up matching issues.

An agent with no label filter processes any issue. See
[`examples/usage-scenarios.md`](examples/usage-scenarios.md) (Scenario 2) for a worked
example.

## Documentation

| Document | What it covers |
|----------|----------------|
| [`SKILL.md`](SKILL.md) | The skill definition — install this. |
| [`docs/board-setup.md`](docs/board-setup.md) | Create a compatible board from scratch with `gh`. |
| [`examples/installation.md`](examples/installation.md) | Install recipes for Claude Code, generic agents, SSH/remote, CI. |
| [`examples/usage-scenarios.md`](examples/usage-scenarios.md) | Natural-language prompts → what the agent does. |
| [`examples/walkthrough.md`](examples/walkthrough.md) | Full end-to-end board walkthrough. |
| [`references/gh-cli-commands.md`](references/gh-cli-commands.md) | Complete `gh` command reference. |
| [`references/graphql-recipes.md`](references/graphql-recipes.md) | Advanced queries (label/assignee filters, bulk ops). |
| [`references/agent-workflow-patterns.md`](references/agent-workflow-patterns.md) | Step-by-step agent playbooks. |

## Repo Structure

```
├── README.md                           # This file
├── SKILL.md                            # The skill definition (install this)
├── docs/
│   └── board-setup.md                  # Create a compatible board from scratch
├── references/
│   ├── gh-cli-commands.md              # Full command reference with examples
│   ├── graphql-recipes.md              # Advanced queries (label/assignee filter, bulk ops)
│   └── agent-workflow-patterns.md      # Step-by-step agent playbooks
├── examples/
│   ├── installation.md                 # Install recipes for multiple agents/setups
│   ├── usage-scenarios.md              # Natural-language usage examples
│   └── walkthrough.md                  # End-to-end board walkthrough
└── LICENSE                             # MIT license
```

## Contributing

Issues and pull requests are welcome. The skill is documentation, so keep changes:

- **Generic** — no org-, user-, or board-specific references in `SKILL.md` or `references/`
  (use `<OWNER>` / `<NUMBER>` placeholders; `octo-org` in examples).
- **ID-free** — never hardcode field or option IDs; resolve them at runtime.
- **Consistent** — follow the Agent Comment Convention and status model already in use.

## License

MIT — see [LICENSE](LICENSE).
