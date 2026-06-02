---
name: note
description: Manage notes in Dalil AI CRM — create, read, update, delete, search, and list note records with rich text body (markdown/blocknote). Use note-relation to attach notes to people, companies, or opportunities.
---

# Dalil AI: Note API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/notes`

**GraphQL search (`POST /graphql`):**
- `searchInput` is a plain `String!` variable — do NOT pass as an object `{query: "...", includedObjectNameSingulars: [...]}` (causes type error)
- `limit` must be hardcoded in the query string — do NOT pass as a `$limit` variable (causes type error)
- Returns IDs only — always follow up with REST to fetch full records

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/notes` | `title` |
| Get | GET | `/rest/notes/{id}` | `id` (path) |
| List | GET | `/rest/notes` | — |
| Update | PATCH | `/rest/notes/{id}` | `id` (path) |
| Delete | DELETE | `/rest/notes/{id}` | `id` (path) |
| Search by title | POST | `/graphql` | `searchInput` |

## Create Note

### Request Body

```json
{
  "title": "Meeting Notes - Q1 Review",
  "bodyV2": {
    "markdown": "Discussed quarterly targets.\nAction items assigned.",
    "blocknote": "[{\"id\":\"unique-uuid-1\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Discussed quarterly targets.\",\"styles\":{}}],\"children\":[]},{\"id\":\"unique-uuid-2\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Action items assigned.\",\"styles\":{}}],\"children\":[]}]"
  }
}
```

### Required Fields

- `title` (string) — Note title

### Optional Fields

- `bodyV2.markdown` (string) — Body content as markdown
- `bodyV2.blocknote` (string) — Body content as blocknote JSON

## Blocknote Format

Each paragraph is a block:
```json
{
  "id": "unique-uuid",
  "type": "paragraph",
  "props": {
    "textColor": "default",
    "backgroundColor": "default",
    "textAlignment": "left"
  },
  "content": [{ "type": "text", "text": "Your text here.", "styles": {} }],
  "children": []
}
```

Multiple paragraphs: split text by newlines, one block per line. The `blocknote` field is a JSON string of an array of these blocks.

## Update Note

```
PATCH /rest/notes/{id}?depth=0
```

Updatable fields: `title`, `bodyV2`

## Get Note

```
GET /rest/notes/{id}?depth=0
```

## List Notes

```
GET /rest/notes?limit=60&order_by=createdAt[DescNullsLast]&filter=...&depth=0
```

## Delete Note

```
DELETE /rest/notes/{id}
```

## Search Pattern

Search by title (GraphQL + REST):

Step 1 — GraphQL:
```json
{
  "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"note\"], limit: 5) { edges { node { recordId objectNameSingular label imageUrl } } } }",
  "variables": { "searchInput": "Meeting Notes" }
}
```
POST to `https://app.usedalil.ai/graphql`

Step 2 — Fetch:
```
GET /rest/notes?filter=id[in]:[recordId1,recordId2]&depth=1
```

## Linking Notes to Records

Notes are standalone. To attach a note to a person, company, or opportunity, create a **Note Relation**:

```
POST /rest/noteTargets
```
```json
{
  "noteId": "uuid-of-note",
  "personId": "uuid-of-person"
}
```

Or use `companyId` or `opportunityId` instead of `personId`.

## Filter Examples

```
# Notes with title containing "Meeting"
filter=title[like]:Meeting

# Notes created after a date
filter=createdAt[gte]:2024-01-01T00:00:00.000Z
```

## Gotchas

1. **Body uses blocknote JSON format** — The `bodyV2` field requires both `markdown` and `blocknote` (JSON string). When sending via the simplified API, you can send plain `body` text and it auto-converts.
2. **Notes are standalone** — They are not directly linked to people/companies. Use Note Relations (`/rest/noteTargets`) to create associations.
3. **Blocknote IDs must be unique** — Each block in the blocknote JSON array needs a unique UUID-format `id`.
4. **Search is title-only** — No body content search is available.
5. **GraphQL search returns IDs only** — Follow up with `GET /rest/notes?filter=id[in]:[id1,id2]` to fetch full records.
6. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
