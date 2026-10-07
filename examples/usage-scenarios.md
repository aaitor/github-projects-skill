# Usage Scenarios

Real ways to drive the skill once it's installed. Each scenario shows a natural-language
prompt you give the agent and what the agent does under the hood. Swap `<OWNER>` and the
project number for your own board.

---

## Scenario 1: Pick up the next task

**You:**
> "Work the next item on project board #1 for `<OWNER>`."

**The agent:**
1. Discovers the board (metadata, fields, README).
2. Lists `Todo` items, sorts by `Priority`.
3. Moves the top item to `In progress` and comments that it's starting.
4. Does the work, then moves the item to `Ready` with a summary comment.

---

## Scenario 2: Scope an agent to a domain with labels

Run several agents on one board without collisions by giving each a label scope.

**You (to the research agent):**
> "You are the research agent. Only work issues labelled `research` on board #1 for `<OWNER>`."

**You (to the coding agent):**
> "You are the coding agent. Only work `coding`-labelled issues on the same board."

**Each agent** filters the backlog to its label (GraphQL recipe #6 in
[`references/graphql-recipes.md`](../references/graphql-recipes.md)) and ignores
everything else. An unlabelled agent picks up anything.

Set it up:

```bash
gh label create research --repo <OWNER>/<REPO> --description "Research tasks"
gh label create coding   --repo <OWNER>/<REPO> --description "Implementation tasks"
gh issue edit <ISSUE_URL> --add-label research
```

---

## Scenario 3: Triage the whole backlog

**You:**
> "Give me a summary of board #1: how many items in each status, and the top 3 by priority."

**The agent** pulls a full board snapshot (GraphQL recipe #2) and reports counts per
status plus the highest-priority open work — no changes made.

---

## Scenario 4: Break a big issue into sub-issues

**You:**
> "Issue #58 is too big. Decompose it into sub-issues on the board."

**The agent:**
1. Reads issue #58.
2. Creates sub-issues, each referencing the parent, and adds them to the board.
3. Sets `Priority`/`Size` on each.
4. Comments on #58 with a checklist linking the sub-issues.

See Pattern 5 in [`references/agent-workflow-patterns.md`](../references/agent-workflow-patterns.md).

---

## Scenario 5: Ask for clarification mid-task

The agent hits an ambiguous requirement while working an item.

**The agent:**
1. Moves the item to `To Review`.
2. Comments with `**Status**: Blocked` and specific questions.
3. Stops and waits.

**You** answer in the issue comments and move it back to `Todo`. The agent (or the next
one) re-picks it up with your answer in context.

---

## Scenario 6: Hand off between agents

**You:**
> "Finish your part of issue #42 and hand the rest to the QA agent."

**The agent** posts a handoff comment (completed work, remaining work, context, files
touched) and moves the item back to `Todo`. The QA agent picks it up following
Scenario 1. See Pattern 6 in
[`references/agent-workflow-patterns.md`](../references/agent-workflow-patterns.md).

---

## Scenario 7: Submit finished work for approval

**You:**
> "I'm done with #42. Submit it for approval."

**The agent** moves the item to `Ready` and posts a completion summary (changes, PR links,
testing, known limitations). A human then moves it to `Done` or back to `Todo` with
feedback.

---

## What the agent will *not* do

By convention, agents never move items to `In Definition` or `Done` — those are
human-controlled. If you ask an agent to close out an item, it will move it to `Ready`
and leave the final approval to you.
