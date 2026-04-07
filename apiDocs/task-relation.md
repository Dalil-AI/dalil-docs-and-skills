# Task Relation

A Task Relation (stored as a `taskTarget`) links a [Task](task.md) to a [Person](person.md), [Company](company.md), or [Opportunity](opportunity.md). This is a many-to-many relationship — a single task can be linked to multiple records, and a record can have multiple tasks attached.

> **Two-step process:** First create the task at `/rest/tasks`, then link it using `/rest/taskTargets`.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/taskTargets` | Create a task relation |
| GET | `/rest/taskTargets/{id}` | Get a task relation by ID |
| GET | `/rest/taskTargets` | List task relations |
| PATCH | `/rest/taskTargets/{id}` | Update a task relation |
| DELETE | `/rest/taskTargets/{id}` | Delete a task relation |

Task Relations do not have a search operation. The list endpoint supports `order_by` but **not** `filter`.

---

## Create a Task Relation

```
POST /rest/taskTargets
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `taskId` | UUID (required) | The task to link |
| `personId` | UUID (optional) | Person to link the task to |
| `companyId` | UUID (optional) | Company to link the task to |
| `opportunityId` | UUID (optional) | Opportunity to link the task to |

Provide `taskId` plus **at least one** target ID.

### Link a Task to a Person

```bash
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "taskId": "task-uuid-here",
    "personId": "person-uuid-here"
  }'
```

### Link a Task to a Company

```bash
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "taskId": "task-uuid-here",
    "companyId": "company-uuid-here"
  }'
```

### Link a Task to an Opportunity

```bash
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "taskId": "task-uuid-here",
    "opportunityId": "opportunity-uuid-here"
  }'
```

---

## Get a Task Relation

```
GET /rest/taskTargets/{id}?depth=1
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Task relation ID |
| `depth` | number | **1** | `1` includes the linked task and target record |

> **Default depth is 1** (unlike most entities where it defaults to 0). This is intentional — a task relation is only useful when it includes the linked records.

```bash
curl -G "https://app.usedalil.ai/rest/taskTargets/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## List Task Relations

```
GET /rest/taskTargets
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor |
| `order_by` | string | — | Sort field and direction |

> **Important:** The task relation list endpoint supports `order_by` but **does not support `filter`**. You cannot filter task relations by `personId`, `taskId`, etc. via the list endpoint. To find tasks for a specific record, use `depth=1` on that record (e.g., `GET /rest/people/{id}?depth=1`) or list all relations and filter client-side.

```bash
# List task relations sorted by creation date
curl -G "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "order_by=createdAt[DescNullsLast]" \
  --data-urlencode "limit=20"
```

---

## Update a Task Relation

Change which task or target record this relation points to.

```
PATCH /rest/taskTargets/{id}
```

Updatable fields: `taskId`, `personId`, `companyId`, `opportunityId`

```bash
curl -X PATCH "https://app.usedalil.ai/rest/taskTargets/relation-uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"personId": "different-person-uuid"}'
```

---

## Delete a Task Relation

```
DELETE /rest/taskTargets/{id}
```

This removes the **link** between the task and the target record. The task itself is not deleted.

```bash
curl -X DELETE "https://app.usedalil.ai/rest/taskTargets/relation-uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Common Workflows

### Create a task and link it to a person

```bash
# Step 1: Create the task
curl -X POST "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Follow up call",
    "status": "TODO",
    "dueAt": "2024-03-15T09:00:00.000Z"
  }'
# Save the returned task ID

# Step 2: Link to person
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"taskId": "task-uuid", "personId": "person-uuid"}'
```

### Create a task linked to both a person and their company

```bash
# Step 1: Create the task
curl -X POST "https://app.usedalil.ai/rest/tasks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Quarterly review", "status": "TODO"}'

# Step 2: Link to person
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"taskId": "task-uuid", "personId": "person-uuid"}'

# Step 3: Link to company (separate relation)
curl -X POST "https://app.usedalil.ai/rest/taskTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"taskId": "task-uuid", "companyId": "company-uuid"}'
```

### Find tasks for a person (via their record)

Because the list endpoint does not support filtering, the best way to get tasks for a specific person is to fetch the person with `depth=1`, which includes linked tasks in the response:

```bash
curl -G "https://app.usedalil.ai/rest/people/person-uuid" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

---

## Common Gotchas

1. **No filter on the list endpoint.** Unlike Note Relations, the task relation list endpoint does not support `filter`. You cannot query "all task relations for person X" directly. Use `depth=1` on the parent record instead.

2. **One target per relation.** Each task relation links a task to exactly ONE target. To link a task to both a person and a company, create two separate task relations.

3. **Default depth is 1.** Unlike most entities (which default to `depth=0`), task relations default to `depth=1`.

4. **Deleting a relation does not delete the task.** Only the link is removed.

5. **URL-encode filter parameters.** Even though filtering isn't supported here, this applies when using filters on other endpoints.

---

## See Also

- [Task](task.md) — creating and managing tasks
- [Note Relation](note-relation.md) — equivalent pattern for notes (supports filtering)
