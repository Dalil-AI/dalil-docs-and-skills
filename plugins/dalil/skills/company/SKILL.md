---
name: company
description: Manage companies/organizations in Dalil AI CRM — create, read, update, delete, search, and list company records including name, domain, LinkedIn, address, employee count, and annual revenue.
---

# Dalil AI: Company API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/companies`

**GraphQL search (`POST /graphql`):**
- `searchInput` is a plain `String!` variable — do NOT pass as an object `{query: "...", includedObjectNameSingulars: [...]}` (causes type error)
- `limit` must be hardcoded in the query string — do NOT pass as a `$limit` variable (causes type error)
- Returns IDs only — always follow up with REST to fetch full records

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/companies` | `name` |
| Get | GET | `/rest/companies/{id}` | `id` (path) |
| List | GET | `/rest/companies` | — |
| Update | PATCH | `/rest/companies/{id}` | `id` (path) |
| Delete | DELETE | `/rest/companies/{id}` | `id` (path) |
| Search by name | POST | `/graphql` | `searchInput` |
| Search by LinkedIn | GET | `/rest/companies?filter=...` | LinkedIn URL |
| Search by domain | GET | `/rest/companies?filter=...` | domain URL |

## Create Company

### Request Body

```json
{
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
  "xLink": {
    "primaryLinkUrl": "https://x.com/acmecorp",
    "primaryLinkLabel": "Twitter",
    "secondaryLinks": []
  },
  "introVideo": {
    "primaryLinkUrl": "https://youtube.com/watch?v=example",
    "primaryLinkLabel": "Company Overview",
    "secondaryLinks": []
  },
  "industry": "Technology",
  "employees": 150,
  "idealCustomerProfile": true,
  "tagline": "Innovation at its finest",
  "visaSponsorship": false,
  "workPolicy": ["ON_SITE", "HYBRID"],
  "annualRecurringRevenue": {
    "amountMicros": 500000000000,
    "currencyCode": "USD"
  },
  "address": {
    "addressStreet1": "123 Main Street",
    "addressStreet2": "Suite 200",
    "addressCity": "New York",
    "addressState": "New York",
    "addressPostcode": "10001",
    "addressCountry": "United States",
    "addressLat": 40.7128,
    "addressLng": -74.0060
  },
  "accountOwnerId": "uuid-of-workspace-member"
}
```

### Required Fields

- `name` (string) — Company name

### Optional Fields

- `domainName` (LINKS) — Company website `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }`
- `linkedinLink` (LINKS) — LinkedIn page
- `xLink` (LINKS) — X/Twitter profile
- `introVideo` (LINKS) — Intro video URL
- `industry` (string) — Business sector
- `employees` (number) — Employee count (min: 1)
- `idealCustomerProfile` (boolean) — ICP match
- `tagline` (string) — Company tagline
- `visaSponsorship` (boolean) — Offers visa sponsorship
- `workPolicy` (string[]) — Values: `ON_SITE`, `HYBRID`, `REMOTE_WORK`
- `annualRecurringRevenue` (CURRENCY) — `{ amountMicros, currencyCode }`
- `address` (ADDRESS) — Full address object
- `accountOwnerId` (UUID) — Account owner

## Update Company

```
PATCH /rest/companies/{id}?depth=0
```

Send only fields to update. Same field shapes as create. `name` is also updatable.

## Get Company

```
GET /rest/companies/{id}?depth=0
```

## List Companies

```
GET /rest/companies?limit=60&order_by=name[AscNullsLast]&filter=...&depth=0
```

## Delete Company

```
DELETE /rest/companies/{id}
```

## Search Patterns

### By Name (GraphQL + REST)

Step 1 — GraphQL:
```json
{
  "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"company\"], limit: 5) { edges { node { recordId objectNameSingular label imageUrl } } } }",
  "variables": { "searchInput": "Acme" }
}
```
POST to `https://app.usedalil.ai/graphql`

Step 2 — Fetch full records:
```
GET /rest/companies?filter=id[in]:[recordId1,recordId2]&depth=1
```

### By LinkedIn (REST)
```
GET /rest/companies?filter=linkedinLink.primaryLinkUrl[ilike]:linkedin.com/company/acme&depth=1&limit=5
```

### By Domain (REST)
```
GET /rest/companies?filter=domainName.primaryLinkUrl[ilike]:acmecorp.com&depth=1&limit=5
```

## Filter Examples

```
# Companies in technology industry
filter=industry[ilike]:technology

# Companies with 50+ employees
filter=employees[gte]:50

# Companies matching ICP
filter=idealCustomerProfile[eq]:true

# Companies with ARR over $100K
filter=annualRecurringRevenue.amountMicros[gt]:100000000000

# Companies without an owner
filter=accountOwnerId[is]:NULL
```

## Gotchas

1. **ARR is in micros** — `$500,000` = `500000000000` micros. Multiply actual amount by 1,000,000.
2. **Link fields need full structure** — Domain, LinkedIn, X, and intro video all use `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }`.
3. **Work policy values are UPPER_SNAKE_CASE** — Use `ON_SITE`, `HYBRID`, `REMOTE_WORK`.
4. **Address is a flat object** — All address fields are at the same level (no nesting), prefixed with `address`.
5. **Domain filter uses nested path** — Use `domainName.primaryLinkUrl[ilike]:value`.
6. **GraphQL search returns IDs only** — Follow up with `GET /rest/companies?filter=id[in]:[id1,id2]` to fetch full records.
7. **Empty fields are auto-cleaned** — Null/undefined/empty values are stripped before sending.
8. **GraphQL `limit` must be hardcoded, not a variable** — Passing `$limit: Int` as a variable causes a type error (`Int` vs `Int!`). Inline the limit directly in the query string: `limit: 5`.
9. **Use `depth=1` for full company fetch, not `depth=2`** — `depth=2` returns extremely large payloads (500KB+) by loading all nested relations. `depth=1` is sufficient to get the company plus its direct relations (account owner, opportunities, people, etc.).
10. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
