# Note

A Note represents documentation, comments, or any textual record in your CRM. Notes are **standalone records** — they exist independently and are then linked to people, companies, or opportunities via [Note Relations](note-relation.md).

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/notes` | Create a new note |
| GET | `/rest/notes/{id}` | Get a note by ID |
| GET | `/rest/notes` | List notes |
| PATCH | `/rest/notes/{id}` | Update a note |
| DELETE | `/rest/notes/{id}` | Delete a note |
| POST | `/graphql` | Search notes by title |

---

## Create a Note

```
POST /rest/notes
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Note title |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `bodyV2.markdown` | string | Body content as Markdown |
| `bodyV2.blocknote` | string | Body content as blocknote JSON (stringified) |

> **Shortcut:** Instead of providing `bodyV2`, you can send plain text in a `body` field and the API will auto-convert it to the correct `bodyV2` format:
> ```json
> { "title": "Call Summary", "body": "Discussed timelines and next steps." }
> ```

### Request Example — Plain Body (Simplest)

```bash
curl -X POST "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Meeting Notes - Q1 Review",
    "body": "Discussed quarterly targets.\nAction items assigned."
  }'
```

### Request Example — Rich Body (bodyV2)

```bash
curl -X POST "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Meeting Notes - Q1 Review",
    "bodyV2": {
      "markdown": "Discussed quarterly targets.\nAction items assigned.",
      "blocknote": "[{\"id\":\"block-1\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Discussed quarterly targets.\",\"styles\":{}}],\"children\":[]},{\"id\":\"block-2\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Action items assigned.\",\"styles\":{}}],\"children\":[]}]"
    }
  }'
```

---

## Understanding the Blocknote Format

When providing `bodyV2` directly (rather than using the plain `body` shortcut), both `markdown` and `blocknote` are required.

The `blocknote` value is a **JSON-stringified array** of paragraph blocks. Each paragraph is one block:

```json
{
  "id": "unique-uuid",
  "type": "paragraph",
  "props": {
    "textColor": "default",
    "backgroundColor": "default",
    "textAlignment": "left"
  },
  "content": [
    { "type": "text", "text": "Your paragraph text here.", "styles": {} }
  ],
  "children": []
}
```

**Rules:**
- Each block needs a unique UUID-format `id` — UUIDs don't need to be real database UUIDs, just unique strings in that format
- Split multi-paragraph content into one block per paragraph (split on `\n`)
- The entire `blocknote` field value is a JSON string (not an object) — stringify the array before sending

---

## Get a Note

```
GET /rest/notes/{id}?depth=0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Note ID |
| `depth` | number | 0 | `0` = note only, `1` = + linked records |

```bash
curl -G "https://app.usedalil.ai/rest/notes/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

---

## List Notes

```
GET /rest/notes
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor (last record's ID) |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 0 | Related objects depth |

```bash
# Most recent notes
curl -G "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "order_by=createdAt[DescNullsLast]"

# Notes with "Meeting" in the title
curl -G "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=title[ilike]:meeting"
```

---

## Update a Note

Send only the fields you want to change.

```
PATCH /rest/notes/{id}
```

Updatable fields: `title`, `bodyV2`

```bash
curl -X PATCH "https://app.usedalil.ai/rest/notes/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Updated Meeting Notes - Q1 Review"
  }'
```

---

## Delete a Note

```
DELETE /rest/notes/{id}
```

> **Note:** Deleting a note also removes any associated Note Relations. The linked records (person, company, opportunity) are not affected.

---

## Search Notes

Notes are searched by **title only** using GraphQL. Body content is not searchable.

### By Title — GraphQL + REST (Two Steps)

**Step 1: GraphQL search → get IDs**

```bash
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"note\"], limit: 5) { edges { node { recordId objectNameSingular label } } } }",
    "variables": {
      "searchInput": "Meeting Notes"
    }
  }'
```

> **GraphQL caveats:**
> - `searchInput` is a plain `String!` variable
> - `limit` must be hardcoded in the query string
> - Returns IDs only

**Step 2: Fetch full records**

```bash
curl -G "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[recordId1,recordId2]" \
  --data-urlencode "depth=1"
```

---

## Linking Notes to Records

Notes must be explicitly linked to CRM records using the [Note Relation](note-relation.md) endpoint. A single note can be linked to multiple records.

### Workflow: Create a Note and Link It

```bash
# Step 1: Create the note
curl -X POST "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Call Summary", "body": "Great discussion about Q2 plans."}'
# Save the returned note ID

# Step 2: Link the note to a person
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"noteId": "note-uuid", "personId": "person-uuid"}'

# Optionally also link to their company
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"noteId": "note-uuid", "companyId": "company-uuid"}'
```

### Find All Notes for a Person

```bash
curl -G "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=personId[eq]:person-uuid" \
  --data-urlencode "depth=1"
```

The response includes the full note object when `depth >= 1`.

---

## Filter Examples

```bash
# Notes with "meeting" in the title (case-insensitive)
filter=title[ilike]:meeting

# Notes created after a specific date
filter=createdAt[gte]:2024-01-01T00:00:00.000Z

# Notes created in the last 30 days
filter=createdAt[gte]:2024-03-01T00:00:00.000Z
```

---

## Common Gotchas

1. **Notes are standalone.** They are not directly attached to people or companies — a separate Note Relation (`/rest/noteTargets`) is required to create the link.

2. **Body uses blocknote JSON format.** The `bodyV2` field requires both `markdown` and `blocknote`. Use the plain `body` shortcut to avoid constructing blocknote JSON manually.

3. **Blocknote IDs must be unique.** Each block in the blocknote array needs a unique UUID-format `id`. Duplicate IDs cause rendering issues.

4. **Search is title-only.** GraphQL search does not index note body content — only the title.

5. **`id[in]` requires bracket syntax.** Use `id[in]:[uuid1,uuid2]` — brackets are required.

6. **URL-encode filter parameters.** Use `curl -G --data-urlencode "filter=..."`.

---

## See Also

- [Note Relation](note-relation.md) — how to link notes to people, companies, and opportunities
- [Field Types](field-types.md) — full BODY_V2 format reference with blocknote examples
- [Task](task.md) — similar to notes but for actionable items
