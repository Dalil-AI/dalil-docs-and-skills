# Task

A Task represents an actionable item or assignment in your CRM — follow-ups, to-dos, reminders, or any work to be done. Tasks are **standalone records** that must be explicitly linked to people, companies, or opportunities via [Task Relations](task-relation.md).

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/tasks` | Create a new task |
| GET | `/rest/tasks/{id}` | Get a task by ID |
| GET | `/rest/tasks` | List tasks |
| PATCH | `/rest/tasks/{id}` | Update a task |
| DELETE | `/rest/tasks/{id}` | Delete a task |
| POST | `/graphql` | Search tasks by title |

---

## Create a Task

```
POST /rest/tasks
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Task title |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `bodyV2.markdown` | string | Task description as Markdown |
| `bodyV2.blocknote` | string | Task description as blocknote JSON (stringified) |
| `status` | string | Task status: `TODO`, `IN_PROGRESS`, or `DONE` |
| `dueAt` | string | Due date (ISO 8601) |
| `assigneeId` | UUID | Assigned workspace member ID |

> **Body shortcut:** You can send plain text in a `body` field (instead of `bodyV2`) and the API will auto-convert it.

### Request Example

```bash
curl -X POST "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Follow up with client",
    "body": "Call to discuss the proposal. Send updated pricing afterward.",
    "status": "TODO",
    "dueAt": "2024-03-15T09:00:00.000Z",
    "assigneeId": "member-uuid-here"
  }'
```

---

## Get a Task

```
GET /rest/tasks/{id}?depth=0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Task ID |
| `depth` | number | 0 | `0` = task only, `1` = + linked records |

```bash
curl -G "https://app.usedalil.ai/rest/tasks/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

---

## List Tasks

```
GET /rest/tasks
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor (last record's ID) |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 0 | Related objects depth |

```bash
# All incomplete tasks, soonest due date first
curl -G "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=status[neq]:DONE" \
  --data-urlencode "order_by=dueAt[AscNullsLast]"

# Tasks assigned to a specific member
curl -G "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=assigneeId[eq]:member-uuid"

# Overdue tasks (not done, past due date)
curl -G "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=status[neq]:DONE,dueAt[lt]:2024-03-01T00:00:00.000Z"
```

---

## Update a Task

Send only the fields you want to change.

```
PATCH /rest/tasks/{id}
```

Updatable fields: `title`, `bodyV2`, `status`, `dueAt`, `assigneeId`

```bash
# Mark a task as done
curl -X PATCH "https://app.usedalil.ai/rest/tasks/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "DONE"}'
```

---

## Delete a Task

```
DELETE /rest/tasks/{id}
```

> **Note:** Deleting a task also removes any associated Task Relations. The linked records (person, company, opportunity) are not affected.

---

## Search Tasks

Tasks are searched by **title only** using GraphQL. Body content is not searchable.

### By Title — GraphQL + REST (Two Steps)

**Step 1: GraphQL search → get IDs**

```bash
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"task\"], limit: 5) { edges { node { recordId objectNameSingular label } } } }",
    "variables": {
      "searchInput": "Follow up"
    }
  }'
```

> **GraphQL caveats:**
> - `searchInput` is a plain `String!` variable
> - `limit` must be hardcoded in the query string
> - Returns IDs only

**Step 2: Fetch full records**

```bash
curl -G "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[recordId1,recordId2]" \
  --data-urlencode "depth=1"
```

---

## Linking Tasks to Records

Tasks must be explicitly linked to CRM records using the [Task Relation](task-relation.md) endpoint. A single task can be linked to multiple records.

### Workflow: Create a Task and Link It

```bash
# Step 1: Create the task
curl -X POST "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Quarterly review call",
    "status": "TODO",
    "dueAt": "2024-04-01T10:00:00.000Z"
  }'
# Save the returned task ID

# Step 2: Link to a person
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"taskId": "task-uuid", "personId": "person-uuid"}'

# Step 3: Also link to their company (separate relation)
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"taskId": "task-uuid", "companyId": "company-uuid"}'
```

### Find All Tasks for a Person

The `/rest/taskTargets` list endpoint does **not** support `filter`. Use `depth=1` on the person record instead — linked tasks are returned inline:

```bash
curl -G "https://app.usedalil.ai/rest/people/person-uuid" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

Alternatively, fetch tasks at `depth=1` using a title or status filter — each task includes a `taskTargets[]` array containing the linked `personId`, `companyId`, and `opportunityId`.

---

## Filter Examples

```bash
# Incomplete tasks
filter=status[neq]:DONE

# Tasks with TODO status
filter=status[eq]:TODO

# Tasks due before a specific date
filter=dueAt[lt]:2024-04-01T00:00:00.000Z

# Tasks assigned to a specific member
filter=assigneeId[eq]:member-uuid

# Overdue incomplete tasks
filter=status[neq]:DONE,dueAt[lt]:2024-03-01T00:00:00.000Z

# Tasks with no due date set
filter=dueAt[is]:NULL
```

---

## Common Gotchas

1. **Tasks are standalone.** They are not directly attached to people or companies — a separate Task Relation (`/rest/taskTargets`) is required to create the link.

2. **Status values are UPPER_SNAKE_CASE.** Use `TODO`, `IN_PROGRESS`, `DONE` — not `"To Do"` or `"done"`.

3. **Due date must be ISO 8601.** Use `"2024-03-15T09:00:00.000Z"` format.

4. **Search is title-only.** GraphQL search does not index task body content.

5. **Task Relations list does not support filtering.** Unlike most list endpoints, `/rest/taskTargets` only supports `order_by` — not `filter`. To find tasks for a specific person or company, use `depth=1` on the parent record or list all relations client-side.

6. **`id[in]` requires bracket syntax.** Use `id[in]:[uuid1,uuid2]`.

7. **URL-encode filter parameters.** Use `curl -G --data-urlencode "filter=..."`.

---

## See Also

- [Task Relation](task-relation.md) — how to link tasks to people, companies, and opportunities
- [Field Types](field-types.md) — BODY_V2 format reference
- [Note](note.md) — similar to tasks but for documentation rather than action items
