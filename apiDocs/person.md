# Person

A Person represents an individual contact in your CRM — a customer, lead, prospect, or any individual you interact with. People can be associated with a Company and linked to Notes and Tasks.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/people` | Create a new person |
| GET | `/rest/people/{id}` | Get a person by ID |
| GET | `/rest/people` | List people |
| PATCH | `/rest/people/{id}` | Update a person |
| DELETE | `/rest/people/{id}` | Delete a person |
| POST | `/graphql` | Search people by name |

---

## Create a Person

```
POST /rest/people
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name.firstName` | string | First name |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `name.lastName` | string | Last name |
| `emails.primaryEmail` | string | Primary email address |
| `emails.additionalEmails` | string[] | Additional email addresses |
| `phones.primaryPhoneNumber` | string | Phone number (without country code) |
| `phones.primaryPhoneCountryCode` | string | Two-letter country code (e.g., `"US"`) |
| `phones.primaryPhoneCallingCode` | string | Calling code with `+` prefix (e.g., `"+1"`) |
| `linkedinLink.primaryLinkUrl` | string | LinkedIn profile URL |
| `linkedinLink.primaryLinkLabel` | string | Display label for the link |
| `xLink.primaryLinkUrl` | string | X (Twitter) profile URL |
| `xLink.primaryLinkLabel` | string | Display label for X |
| `avatarUrl` | string | URL to avatar image |
| `jobTitle` | string | Job title or position |
| `city` | string | City of residence or work |
| `companyId` | UUID | Associated company ID |
| `ownerId` | UUID | Record owner (workspace member ID) |

### Request Example

```bash
curl -X POST "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": {
      "firstName": "John",
      "lastName": "Smith"
    },
    "emails": {
      "primaryEmail": "john.smith@example.com",
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
    "jobTitle": "Software Engineer",
    "city": "New York",
    "companyId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

---

## Get a Person

```
GET /rest/people/{id}?depth=0
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `id` | UUID (path) | — | Person ID |
| `depth` | number | 0 | `0` = record only, `1` = + relations, `2` = + nested |

```bash
curl -G "https://app.usedalil.ai/rest/people/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "depth=1"
```

---

## List People

```
GET /rest/people
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Pagination cursor (last record's ID) |
| `order_by` | string | — | Sort field and direction |
| `filter` | string | — | Filter conditions |
| `depth` | number | 0 | Related objects depth |

```bash
# 20 most recently created people
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=20" \
  --data-urlencode "order_by=createdAt[DescNullsLast]"

# People at a specific company
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=companyId[eq]:company-uuid-here" \
  --data-urlencode "depth=1"
```

---

## Update a Person

Send only the fields you want to change — omitted fields are left unchanged.

```
PATCH /rest/people/{id}
```

```bash
curl -X PATCH "https://app.usedalil.ai/rest/people/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "jobTitle": "Senior Software Engineer",
    "city": "San Francisco"
  }'
```

---

## Delete a Person

```
DELETE /rest/people/{id}
```

```bash
curl -X DELETE "https://app.usedalil.ai/rest/people/uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Search People

People can be searched by **name** (via GraphQL), or by **email**, **LinkedIn URL**, or **phone** (via REST filter).

### By Name — GraphQL + REST (Two Steps)

GraphQL full-text search returns record IDs only. You must follow up with a REST request to get full records.

**Step 1: GraphQL search → get IDs**

```bash
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"person\"], limit: 5) { edges { node { recordId objectNameSingular label } } } }",
    "variables": {
      "searchInput": "John Smith"
    }
  }'
```

> **GraphQL caveats:**
> - `searchInput` must be a plain `String!` variable — do NOT pass it as `{query: "...", includedObjectNameSingulars: [...]}` (causes a type error)
> - `limit` must be hardcoded in the query string, not passed as a `$limit` variable (causes a type error)
> - Do not use inline fragments like `... on PersonSearchResult` — the `search` query returns a unified structure; access `recordId`, `label`, etc. directly on `node`

**Step 2: Fetch full records**

```bash
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[recordId1,recordId2]" \
  --data-urlencode "depth=1"
```

### By Email — REST Filter

```bash
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=emails.primaryEmail[ilike]:john@example.com" \
  --data-urlencode "depth=1"
```

### By LinkedIn URL — REST Filter

```bash
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=linkedinLink.primaryLinkUrl[ilike]:linkedin.com/in/johnsmith" \
  --data-urlencode "depth=1"
```

### By Phone Number — REST Filter

Phone numbers are stored split: the calling code (`+1`) and the number (`5551234567`) are separate fields. Filter both for precise matching:

```bash
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=phones.primaryPhoneNumber[ilike]:5551234567,phones.primaryPhoneCallingCode[eq]:+1" \
  --data-urlencode "depth=1"
```

---

## Filter Examples

```bash
# People at a specific company
filter=companyId[eq]:company-uuid

# People with "engineer" in their job title
filter=jobTitle[ilike]:engineer

# People without a company assigned
filter=companyId[is]:NULL

# People with a score above 5
filter=score[gt]:5

# People created after a specific date
filter=createdAt[gte]:2024-01-01T00:00:00.000Z

# People in San Francisco
filter=city[ilike]:san francisco
```

---

## Common Gotchas

1. **Name is a nested object.** Use `{ "name": { "firstName": "John" } }`, not `"firstName": "John"` at the top level. Sending a flat `firstName` field will be ignored.

2. **Link fields require the full structure.** LinkedIn, X, and other link fields need `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }`. Sending just a URL string won't work.

3. **Phone search requires splitting the number.** When given a number like `+15551234567`, split it into calling code `+1` and number `5551234567` for the filter.

4. **Email filter uses a nested path.** Use `emails.primaryEmail[ilike]:value`, not `primaryEmail[ilike]:value`.

5. **GraphQL search returns IDs only.** Always follow up with `GET /rest/people?filter=id[in]:[id1,id2]` to get full records.

6. **`id[in]` requires bracket syntax.** Use `id[in]:[uuid1,uuid2]` — not `id[in]:uuid1,uuid2`. The brackets are required or you'll get a 500 error.

7. **URL-encode filter parameters.** Filter strings contain `[`, `]`, `:` that break raw URLs. Use `curl -G --data-urlencode "filter=..."` or equivalent in your HTTP library.

8. **`depth=1` list responses are large.** A single `depth=1` list call can return ~42KB+ per page. If you need depth on specific records, prefer fetching them individually via `GET /rest/people/{id}?depth=1`.

---

## See Also

- [Field Types](field-types.md) — full reference for FULL_NAME, EMAILS, PHONES, LINKS formats
- [Note Relation](note-relation.md) — how to attach notes to a person
- [Task Relation](task-relation.md) — how to attach tasks to a person
- [Pagination, Filtering & Sorting](pagination-filtering-sorting.md) — complete filter syntax reference
