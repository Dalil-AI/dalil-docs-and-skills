---
name: pipeline
description: Manage dynamic CRM pipelines in Dalil AI — discover available pipelines, retrieve their fields and stages, and create/update/delete pipeline records. Endpoints are dynamic (namePlural varies per pipeline); always discover before operating.
---

# Dalil AI: Pipeline API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/{namePlural}` (dynamic per pipeline)

**GraphQL search (`POST /graphql`):**
- Uses `searchPipeline` query (NOT `search`) — requires both `searchInput: String!` and `nameSingular: String!` variables
- `searchInput` is a plain `String!` variable — do NOT pass as an object (causes type error)
- `limit` must be hardcoded in the query string — do NOT pass as a `$limit` variable (causes type error)
- Returns IDs only — always follow up with REST to fetch full records

## Important: Dynamic Endpoints

Pipeline endpoints are NOT fixed. Each pipeline has its own `namePlural` (e.g., `startupFundraisings`, `salesPipelines`). You MUST discover available pipelines first.

## Discovery (Required First Step)

### List all pipelines
```
GET /rest/metadata/pipelines
```

### Get pipeline fields
```
GET /rest/metadata/pipelines/{pipelineId}
```

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/{namePlural}` | `recordId` (parent) |
| Get | GET | `/rest/{namePlural}/{id}` | `id` (path) |
| List | GET | `/rest/{namePlural}` | — |
| Update | PATCH | `/rest/{namePlural}/{id}` | `id` (path) |
| Delete | DELETE | `/rest/{namePlural}/{id}` | `id` (path) |
| Search | POST | `/graphql` | `searchInput`, `nameSingular` |

## Create Pipeline Record

### Request Body

```json
{
  "recordId": "uuid-of-parent-company-or-person",
  "stage": "SERIES_A",
  "fundingAmount": 5000000000000,
  "targetCloseDate": "2024-09-01T00:00:00.000Z"
}
```

### Required Fields

- `recordId` (UUID) — Parent record ID (Company or Person, depends on pipeline type)

### Custom Properties

All other fields are pipeline-specific. Discover them via:
```
GET /rest/metadata/pipelines/{pipelineId}
```

Field types follow the same rules as other entities:
- SELECT fields: UPPER_SNAKE_CASE values
- CURRENCY fields: amounts in micros
- DATE_TIME fields: ISO 8601 format
- BOOLEAN fields: true/false

## Update Pipeline Record

```
PATCH /rest/{namePlural}/{id}?depth=1
```

Send only custom properties to update.

## Get Pipeline Record

```
GET /rest/{namePlural}/{id}?depth=1
```

Default depth is 1 (includes parent record).

## List Pipeline Records

```
GET /rest/{namePlural}?limit=60&order_by=createdAt[DescNullsLast]&filter=...&depth=1
```

Default depth is 1.

## Delete Pipeline Record

```
DELETE /rest/{namePlural}/{id}
```

## Search Pattern

Pipeline search uses a DIFFERENT GraphQL query: `searchPipeline` (not `search`).

Step 1 — GraphQL:
```json
{
  "query": "query SearchPipeline($searchInput: String!, $nameSingular: String!) { searchPipeline(searchInput: $searchInput, nameSingular: $nameSingular, limit: 5) { recordId } }",
  "variables": {
    "searchInput": "Acme",
    "nameSingular": "startupFundraising"
  }
}
```
POST to `https://app.usedalil.ai/graphql`

Step 2 — Fetch:
```
GET /rest/{namePlural}?filter=id[in]:[recordId1,recordId2]&depth=1
```

## Full Workflow

```
# 1. Discover pipelines
GET /rest/metadata/pipelines

# 2. Get fields for a specific pipeline
GET /rest/metadata/pipelines/{pipelineId}
# Response includes: nameSingular, namePlural, fields[], parentRecordType

# 3. Get parent records (companies or people, based on pipeline)
GET /rest/companies?limit=100
# or
GET /rest/people?limit=100

# 4. Create a record
POST /rest/{namePlural}
{ "recordId": "parent-uuid", ...customFields }

# 5. List records
GET /rest/{namePlural}?depth=1

# 6. Update a record
PATCH /rest/{namePlural}/{id}
{ ...fieldsToUpdate }
```

## Filter Examples

```
# Records at a specific stage
filter=stage[eq]:FUNDED

# Records created after a date
filter=createdAt[gte]:2024-01-01T00:00:00.000Z
```

## Gotchas

1. **Endpoints are dynamic** — The URL path comes from `pipelineMetadata.namePlural`. There is no fixed `/rest/pipelines` endpoint for records.
2. **Must discover pipelines first** — Call `GET /rest/metadata/pipelines` before any pipeline operations.
3. **Uses different GraphQL query** — Pipeline search uses `searchPipeline`, not `search`.
4. **GraphQL requires `nameSingular`** — The search query needs the pipeline's singular name, not plural.
5. **Parent record is required on create** — `recordId` links the pipeline record to a Company or Person.
6. **Default depth is 1** — Pipeline records default to depth 1 (includes parent record).
7. **Custom fields vary** — Each pipeline has unique fields. Always check metadata before creating/updating.
8. **SELECT values are UPPER_SNAKE_CASE** — Same as other entities: `FUNDED`, `SERIES_A`, `DEMO`, etc.
9. **Currency amounts in micros** — Same rule: multiply by 1,000,000.
10. **GraphQL `limit` must be hardcoded, not a variable** — Passing `$limit: Int` as a variable causes a type error. Inline it directly: `limit: 5`.
11. **GraphQL search returns IDs only** — Follow up with `GET /rest/{namePlural}?filter=id[in]:[id1,id2]` to fetch full records.
12. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
