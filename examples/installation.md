# Installation Examples

The skill is plain Markdown — `SKILL.md` plus the `references/` files. "Installing" it
just means making that content available to your agent. Below are concrete recipes for
several agents and setups.

In every case you first need the `gh` CLI, authenticated with the `project` scope:

```bash
gh auth login
gh auth refresh -s project
gh project list --owner <OWNER>   # verify access
```

---

## 1. Claude Code — symlink (recommended)

A symlink keeps the installed skill in sync with the repo as you pull updates.

```bash
git clone https://github.com/aaitor/github-projects-skill.git
ln -s "$(pwd)/github-projects-skill" ~/.claude/skills/github-project-board
```

The skill now appears as `github-project-board`. Invoke it with `/github-project-board`
or let Claude activate it when you mention a board, backlog, or sprint.

## 2. Claude Code — copy (pinned)

Use a copy when you want a fixed version that won't change when the repo updates:

```bash
git clone https://github.com/aaitor/github-projects-skill.git
cp -r github-projects-skill ~/.claude/skills/github-project-board
```

Re-run the `cp` to update.

## 3. Claude Code — project-scoped skill

To ship the skill with a specific repo (so teammates get it automatically), vendor it
under the project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r /path/to/github-projects-skill .claude/skills/github-project-board
git add .claude/skills/github-project-board
git commit -m "chore: add github-project-board skill"
```

## 4. Generic LLM agent — system prompt / context

Any agent that can run shell commands and read context can use this skill. Include the
skill content as system prompt or retrieved context:

```bash
# Concatenate the skill and its references into one context blob
cat SKILL.md references/*.md > /tmp/github-project-board-context.md
```

Feed `/tmp/github-project-board-context.md` to your agent as a system message or
knowledge-base document. The agent needs shell access to run `gh` commands.

## 5. SSH / remote agent (e.g. OpenClaw, a server-side bot)

For an agent running on a remote host, install the skill and `gh` on that host:

```bash
ssh user@your-host

# on the remote host:
gh auth login
gh auth refresh -s project
git clone https://github.com/aaitor/github-projects-skill.git ~/skills/github-project-board
cat ~/skills/github-project-board/SKILL.md   # load into the agent's context
```

If the agent framework supports a skills directory, point it at
`~/skills/github-project-board` the same way as Claude Code.

## 6. CI / headless automation

Use the commands directly in a workflow — no "agent" required. Authenticate with a token
that has project access:

```yaml
# .github/workflows/triage.yml (excerpt)
- name: Move approved issues to Done
  env:
    GH_TOKEN: ${{ secrets.PROJECT_TOKEN }}   # needs 'project' scope
  run: |
    # follow references/gh-cli-commands.md for the exact commands
    gh project item-list 1 --owner <OWNER> --format json --limit 100 | jq '...'
```

See [`references/gh-cli-commands.md`](../references/gh-cli-commands.md) for the full
command set these snippets draw from.

---

## Verifying the install

Point your agent at a board and ask it to read the backlog:

> "Read the Todo items on project board #1 for owner `<OWNER>` and tell me the
> highest-priority one."

A correctly installed agent will run the discovery commands, resolve the Status field,
list the `Todo` items, and report back. If it hardcodes IDs or skips discovery, re-check
that it has the full `SKILL.md` content in context.
