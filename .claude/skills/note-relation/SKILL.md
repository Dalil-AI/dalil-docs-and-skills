---
name: note-relation
description: Link notes to CRM entities (people, companies, opportunities) in Dalil AI — create, read, update, delete, and list noteTarget records that associate a note with one or more target records.
---

# Dalil AI: Note Relation API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/noteTargets`

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/noteTargets` | `noteId` + at least one target ID |
| Get | GET | `/rest/noteTargets/{id}` | `id` (path) |
| List | GET | `/rest/noteTargets` | — |
| Update | PATCH | `/rest/noteTargets/{id}` | `id` (path) |
| Delete | DELETE | `/rest/noteTargets/{id}` | `id` (path) |

No search operation available.

## Create Note Relation

### Link note to a person

```json
{
  "noteId": "uuid-of-note",
  "personId": "uuid-of-person"
}
```

### Link note to a company

```json
{
  "noteId": "uuid-of-note",
  "companyId": "uuid-of-company"
}
```

### Link note to an opportunity

```json
{
  "noteId": "uuid-of-note",
  "opportunityId": "uuid-of-opportunity"
}
```

### Fields

- `noteId` (UUID, required) — The note to link
- `personId` (UUID, optional) — Target person
- `companyId` (UUID, optional) — Target company
- `opportunityId` (UUID, optional) — Target opportunity

At least one target ID should be provided alongside `noteId`.

## Update Note Relation

```
PATCH /rest/noteTargets/{id}?depth=1
```

Updatable fields: `noteId`, `personId`, `companyId`, `opportunityId`

## Get Note Relation

```
GET /rest/noteTargets/{id}?depth=1
```

Default depth is 1 (includes linked note and target record).

## List Note Relations

```
GET /rest/noteTargets?limit=60&order_by=createdAt[DescNullsLast]&filter=...&depth=1
```

## Delete Note Relation

```
DELETE /rest/noteTargets/{id}
```

Deletes the link only, not the note itself.

## Common Workflows

### Create a note and link it to a person

```
# Step 1: Create the note
POST /rest/notes
{ "title": "Call Summary", "body": "Discussed project timeline." }

# Step 2: Link to person
POST /rest/noteTargets
{ "noteId": "returned-note-id", "personId": "person-uuid" }
```

### Find all notes for a person

```
GET /rest/noteTargets?filter=personId[eq]:person-uuid&depth=1
```

The response includes the full note object when depth >= 1.

### Find all notes for a company

```
GET /rest/noteTargets?filter=companyId[eq]:company-uuid&depth=1
```

## Filter Examples

```
# Relations for a specific note
filter=noteId[eq]:note-uuid

# Relations linking to a person
filter=personId[eq]:person-uuid

# Relations where a person is linked (any person)
filter=personId[is]:NOT_NULL

# Relations where company is linked
filter=companyId[is]:NOT_NULL
```

## Gotchas

1. **No search operation** — Note relations can only be filtered via the list endpoint.
2. **Default depth is 1** — Unlike most entities (default 0), note relations default to depth 1 to include linked records.
3. **Deleting a relation does not delete the note** — Only the link is removed.
4. **One target per relation** — Each note relation links a note to ONE target. To link a note to both a person and a company, create two separate note relations.
5. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
