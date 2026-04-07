---
name: person
description: Manage people/contacts in Dalil AI CRM — create, read, update, delete, search, and list person records including name, email, phone, LinkedIn, city, job title, and company associations.
---

# Dalil AI: Person API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*
- **Resource path:** `/rest/people`

**GraphQL search (`POST /graphql`):**
- `searchInput` is a plain `String!` variable — do NOT pass as an object `{query: "...", includedObjectNameSingulars: [...]}` (causes type error)
- `limit` must be hardcoded in the query string — do NOT pass as a `$limit` variable (causes type error)
- Returns IDs only — always follow up with REST to fetch full records

## Endpoints

| Operation | Method | Path | Required Fields |
|-----------|--------|------|-----------------|
| Create | POST | `/rest/people` | `name.firstName` |
| Get | GET | `/rest/people/{id}` | `id` (path) |
| List | GET | `/rest/people` | — |
| Update | PATCH | `/rest/people/{id}` | `id` (path) |
| Delete | DELETE | `/rest/people/{id}` | `id` (path) |
| Search by name | POST | `/graphql` | `searchInput` |
| Search by email | GET | `/rest/people?filter=...` | email value |
| Search by LinkedIn | GET | `/rest/people?filter=...` | LinkedIn URL |
| Search by phone | GET | `/rest/people?filter=...` | phone number |

## Create Person

### Request Body

```json
{
  "name": {
    "firstName": "John",
    "lastName": "Smith"
  },
  "emails": {
    "primaryEmail": "john@example.com",
    "additionalEmails": []
  },
  "phones": {
    "primaryPhoneNumber": "5551234567",
    "primaryPhoneCountryCode": "US",
    "primaryPhoneCallingCode": "+1",
    "additionalPhones": []
  },
  "linkedinLink": {
    "primaryLinkUrl": "https://linkedin.com/in/johnsmith",
    "primaryLinkLabel": "LinkedIn",
    "secondaryLinks": []
  },
  "xLink": {
    "primaryLinkUrl": "https://x.com/johnsmith",
    "primaryLinkLabel": "Twitter",
    "secondaryLinks": []
  },
  "avatarUrl": "https://example.com/avatar.jpg",
  "jobTitle": "Software Engineer",
  "city": "New York",
  "companyId": "uuid-of-company",
  "ownerId": "uuid-of-workspace-member"
}
```

### Required Fields

- `name.firstName` (string) — First name of the person

### Optional Fields

- `name.lastName` (string) — Last name
- `emails.primaryEmail` (string) — Primary email
- `emails.additionalEmails` (string[]) — Additional emails
- `phones.primaryPhoneNumber` (string) — Phone number without country code
- `phones.primaryPhoneCountryCode` (string) — Two-letter country code, e.g. "US"
- `phones.primaryPhoneCallingCode` (string) — Calling code with +, e.g. "+1"
- `linkedinLink.primaryLinkUrl` (string) — LinkedIn URL
- `linkedinLink.primaryLinkLabel` (string) — Display label
- `xLink.primaryLinkUrl` (string) — X/Twitter URL
- `xLink.primaryLinkLabel` (string) — Display label
- `avatarUrl` (string) — Avatar image URL
- `jobTitle` (string) — Job title
- `city` (string) — City
- `companyId` (UUID) — Associated company
- `ownerId` (UUID) — Record owner (workspace member)

## Update Person

```
PATCH /rest/people/{id}?depth=0
```

Send only fields to update. Same field shapes as create.

## Get Person

```
GET /rest/people/{id}?depth=0
```

Query params: `depth` (0, 1, or 2)

## List People

```
GET /rest/people?limit=60&order_by=createdAt[DescNullsLast]&filter=...&depth=0
```

Query params: `limit`, `starting_after`, `order_by`, `filter`, `depth`

## Delete Person

```
DELETE /rest/people/{id}
```

## Search Patterns

### By Name (GraphQL + REST)

Step 1 — GraphQL search to get IDs:
```json
{
  "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"person\"], limit: 5) { edges { node { recordId objectNameSingular label imageUrl } } } }",
  "variables": {
    "searchInput": "John Smith"
  }
}
```
POST to `https://app.usedalil.ai/graphql`

Step 2 — Fetch full records:
```
GET /rest/people?filter=id[in]:[recordId1,recordId2]&depth=1
```

### By Email (REST)
```
GET /rest/people?filter=emails.primaryEmail[ilike]:john@example.com&depth=1&limit=5
```

### By LinkedIn (REST)
```
GET /rest/people?filter=linkedinLink.primaryLinkUrl[ilike]:linkedin.com/in/john&depth=1&limit=5
```

### By Phone (REST)
```
GET /rest/people?filter=phones.primaryPhoneNumber[ilike]:5551234567,phones.primaryPhoneCallingCode[eq]:+1&depth=1&limit=5
```

## Filter Examples

```
# People at a specific company
filter=companyId[eq]:company-uuid

# People with a job title containing "engineer"
filter=jobTitle[ilike]:engineer

# People created after a date
filter=createdAt[gte]:2024-01-01T00:00:00.000Z

# People with a score above 5
filter=score[gt]:5

# People without a company
filter=companyId[is]:NULL
```

## Gotchas

1. **Name is a nested object** — Use `{ "name": { "firstName": "John" } }`, not `"firstName": "John"` at the top level.
2. **Link fields require full structure** — LinkedIn, X, and other links need `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }`.
3. **Phone search requires splitting** — When given a number like "+15551234567", split into calling code "+1" and number "5551234567" for the filter.
4. **GraphQL search returns IDs only** — Follow up with `GET /rest/people?filter=id[in]:[id1,id2]` to fetch full records.
5. **Empty fields are auto-cleaned** — Null, undefined, or empty string values are stripped from the request body before sending.
6. **Email filter uses nested path** — Use `emails.primaryEmail[ilike]:value`, not `primaryEmail[ilike]:value`.
7. **GraphQL `limit` must be hardcoded, not a variable** — Passing `$limit: Int` as a variable causes a type error. Inline it directly: `limit: 5`.
8. **`depth=1` responses are large (~42KB for a list)** — Even at depth=1, list responses with multiple records are very large and may exceed tool read limits. Prefer fetching records individually via `GET /rest/people/{id}?depth=1` and extracting only the fields you need, rather than batch-fetching with `filter=id[in]:[...]&depth=1`.
9. **URL-encode GET filter params** — Filter strings contain special characters (`[`, `]`, `:`) that break manually constructed URLs. Use URL encoding when making requests (e.g., `curl -G --data-urlencode "filter=..."`).
10. **`id[in]` filter requires bracket syntax** — The array value must be wrapped in brackets: `id[in]:[uuid1,uuid2]`. Passing a bare comma-separated list (`id[in]:uuid1,uuid2`) causes a 500 error.
11. **GraphQL `searchInput` is a plain string, not an object** — Pass the search term as a `String!` variable. Do NOT pass it as `{query: "...", includedObjectNameSingulars: [...]}` — that causes a type error. The `includedObjectNameSingulars` and `limit` are separate top-level arguments on the `search` field.
12. **`PersonSearchResult` is not a valid GraphQL type** — Do not use inline fragments like `... on PersonSearchResult`. The `search` query returns a unified edge/node structure; access fields directly on `node` (e.g., `recordId`, `objectNameSingular`, `label`).
