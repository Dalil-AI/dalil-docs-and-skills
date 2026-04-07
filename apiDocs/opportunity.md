# Opportunity

An Opportunity represents a sales deal, revenue opportunity, or business prospect in your CRM pipeline. Opportunities can be linked to a Company and a point-of-contact Person, and associated with Notes and Tasks.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/opportunities` | Create a new opportunity |
| GET | `/rest/opportunities/{id}` | Get an opportunity by ID |
| GET | `/rest/opportunities` | List opportunities |
| PATCH | `/rest/opportunities/{id}` | Update an opportunity |
| DELETE | `/rest/opportunities/{id}` | Delete an opportunity |
| POST | `/graphql` | Search opportunities by name |

---

## Create an Opportunity

```
POST /rest/opportunities
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Opportunity name |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `amount.amountMicros` | number | Deal amount in micros (multiply by 1,000,000) |
| `amount.currencyCode` | string | Three-letter currency code (e.g., `"USD"`) |
| `stage` | string | Pipeline stage in UPPER_SNAKE_CASE |
| `closeDate` | string | Expected close date (ISO 8601) |
| `companyId` | UUID | Associated company |
| `pointOfContactId` | UUID | Primary contact person |
| `ownerId` | UUID | Opportunity owner (workspace member ID) |

### Stage Values

Stage is a SELECT field — values must be `UPPER_SNAKE_CASE`:

| Stage | Meaning |
|-------|---------|
| `DISCOVERY` | Initial discovery stage |
| `PROPOSAL` | Proposal sent |
| `NEGOTIATION` | In negotiation |
| `CLOSED_WON` | Deal won |
| `CLOSED_LOST` | Deal lost |

> **Note:** Your workspace may have custom stage values. Use the [Metadata API](metadata.md) to discover valid options for your workspace.

### Request Example

```bash
curl -X POST "https://app.usedalil.ai/rest/opportunities" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Q1 Enterprise License Deal",
    "amount": {
      "amountMicros": 50000000000,
      "currencyCode": "USD"
    },
    "stage": "DISCOVERY",
    "closeDate": "2024-06-15T00:00:00.000Z",
    "companyId": "company-uuid-here",
    "pointOfContactId": "person-uuid-here",
    "ownerId": "member-uuid-here"
  }'
```

> **Amount in micros:** `$50,000` = `50000000000` (multiply by 1,000,000). See the [Currency field type](field-types.md#currency) for a conversion table.

---

## Get an Opportunity

```
GET /rest/opportunities/{id}?depth=0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Opportunity ID |
| `depth` | number | 0 | `0` = record only, `1` = + relations |

```bash
curl -G "https://app.usedalil.ai/rest/opportunities/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

---

## List Opportunities

```
GET /rest/opportunities
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor (last record's ID) |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 0 | Related objects depth |

```bash
# Open opportunities by soonest close date
curl -G "https://app.usedalil.ai/rest/opportunities" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=stage[neq]:CLOSED_WON,stage[neq]:CLOSED_LOST" \
  --data-urlencode "order_by=closeDate[AscNullsLast]"

# Opportunities worth over $50K
curl -G "https://app.usedalil.ai/rest/opportunities" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=amount.amountMicros[gt]:50000000000"
```

---

## Update an Opportunity

Send only the fields you want to change.

```
PATCH /rest/opportunities/{id}
```

```bash
# Mark a deal as won and update the amount
curl -X PATCH "https://app.usedalil.ai/rest/opportunities/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "stage": "CLOSED_WON",
    "amount": {
      "amountMicros": 75000000000,
      "currencyCode": "USD"
    }
  }'
```

---

## Delete an Opportunity

```
DELETE /rest/opportunities/{id}
```

```bash
curl -X DELETE "https://app.usedalil.ai/rest/opportunities/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Search Opportunities

Opportunities are searched by **name only** using GraphQL. There is no email or domain search for opportunities.

### By Name — GraphQL + REST (Two Steps)

**Step 1: GraphQL search → get IDs**

```bash
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"opportunity\"], limit: 5) { edges { node { recordId objectNameSingular label } } } }",
    "variables": {
      "searchInput": "Enterprise"
    }
  }'
```

> **GraphQL caveats:**
> - `searchInput` is a plain `String!` variable — do not pass it as an object
> - `limit` must be hardcoded in the query string (not a `$limit` variable)
> - Returns IDs only — always follow up with REST

**Step 2: Fetch full records**

```bash
curl -G "https://app.usedalil.ai/rest/opportunities" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[recordId1,recordId2]" \
  --data-urlencode "depth=1"
```

---

## Filter Examples

```bash
# Opportunities in the discovery stage
filter=stage[eq]:DISCOVERY

# Won opportunities
filter=stage[eq]:CLOSED_WON

# Opportunities worth over $50K ($50,000 = 50000000000 micros)
filter=amount.amountMicros[gt]:50000000000

# Opportunities closing before a specific date
filter=closeDate[lt]:2024-07-01T00:00:00.000Z

# Opportunities for a specific company
filter=companyId[eq]:company-uuid

# Opportunities with no amount set
filter=amount.amountMicros[is]:NULL
```

---

## Common Gotchas

1. **Amount is in micros.** `$50,000` = `50000000000` micros. Multiply actual amount by 1,000,000 — it's easy to be off by a factor of 1,000.

2. **Stage values are UPPER_SNAKE_CASE.** Use `CLOSED_WON`, not `"Closed Won"` or `"closed_won"`.

3. **Amount is a nested object.** Use `{ "amount": { "amountMicros": 50000000000, "currencyCode": "USD" } }`, not `"amountMicros"` at the top level.

4. **Close date must be ISO 8601.** Use `"2024-06-15T00:00:00.000Z"` format.

5. **Search is name-only.** Unlike Person/Company, there is no field-specific REST filter search for opportunities — only GraphQL name search.

6. **Amount filter uses a nested path.** Use `amount.amountMicros[gt]:value`, not `amountMicros[gt]:value`.

7. **`id[in]` requires bracket syntax.** `id[in]:[uuid1,uuid2]` — brackets are required.

8. **URL-encode filter parameters.** Use `curl -G --data-urlencode "filter=..."`.

---

## See Also

- [Company](company.md) — the company associated with a deal
- [Person](person.md) — the point-of-contact for a deal
- [Note Relation](note-relation.md) — attach notes to an opportunity
- [Task Relation](task-relation.md) — attach tasks to an opportunity
- [Field Types](field-types.md) — CURRENCY field format and micros conversion
