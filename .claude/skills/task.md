---
name: task
description: Manage tasks/to-dos in Dalil AI CRM — create, read, update, delete, search, and list task records including title, due date, status, and assignee. Use task-relation to attach tasks to people, companies, or opportunities.
---

# Dalil AI: Task API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/tasks`

**GraphQL search (`POST /graphql`):**
- `searchInput` is a plain `String!` variable — do NOT pass as an object `{query: "...", includedObjectNameSingulars: [...]}` (causes type error)
- `limit` must be hardcoded in the query string — do NOT pass as a `$limit` variable (causes type error)
- Returns IDs only — always follow up with REST to fetch full records

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/tasks` | `title` |
| Get | GET | `/rest/tasks/{id}` | `id` (path) |
| List | GET | `/rest/tasks` | — |
| Update | PATCH | `/rest/tasks/{id}` | `id` (path) |
| Delete | DELETE | `/rest/tasks/{id}` | `id` (path) |
| Search by title | POST | `/graphql` | `searchInput` |

## Create Task

### Request Body

```json
{
  "title": "Follow up with client",
  "bodyV2": {
    "markdown": "Call to discuss the proposal.\nSend updated pricing.",
    "blocknote": "[{\"id\":\"block-1\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Call to discuss the proposal.\",\"styles\":{}}],\"children\":[]},{\"id\":\"block-2\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Send updated pricing.\",\"styles\":{}}],\"children\":[]}]"
  },
  "status": "TODO",
  "dueAt": "2024-03-15T09:00:00.000Z",
  "assigneeId": "uuid-of-workspace-member"
}
```

### Required Fields

- `title` (string) — Task title

### Optional Fields

- `bodyV2.markdown` (string) — Body content as markdown
- `bodyV2.blocknote` (string) — Body content as blocknote JSON
- `status` (string) — `TODO`, `IN_PROGRESS`, or `DONE`
- `dueAt` (string) — Due date in ISO 8601 format
- `assigneeId` (UUID) — Assigned workspace member

## Update Task

```
PATCH /rest/tasks/{id}?depth=0
```

Updatable fields: `title`, `bodyV2`, `status`, `dueAt`, `assigneeId`

## Get Task

```
GET /rest/tasks/{id}?depth=0
```

## List Tasks

```
GET /rest/tasks?limit=60&order_by=dueAt[AscNullsLast]&filter=...&depth=0
```

## Delete Task

```
DELETE /rest/tasks/{id}
```

## Search Pattern

Search by title (GraphQL + REST):

Step 1 — GraphQL:
```json
{
  "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"task\"], limit: 5) { edges { node { recordId objectNameSingular label imageUrl } } } }",
  "variables": { "searchInput": "Follow up" }
}
```
POST to `https://app.usedalil.ai/graphql`

Step 2 — Fetch:
```
GET /rest/tasks?filter=id[in]:[recordId1,recordId2]&depth=1
```

## Linking Tasks to Records

Tasks are standalone. To attach a task to a person, company, or opportunity, create a **Task Relation**:

```
POST /rest/taskTargets
```
```json
{
  "taskId": "uuid-of-task",
  "personId": "uuid-of-person"
}
```

Or use `companyId` or `opportunityId` instead of `personId`.

## Filter Examples

```
# Incomplete tasks
filter=status[neq]:DONE

# Tasks due before a date
filter=dueAt[lt]:2024-04-01T00:00:00.000Z

# Tasks with TODO status
filter=status[eq]:TODO

# Tasks assigned to a specific member
filter=assigneeId[eq]:member-uuid

# Overdue incomplete tasks
filter=status[neq]:DONE,dueAt[lt]:2024-03-01T00:00:00.000Z
```

## Gotchas

1. **Status values are UPPER_SNAKE_CASE** — Use `TODO`, `IN_PROGRESS`, `DONE`.
2. **Body uses blocknote JSON format** — The `bodyV2` field requires both `markdown` and `blocknote`. When using the simplified API, send plain `body` text for auto-conversion.
3. **Tasks are standalone** — Not directly linked to records. Use Task Relations (`/rest/taskTargets`) to associate with people/companies/opportunities.
4. **Due date is ISO 8601** — Use format `2024-03-15T09:00:00.000Z`.
5. **Search is title-only** — No body content search is available.
6. **GraphQL search returns IDs only** — Follow up with `GET /rest/tasks?filter=id[in]:[id1,id2]` to fetch full records.
7. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
8. **REST response wraps results under a named key** — `.data` is NOT a plain array. It's an object keyed by the resource name: `{ "data": { "tasks": [...] }, "pageInfo": {...}, "totalCount": N }`. Iterate with `.data.tasks[]`, not `.data[]`.
9. **`depth=1` on tasks includes `taskTargets` inline** — Each task in the response will contain a `taskTargets[]` array with `personId`, `companyId`, and `opportunityId`. This is more efficient than querying `/rest/taskTargets` separately when you need related record IDs for a batch of tasks.
