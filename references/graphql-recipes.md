# GraphQL Recipes for GitHub Projects v2

Advanced queries for operations the `gh` CLI doesn't cover directly. All use `gh api graphql`.

## 1. Filter Board Items by Assignee

The `item-list` CLI command doesn't support filtering by assignee. Use GraphQL instead.

```bash
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
                assignees(first: 10) {
                  nodes { login }
                }
              }
            }
          }
        }
      }
    }
  }
' -f owner="<OWNER>" -F number=<NUMBER> \
  --jq '.data.user.projectV2.items.nodes[]
    | select(.content.assignees.nodes[]?.login == "<USERNAME>")
    | {id, title: .content.title, number: .content.number, status: .fieldValueByName.name}'
```

## 2. Get Items with All Field Values (Full Board Snapshot)

Retrieves every item with all its field values — useful for a complete board state overview.

```bash
gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    user(login: $owner) {
      projectV2(number: $number) {
        items(first: 100) {
          nodes {
            id
            content {
              ... on Issue {
                number
                title
                url
                state
              }
              ... on DraftIssue {
                title
                body
              }
            }
            fieldValues(first: 20) {
              nodes {
                ... on ProjectV2ItemFieldSingleSelectValue {
                  field { ... on ProjectV2SingleSelectField { name } }
                  name
                }
                ... on ProjectV2ItemFieldTextValue {
                  field { ... on ProjectV2Field { name } }
                  text
                }
                ... on ProjectV2ItemFieldNumberValue {
                  field { ... on ProjectV2Field { name } }
                  number
                }
                ... on ProjectV2ItemFieldDateValue {
                  field { ... on ProjectV2Field { name } }
                  date
                }
                ... on ProjectV2ItemFieldIterationValue {
                  field { ... on ProjectV2IterationField { name } }
                  title
                  startDate
                  duration
                }
              }
            }
          }
        }
      }
    }
  }
' -f owner="<OWNER>" -F number=<NUMBER>
```

## 3. Filter by Current Iteration

Find items assigned to the current iteration.

```bash
gh api graphql -f query='
  query($owner: String!, $number: Int!) {
    user(login: $owner) {
      projectV2(number: $number) {
        items(first: 100) {
          nodes {
            id
            content {
              ... on Issue { number title url }
            }
            fieldValueByName(name: "Status") {
              ... on ProjectV2ItemFieldSingleSelectValue { name }
            }
            fieldValueByName(name: "Iteration") {
              ... on ProjectV2ItemFieldIterationValue { title startDate duration }
            }
          }
        }
      }
    }
  }
' -f owner="<OWNER>" -F number=<NUMBER> \
  --jq '.data.user.projectV2.items.nodes[]
    | select(.fieldValueByName != null)
    | {id, title: .content.title, status: (.fieldValues // {} | .nodes[]? | select(.field.name == "Status") | .name), iteration: (.fieldValues // {} | .nodes[]? | select(.field.name == "Iteration") | .title)}'
```

## 4. Look Up Project Item ID from Issue URL

When you have an issue URL but need the board item ID for `item-edit`.

```bash
gh api graphql -f query='
  query($owner: String!, $number: Int!, $issueNumber: Int!, $repo: String!) {
    user(login: $owner) {
      projectV2(number: $number) {
        items(first: 100) {
          nodes {
            id
            content {
              ... on Issue {
                number
                repository { name }
              }
            }
          }
        }
      }
    }
  }
' -f owner="<OWNER>" -F number=<NUMBER> -F issueNumber=<ISSUE_NUM> -f repo="<REPO>" \
  --jq '.data.user.projectV2.items.nodes[]
    | select(.content.number == <ISSUE_NUM> and .content.repository.name == "<REPO>")
    | .id'
```

Simpler alternative if you know the issue node ID:

```bash
gh api graphql -f query='
  query($projectId: ID!, $contentId: ID!) {
    node(id: $projectId) {
      ... on ProjectV2 {
        items(first: 100) {
          nodes {
            id
            content { ... on Issue { number url } }
          }
        }
      }
    }
  }
' -f projectId="<PROJECT_ID>" -f contentId="<ISSUE_NODE_ID>" \
  --jq '.data.node.items.nodes[] | select(.content.url == "<ISSUE_URL>") | .id'
```

## 5. Bulk Status Update

Move multiple items to a new status in a single API call using mutations.

```bash
# First, build a mutation with aliases for each item
# Example for 3 items:
gh api graphql -f query='
  mutation($projectId: ID!, $statusFieldId: ID!, $optionId: String!,
           $item1: ID!, $item2: ID!, $item3: ID!) {
    a: updateProjectV2ItemFieldValue(input: {
      projectId: $projectId, itemId: $item1,
      fieldId: $statusFieldId,
      value: { singleSelectOptionId: $optionId }
    }) { projectV2Item { id } }
    b: updateProjectV2ItemFieldValue(input: {
      projectId: $projectId, itemId: $item2,
      fieldId: $statusFieldId,
      value: { singleSelectOptionId: $optionId }
    }) { projectV2Item { id } }
    c: updateProjectV2ItemFieldValue(input: {
      projectId: $projectId, itemId: $item3,
      fieldId: $statusFieldId,
      value: { singleSelectOptionId: $optionId }
    }) { projectV2Item { id } }
  }
' \
  -f projectId="<PROJECT_ID>" \
  -f statusFieldId="<STATUS_FIELD_ID>" \
  -f optionId="<TARGET_OPTION_ID>" \
  -f item1="<ITEM_ID_1>" \
  -f item2="<ITEM_ID_2>" \
  -f item3="<ITEM_ID_3>"
```

For more than a few items, generate the mutation dynamically in a script.

## Notes

- **Organization-owned projects**: Replace `user(login: $owner)` with `organization(login: $owner)` in all queries.
- **Pagination**: All examples use `first: 100`. For boards with 100+ items, implement cursor-based pagination using `pageInfo { hasNextPage endCursor }` and the `after` parameter.
- **Rate limits**: GraphQL API has a point-based rate limit (5,000 points/hour). Each query costs 1 point base plus complexity. Bulk mutations are more efficient than individual API calls.
