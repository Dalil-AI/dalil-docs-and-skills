# Pipeline

A Pipeline represents a custom workflow or process in Dalil AI — such as a sales pipeline, recruitment process, or fundraising tracker. Pipeline records are **dynamic**: each pipeline has its own set of fields and its own REST endpoint based on the pipeline's `namePlural`.

## Key Concept: Dynamic Endpoints

Unlike other entities with fixed endpoints (e.g., `/rest/people`), pipeline endpoints are determined by the pipeline's metadata. **You must discover the pipeline's `namePlural` before making any requests.**

```
/rest/{namePlural}
```

For example, if a pipeline's `namePlural` is `startupFundraisings`, its endpoint is `/rest/startupFundraisings`.

---

## Step 1: Discover Available Pipelines

Always start here before any pipeline operations:

```bash
curl -G "https://app.usedalil.ai/rest/metadata/pipelines" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

The response includes each pipeline's `id`, `nameSingular`, `namePlural`, `labelSingular`, `labelPlural`, and parent record type.

## Step 2: Get Pipeline Fields

To know what fields you can set on a pipeline record:

```bash
curl -G "https://app.usedalil.ai/rest/metadata/pipelines/{pipelineId}" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

The response includes field definitions: names, types, and valid option values (for SELECT fields).

---

## Endpoints

Once you know the `namePlural`, the pattern mirrors all other entities:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/{namePlural}` | Create a pipeline record |
| GET | `/rest/{namePlural}/{id}` | Get a pipeline record by ID |
| GET | `/rest/{namePlural}` | List pipeline records |
| PATCH | `/rest/{namePlural}/{id}` | Update a pipeline record |
| DELETE | `/rest/{namePlural}/{id}` | Delete a pipeline record |
| POST | `/graphql` | Search pipeline records |

---

## Create a Pipeline Record

```
POST /rest/{namePlural}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `recordId` | UUID | Parent record ID (Person or Company — depends on the pipeline) |

### Custom Properties

All other fields are pipeline-specific and discovered from the pipeline's metadata. Common field types include:

- **SELECT** fields (stages, statuses) — values are `UPPER_SNAKE_CASE`
- **CURRENCY** fields — amounts in micros (multiply by 1,000,000)
- **DATE_TIME** fields — ISO 8601 format
- **TEXT**, **NUMBER**, **BOOLEAN** fields

### Request Example

```bash
curl -X POST "https://app.usedalil.ai/rest/startupFundraisings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "recordId": "company-uuid-here",
    "stage": "SERIES_A",
    "fundingAmount": 5000000000000,
    "targetCloseDate": "2024-09-01T00:00:00.000Z"
  }'
```

---

## Get a Pipeline Record

```
GET /rest/{namePlural}/{id}?depth=1
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Record ID |
| `depth` | number | **1** | `1` includes the parent record (Person or Company) |

> **Default depth is 1** for pipeline records (unlike most entities which default to 0), because the parent record is almost always needed.

```bash
curl -G "https://app.usedalil.ai/rest/startupFundraisings/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## List Pipeline Records

```
GET /rest/{namePlural}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 1 | Related objects depth |

```bash
# Records in a specific stage
curl -G "https://app.usedalil.ai/rest/startupFundraisings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=stage[eq]:FUNDED" \
  --data-urlencode "depth=1"

# Records created after a date
curl -G "https://app.usedalil.ai/rest/startupFundraisings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=createdAt[gte]:2024-01-01T00:00:00.000Z"
```

---

## Update a Pipeline Record

Send only the custom properties you want to update.

```
PATCH /rest/{namePlural}/{id}
```

```bash
curl -X PATCH "https://app.usedalil.ai/rest/startupFundraisings/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"stage": "FUNDED"}'
```

---

## Delete a Pipeline Record

```
DELETE /rest/{namePlural}/{id}
```

```bash
curl -X DELETE "https://app.usedalil.ai/rest/startupFundraisings/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Search Pipeline Records

Pipeline search uses a **different GraphQL query** than other entities: `searchPipeline` (not `search`). It also requires the pipeline's `nameSingular`.

### By Name — GraphQL + REST (Two Steps)

**Step 1: GraphQL search → get IDs**

```bash
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query SearchPipeline($searchInput: String!, $nameSingular: String!) { searchPipeline(searchInput: $searchInput, nameSingular: $nameSingular, limit: 5) { recordId } }",
    "variables": {
      "searchInput": "Acme",
      "nameSingular": "startupFundraising"
    }
  }'
```

> **Pipeline GraphQL caveats:**
> - Use `searchPipeline`, not `search` — the regular `search` query does not work for pipelines
> - `nameSingular` is required — this is the singular pipeline name (not `namePlural`)
> - `limit` must be hardcoded in the query string, not a `$limit` variable
> - Returns only `recordId` — follow up with REST to get full records

**Step 2: Fetch full records**

```bash
curl -G "https://app.usedalil.ai/rest/startupFundraisings" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[recordId1,recordId2]" \
  --data-urlencode "depth=1"
```

---

## Full Workflow

```bash
# 1. Discover available pipelines
curl -G "https://app.usedalil.ai/rest/metadata/pipelines" \
  -H "Authorization: Bearer YOUR_API_KEY"
# Note: namePlural (e.g., "startupFundraisings") and nameSingular (e.g., "startupFundraising")

# 2. Get the pipeline's field definitions
curl -G "https://app.usedalil.ai/rest/metadata/pipelines/{pipelineId}" \
  -H "Authorization: Bearer YOUR_API_KEY"
# Note: field names, types, and valid SELECT option values

# 3. Get parent records (companies or people, based on the pipeline type)
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=100"

# 4. Create a pipeline record
curl -X POST "https://app.usedalil.ai/rest/{namePlural}" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"recordId": "parent-uuid", "stage": "STAGE_VALUE", ...}'

# 5. List records
curl -G "https://app.usedalil.ai/rest/{namePlural}" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"

# 6. Update a record
curl -X PATCH "https://app.usedalil.ai/rest/{namePlural}/{id}" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"stage": "NEW_STAGE"}'
```

---

## Filter Examples

```bash
# Records at a specific stage
filter=stage[eq]:FUNDED

# Records created after a date
filter=createdAt[gte]:2024-01-01T00:00:00.000Z

# Records for a specific parent
filter=recordId[eq]:company-uuid
```

---

## Common Gotchas

1. **Endpoints are dynamic.** The URL path comes from `pipelineMetadata.namePlural`. There is no fixed `/rest/pipelines` endpoint for records. Always discover first.

2. **Must discover pipelines before any operation.** Call `GET /rest/metadata/pipelines` to get `namePlural` and `nameSingular` before anything else.

3. **Pipeline search uses `searchPipeline`, not `search`.** Using the regular `search` query will return no results.

4. **GraphQL requires `nameSingular`, not `namePlural`.** The search query takes the singular name (e.g., `startupFundraising`), not the plural.

5. **Parent record is required on create.** `recordId` links the pipeline record to a Company or Person — it cannot be null.

6. **Default depth is 1.** Pipeline records default to `depth=1` (includes parent record) rather than `depth=0`.

7. **Custom fields vary by pipeline.** Each pipeline has unique fields. Always check the metadata before creating or updating records.

8. **SELECT values are UPPER_SNAKE_CASE.** Same as other entities: `FUNDED`, `SERIES_A`, `DEMO`, etc.

9. **Currency amounts are in micros.** Same rule as other entities: multiply actual amount by 1,000,000.

10. **GraphQL `limit` must be hardcoded.** Passing `$limit: Int` as a variable causes a type error. Inline it: `limit: 5`.

11. **`id[in]` requires bracket syntax.** Use `id[in]:[uuid1,uuid2]`.

12. **URL-encode filter parameters.** Use `curl -G --data-urlencode "filter=..."`.

---

## See Also

- [Metadata](metadata.md) — discovering pipeline definitions and custom fields
- [Field Types](field-types.md) — SELECT, CURRENCY, and other field type formats
