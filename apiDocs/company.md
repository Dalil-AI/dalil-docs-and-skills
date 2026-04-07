# Company

A Company represents an organization or business in your CRM — a client, prospect, partner, or any entity you work with. Companies can be linked to People, Opportunities, Notes, and Tasks.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/companies` | Create a new company |
| GET | `/rest/companies/{id}` | Get a company by ID |
| GET | `/rest/companies` | List companies |
| PATCH | `/rest/companies/{id}` | Update a company |
| DELETE | `/rest/companies/{id}` | Delete a company |
| POST | `/graphql` | Search companies by name |

---

## Create a Company

```
POST /rest/companies
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Company name |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `domainName.primaryLinkUrl` | string | Company website URL |
| `domainName.primaryLinkLabel` | string | Display label for website |
| `linkedinLink.primaryLinkUrl` | string | LinkedIn page URL |
| `linkedinLink.primaryLinkLabel` | string | Display label for LinkedIn |
| `xLink.primaryLinkUrl` | string | X (Twitter) profile URL |
| `xLink.primaryLinkLabel` | string | Display label for X |
| `introVideo.primaryLinkUrl` | string | Intro video URL |
| `introVideo.primaryLinkLabel` | string | Display label for video |
| `industry` | string | Business sector (e.g., `"Technology"`) |
| `employees` | number | Total employee count (min: 1) |
| `idealCustomerProfile` | boolean | Whether this company matches ICP criteria |
| `tagline` | string | Company tagline or slogan |
| `visaSponsorship` | boolean | Whether the company offers visa sponsorship |
| `workPolicy` | string[] | `ON_SITE`, `HYBRID`, and/or `REMOTE_WORK` |
| `annualRecurringRevenue.amountMicros` | number | ARR in micros (multiply by 1,000,000) |
| `annualRecurringRevenue.currencyCode` | string | Currency code (e.g., `"USD"`) |
| `address` | object | Full address object (see below) |
| `accountOwnerId` | UUID | Account owner (workspace member ID) |

### Address Object

The address is a flat object — all fields are at the same level, prefixed with `address`:

```json
{
  "addressStreet1": "123 Main Street",
  "addressStreet2": "Suite 200",
  "addressCity": "New York",
  "addressState": "New York",
  "addressPostcode": "10001",
  "addressCountry": "United States",
  "addressLat": 40.7128,
  "addressLng": -74.0060
}
```

### Request Example

```bash
curl -X POST "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Corp",
    "domainName": {
      "primaryLinkUrl": "https://acmecorp.com",
      "primaryLinkLabel": "Website",
      "secondaryLinks": []
    },
    "linkedinLink": {
      "primaryLinkUrl": "https://linkedin.com/company/acme-corp",
      "primaryLinkLabel": "LinkedIn",
      "secondaryLinks": []
    },
    "industry": "Technology",
    "employees": 150,
    "idealCustomerProfile": true,
    "workPolicy": ["HYBRID"],
    "annualRecurringRevenue": {
      "amountMicros": 500000000000,
      "currencyCode": "USD"
    },
    "address": {
      "addressStreet1": "123 Main Street",
      "addressCity": "New York",
      "addressState": "New York",
      "addressPostcode": "10001",
      "addressCountry": "United States"
    }
  }'
```

---

## Get a Company

```
GET /rest/companies/{id}?depth=0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Company ID |
| `depth` | number | 0 | `0` = record only, `1` = + relations |

```bash
curl -G "https://app.usedalil.ai/rest/companies/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

> **Depth guidance:** Use `depth=1` to retrieve the company with its direct relations (account owner, opportunities, people). Avoid `depth=2` — it loads all nested relations and returns extremely large payloads (500KB+).

---

## List Companies

```
GET /rest/companies
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor (last record's ID) |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 0 | Related objects depth |

```bash
# Companies with 50+ employees, sorted alphabetically
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=employees[gte]:50" \
  --data-urlencode "order_by=name[AscNullsLast]"

# ICP-matching companies
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=idealCustomerProfile[eq]:true" \
  --data-urlencode "depth=1"
```

---

## Update a Company

Send only the fields you want to change — omitted fields are left unchanged.

```
PATCH /rest/companies/{id}
```

```bash
curl -X PATCH "https://app.usedalil.ai/rest/companies/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "employees": 200,
    "tagline": "Innovation at its finest"
  }'
```

---

## Delete a Company

```
DELETE /rest/companies/{id}
```

```bash
curl -X DELETE "https://app.usedalil.ai/rest/companies/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Search Companies

Companies can be searched by **name** (via GraphQL), or by **LinkedIn URL** or **domain** (via REST filter).

### By Name — GraphQL + REST (Two Steps)

**Step 1: GraphQL search → get IDs**

```bash
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"company\"], limit: 5) { edges { node { recordId objectNameSingular label } } } }",
    "variables": {
      "searchInput": "Acme"
    }
  }'
```

> **GraphQL caveats:**
> - `searchInput` must be a plain `String!` variable
> - `limit` must be hardcoded in the query string (not passed as `$limit` variable)
> - The response returns IDs only — always follow up with REST

**Step 2: Fetch full records**

```bash
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[recordId1,recordId2]" \
  --data-urlencode "depth=1"
```

### By LinkedIn URL — REST Filter

```bash
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=linkedinLink.primaryLinkUrl[ilike]:linkedin.com/company/acme"
```

### By Domain — REST Filter

```bash
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=domainName.primaryLinkUrl[ilike]:acmecorp.com"
```

---

## Filter Examples

```bash
# Companies in a specific industry
filter=industry[ilike]:technology

# Companies with 50 or more employees
filter=employees[gte]:50

# Companies matching ICP
filter=idealCustomerProfile[eq]:true

# Companies with ARR over $100K ($100,000 = 100000000000 micros)
filter=annualRecurringRevenue.amountMicros[gt]:100000000000

# Companies without an assigned owner
filter=accountOwnerId[is]:NULL

# Companies offering remote work
filter=workPolicy[in]:[REMOTE_WORK]
```

---

## Common Gotchas

1. **ARR is in micros.** `$500,000` = `500000000000` micros (eleven digits). Multiply actual amount by 1,000,000.

2. **All link fields need the full structure.** Domain, LinkedIn, X, and intro video all use `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }`. A bare URL string won't work.

3. **Work policy values are UPPER_SNAKE_CASE.** Use `ON_SITE`, `HYBRID`, `REMOTE_WORK` — not `"Remote Work"` or `"remote_work"`.

4. **Address is a flat object.** Unlike most composite types, address fields are not nested — they're all at the top level with `address` prefix (e.g., `addressCity`, not `address.city`).

5. **Domain filter uses a nested path.** Use `domainName.primaryLinkUrl[ilike]:value`, not `domainName[ilike]:value`.

6. **Avoid `depth=2`.** It returns 500KB+ responses. Use `depth=1` for the company's direct relations.

7. **`id[in]` requires bracket syntax.** Use `id[in]:[uuid1,uuid2]` — the brackets are mandatory.

8. **URL-encode filter parameters.** Use `curl -G --data-urlencode "filter=..."` or equivalent.

---

## See Also

- [Person](person.md) — people associated with a company
- [Opportunity](opportunity.md) — deals linked to a company
- [Field Types](field-types.md) — LINKS, CURRENCY, ADDRESS format reference
- [Metadata](metadata.md) — discover custom fields for companies
