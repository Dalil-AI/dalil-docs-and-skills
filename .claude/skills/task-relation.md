---
name: task-relation
description: Link tasks to CRM entities (people, companies, opportunities) in Dalil AI — create, read, update, delete, and list taskTarget records that associate a task with one or more target records.
---

# Dalil AI: Task Relation API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/taskTargets`

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/taskTargets` | `taskId` + at least one target ID |
| Get | GET | `/rest/taskTargets/{id}` | `id` (path) |
| List | GET | `/rest/taskTargets` | — |
| Update | PATCH | `/rest/taskTargets/{id}` | `id` (path) |
| Delete | DELETE | `/rest/taskTargets/{id}` | `id` (path) |

No search operation available. List supports `orderBy` only (no `filter`).

## Create Task Relation

### Link task to a person

```json
{
  "taskId": "uuid-of-task",
  "personId": "uuid-of-person"
}
```

### Link task to a company

```json
{
  "taskId": "uuid-of-task",
  "companyId": "uuid-of-company"
}
```

### Link task to an opportunity

```json
{
  "taskId": "uuid-of-task",
  "opportunityId": "uuid-of-opportunity"
}
```

### Fields

- `taskId` (UUID, required) — The task to link
- `personId` (UUID, optional) — Target person
- `companyId` (UUID, optional) — Target company
- `opportunityId` (UUID, optional) — Target opportunity

## Update Task Relation

```
PATCH /rest/taskTargets/{id}?depth=1
```

Updatable fields: `taskId`, `personId`, `companyId`, `opportunityId`

## Get Task Relation

```
GET /rest/taskTargets/{id}?depth=1
```

Default depth is 1 (includes linked task and target record).

## List Task Relations

```
GET /rest/taskTargets?limit=60&order_by=createdAt[DescNullsLast]
```

**Important:** List endpoint supports `orderBy` only. No `filter` parameter.

## Delete Task Relation

```
DELETE /rest/taskTargets/{id}
```

Deletes the link only, not the task itself.

## Common Workflows

### Create a task and link it to a person

```
# Step 1: Create the task
POST /rest/tasks
{ "title": "Follow up call", "status": "TODO", "dueAt": "2024-03-15T09:00:00.000Z" }

# Step 2: Link to person
POST /rest/taskTargets
{ "taskId": "returned-task-id", "personId": "person-uuid" }
```

### Create a task linked to both a person and company

```
# Step 1: Create the task
POST /rest/tasks
{ "title": "Quarterly review", "status": "TODO" }

# Step 2: Link to person
POST /rest/taskTargets
{ "taskId": "task-uuid", "personId": "person-uuid" }

# Step 3: Link to company
POST /rest/taskTargets
{ "taskId": "task-uuid", "companyId": "company-uuid" }
```

## Gotchas

1. **No search or filter on list** — Task relations can only be listed with `orderBy`, not filtered. To find relations for a specific task or record, you must list all and filter client-side, or use depth on the parent record.
2. **Default depth is 1** — Unlike most entities (default 0), task relations default to depth 1.
3. **Deleting a relation does not delete the task** — Only the link is removed.
4. **One target per relation** — Each task relation links a task to ONE target. To link a task to both a person and a company, create two separate relations.
5. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
