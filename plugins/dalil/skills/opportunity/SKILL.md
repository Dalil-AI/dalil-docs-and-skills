---
name: opportunity
description: Manage sales opportunities/deals in Dalil AI CRM — create, read, update, delete, search, and list opportunity records including deal name, amount (in micros), stage, close date, and point-of-contact associations.
---

# Dalil AI: Opportunity API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/opportunities`

**GraphQL search (`POST /graphql`):**
- `searchInput` is a plain `String!` variable — do NOT pass as an object `{query: "...", includedObjectNameSingulars: [...]}` (causes type error)
- `limit` must be hardcoded in the query string — do NOT pass as a `$limit` variable (causes type error)
- Returns IDs only — always follow up with REST to fetch full records

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/opportunities` | `name` |
| Get | GET | `/rest/opportunities/{id}` | `id` (path) |
| List | GET | `/rest/opportunities` | — |
| Update | PATCH | `/rest/opportunities/{id}` | `id` (path) |
| Delete | DELETE | `/rest/opportunities/{id}` | `id` (path) |
| Search by name | POST | `/graphql` | `searchInput` |

## Create Opportunity

### Request Body

```json
{
  "name": "Q1 Enterprise License Deal",
  "amount": {
    "amountMicros": 50000000000,
    "currencyCode": "USD"
  },
  "stage": "DISCOVERY",
  "closeDate": "2024-06-15T00:00:00.000Z",
  "companyId": "uuid-of-company",
  "pointOfContactId": "uuid-of-person",
  "ownerId": "uuid-of-workspace-member"
}
```

### Required Fields

- `name` (string) — Opportunity name

### Optional Fields

- `amount.amountMicros` (number) — Deal amount in micros (multiply by 1,000,000)
- `amount.currencyCode` (string) — Three-letter currency code, e.g. "USD"
- `stage` (string) — Pipeline stage in UPPER_SNAKE_CASE: `DISCOVERY`, `PROPOSAL`, `NEGOTIATION`, `CLOSED_WON`, `CLOSED_LOST`
- `closeDate` (string) — Expected close date in ISO 8601 format
- `companyId` (UUID) — Associated company
- `pointOfContactId` (UUID) — Main contact person
- `ownerId` (UUID) — Opportunity owner (workspace member)

## Update Opportunity

```
PATCH /rest/opportunities/{id}?depth=0
```

Send only fields to update. Same shapes as create. `name` is also updatable.

## Get Opportunity

```
GET /rest/opportunities/{id}?depth=0
```

## List Opportunities

```
GET /rest/opportunities?limit=60&order_by=closeDate[AscNullsLast]&filter=...&depth=0
```

## Delete Opportunity

```
DELETE /rest/opportunities/{id}
```

## Search Pattern

Search by name only (GraphQL + REST):

Step 1 — GraphQL:
```json
{
  "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"opportunity\"], limit: 5) { edges { node { recordId objectNameSingular label imageUrl } } } }",
  "variables": { "searchInput": "Enterprise" }
}
```
POST to `https://app.usedalil.ai/graphql`

Step 2 — Fetch:
```
GET /rest/opportunities?filter=id[in]:[recordId1,recordId2]&depth=1
```

## Filter Examples

```
# Opportunities in discovery stage
filter=stage[eq]:DISCOVERY

# Opportunities worth over $50K ($50,000 = 50000000000 micros)
filter=amount.amountMicros[gt]:50000000000

# Opportunities closing before a date
filter=closeDate[lt]:2024-07-01T00:00:00.000Z

# Opportunities for a specific company
filter=companyId[eq]:company-uuid

# Won opportunities
filter=stage[eq]:CLOSED_WON
```

## Gotchas

1. **Amount is in micros** — `$50,000` = `50000000000` micros. Multiply actual amount by 1,000,000.
2. **Stage values are UPPER_SNAKE_CASE** — Use `CLOSED_WON`, not `Closed Won` or `closed_won`.
3. **Amount is a nested object** — Use `{ "amount": { "amountMicros": 50000000000, "currencyCode": "USD" } }`.
4. **Close date is ISO 8601** — Use format `2024-06-15T00:00:00.000Z`.
5. **Search is name-only** — Unlike Person/Company, no field-specific search (email, domain, etc.).
6. **GraphQL search returns IDs only** — Follow up with `GET /rest/opportunities?filter=id[in]:[id1,id2]` to fetch full records.
7. **Amount filter uses nested path** — Use `amount.amountMicros[gt]:value`.
8. **GraphQL `limit` must be hardcoded, not a variable** — Passing `$limit: Int` as a variable causes a type error. Inline it directly: `limit: 5`.
9. **`depth=1` responses are large** — Use `depth=0` if you only need the opportunity's own fields without relations.
10. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
