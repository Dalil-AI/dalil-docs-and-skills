# Note Relation

A Note Relation (stored as a `noteTarget`) links a [Note](note.md) to a [Person](person.md), [Company](company.md), or [Opportunity](opportunity.md). This is a many-to-many relationship — a single note can be linked to multiple records, and a record can have multiple notes attached.

> **Two-step process:** First create the note at `/rest/notes`, then link it using `/rest/noteTargets`.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/noteTargets` | Create a note relation |
| GET | `/rest/noteTargets/{id}` | Get a note relation by ID |
| GET | `/rest/noteTargets` | List note relations |
| PATCH | `/rest/noteTargets/{id}` | Update a note relation |
| DELETE | `/rest/noteTargets/{id}` | Delete a note relation |

Note Relations do not have a search operation.

---

## Create a Note Relation

```
POST /rest/noteTargets
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `noteId` | UUID (required) | The note to link |
| `personId` | UUID (optional) | Person to link the note to |
| `companyId` | UUID (optional) | Company to link the note to |
| `opportunityId` | UUID (optional) | Opportunity to link the note to |

Provide `noteId` plus **at least one** target ID.

### Link a Note to a Person

```bash
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "noteId": "note-uuid-here",
    "personId": "person-uuid-here"
  }'
```

### Link a Note to a Company

```bash
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "noteId": "note-uuid-here",
    "companyId": "company-uuid-here"
  }'
```

### Link a Note to an Opportunity

```bash
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "noteId": "note-uuid-here",
    "opportunityId": "opportunity-uuid-here"
  }'
```

---

## Get a Note Relation

```
GET /rest/noteTargets/{id}?depth=1
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Note relation ID |
| `depth` | number | **1** | `1` includes the linked note and target record |

> **Default depth is 1** (unlike most entities where it defaults to 0). This is intentional — a note relation is only useful when it includes the linked records.

```bash
curl -G "https://app.usedalil.ai/rest/noteTargets/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## List Note Relations

```
GET /rest/noteTargets
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 1 | Related objects depth |

```bash
# All note relations for a specific person
curl -G "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=personId[eq]:person-uuid-here" \
  --data-urlencode "depth=1"

# All note relations for a company
curl -G "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=companyId[eq]:company-uuid-here" \
  --data-urlencode "depth=1"

# Relations where a person is linked (any person)
curl -G "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=personId[is]:NOT_NULL" \
  --data-urlencode "depth=1"
```

---

## Update a Note Relation

Change which note or target record this relation points to.

```
PATCH /rest/noteTargets/{id}
```

Updatable fields: `noteId`, `personId`, `companyId`, `opportunityId`

```bash
curl -X PATCH "https://app.usedalil.ai/rest/noteTargets/relation-uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"personId": "different-person-uuid"}'
```

---

## Delete a Note Relation

```
DELETE /rest/noteTargets/{id}
```

This removes the **link** between the note and the target record. The note itself is not deleted.

```bash
curl -X DELETE "https://app.usedalil.ai/rest/noteTargets/relation-uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Common Workflows

### Create a note and link it to both a person and their company

```bash
# Step 1: Create the note
NOTE_RESPONSE=$(curl -s -X POST "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Discovery Call Notes", "body": "Discussed requirements and timeline."}')
# Extract noteId from response...

# Step 2: Link to person
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"noteId": "note-uuid", "personId": "person-uuid"}'

# Step 3: Link to company (separate relation record)
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"noteId": "note-uuid", "companyId": "company-uuid"}'
```

### Get all notes for a person (with note content)

```bash
curl -G "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=personId[eq]:person-uuid" \
  --data-urlencode "depth=1" \
  --data-urlencode "order_by=createdAt[DescNullsLast]"
```

The response at `depth=1` includes the full note object and the linked record object in each relation.

---

## Filter Examples

```bash
# Relations for a specific note
filter=noteId[eq]:note-uuid

# Relations linking to a person
filter=personId[eq]:person-uuid

# Relations linking to a company
filter=companyId[eq]:company-uuid

# Relations where a person is linked (any)
filter=personId[is]:NOT_NULL

# Relations where a company is linked (any)
filter=companyId[is]:NOT_NULL
```

---

## Common Gotchas

1. **One target per relation.** Each note relation links a note to exactly ONE target. To link a note to both a person and a company, create two separate note relations.

2. **Default depth is 1.** Unlike most entities (which default to `depth=0`), note relations default to `depth=1` so the linked records are included automatically.

3. **Deleting a relation does not delete the note.** Only the link is removed.

4. **No search operation.** Note relations can only be filtered via the list endpoint.

5. **URL-encode filter parameters.** Use `curl -G --data-urlencode "filter=..."`.

---

## See Also

- [Note](note.md) — creating and managing notes
- [Task Relation](task-relation.md) — equivalent pattern for tasks
