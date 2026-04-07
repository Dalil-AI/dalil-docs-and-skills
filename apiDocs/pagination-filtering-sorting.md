# Pagination, Filtering & Sorting

---

## Pagination

Dalil AI uses **cursor-based pagination** on all list endpoints. Unlike page-number pagination, this approach stays consistent even when records are added or removed between requests.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 60 | Records per page |
| `starting_after` | UUID | — | Return records after this record ID |
| `ending_before` | UUID | — | Return records before this record ID |

### How It Works

1. Make your first request with a `limit`
2. Take the `id` of the **last record** in the response
3. Pass it as `starting_after` in the next request
4. Repeat until the response returns fewer records than your `limit` (that's the last page)

### Example

```bash
# Page 1
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=20" \
  --data-urlencode "order_by=createdAt[DescNullsLast]"

# Page 2 — pass the last record's ID from page 1
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=20" \
  --data-urlencode "order_by=createdAt[DescNullsLast]" \
  --data-urlencode "starting_after=550e8400-e29b-41d4-a716-446655440000"
```

---

## Filtering

Use the `filter` query parameter to narrow results. Multiple filters are comma-separated and applied with AND logic.

### Syntax

```
filter=field[comparator]:value
```

> **Important:** Filter strings contain special characters (`[`, `]`, `:`) that break raw URLs. Always URL-encode filter parameters:
> - With curl: use `-G` + `--data-urlencode "filter=..."`
> - With HTTP libraries: pass filters as query parameters, not raw strings

### Comparators

| Comparator | Description | Example |
|------------|-------------|---------|
| `eq` | Equals | `stage[eq]:DISCOVERY` |
| `neq` | Not equal to | `status[neq]:DONE` |
| `in` | In list | `id[in]:[uuid1,uuid2,uuid3]` |
| `gt` | Greater than | `score[gt]:5` |
| `gte` | Greater than or equal | `employees[gte]:50` |
| `lt` | Less than | `score[lt]:10` |
| `lte` | Less than or equal | `amount.amountMicros[lte]:100000000` |
| `startsWith` | Starts with (case-sensitive) | `name[startsWith]:Ac` |
| `like` | Contains (case-sensitive) | `title[like]:Meeting` |
| `ilike` | Contains (case-insensitive) | `name[ilike]:acme` |
| `is` | Null / not-null check | `companyId[is]:NULL` or `companyId[is]:NOT_NULL` |

### Nested Field Filtering

Composite fields (EMAILS, PHONES, LINKS, CURRENCY) use dot notation to access sub-fields:

```bash
# Filter by primary email
filter=emails.primaryEmail[ilike]:john@example.com

# Filter by phone number
filter=phones.primaryPhoneNumber[ilike]:5551234567

# Filter by LinkedIn URL
filter=linkedinLink.primaryLinkUrl[ilike]:linkedin.com/in/john

# Filter by company website domain
filter=domainName.primaryLinkUrl[ilike]:acmecorp.com

# Filter by deal amount (in micros)
filter=amount.amountMicros[gt]:50000000000
```

### Multiple Filters (AND)

Separate multiple conditions with commas:

```bash
# People named John with score above 5
filter=name.firstName[eq]:John,score[gt]:5

# Tech companies with 50+ employees
filter=industry[ilike]:technology,employees[gte]:50

# Open opportunities worth over $1K
filter=stage[neq]:CLOSED_LOST,amount.amountMicros[gt]:1000000000
```

### Filtering by Multiple IDs

Use the `in` comparator with bracket-enclosed, comma-separated UUIDs. **The brackets are required.**

```bash
filter=id[in]:[uuid1,uuid2,uuid3]
```

> **Warning:** Omitting the brackets — `id[in]:uuid1,uuid2` — causes a `500` error. The correct form wraps the list: `id[in]:[uuid1,uuid2]`.

---

## Sorting

Use the `order_by` parameter to sort results.

### Syntax

```
order_by=field[Direction]
```

### Sort Directions

| Direction | Meaning |
|-----------|---------|
| `AscNullsFirst` | Ascending, nulls sorted first |
| `AscNullsLast` | Ascending, nulls sorted last |
| `DescNullsFirst` | Descending, nulls sorted first |
| `DescNullsLast` | Descending, nulls sorted last |

### Examples

```bash
# Newest first
order_by=createdAt[DescNullsLast]

# Alphabetical by name
order_by=name[AscNullsLast]

# Soonest close date first (nulls last)
order_by=closeDate[AscNullsLast]

# By due date ascending
order_by=dueAt[AscNullsLast]
```

---

## Depth (Related Objects)

The `depth` parameter controls how many levels of related records are included in the response.

| Value | What's Included |
|-------|----------------|
| `0` | The primary record only (default for most entities) |
| `1` | Primary record + directly related records (e.g., a person's company) |
| `2` | Primary record + relations + their relations |

```bash
# Person record only
GET /rest/people/uuid?depth=0

# Person with their company and tasks
GET /rest/people/uuid?depth=1

# Company with its people and their companies (expensive)
GET /rest/companies/uuid?depth=2
```

> **Warning:** `depth=2` returns extremely large payloads (500KB+ per record) because it loads all nested relations. Avoid it unless you specifically need deeply nested data. `depth=1` is sufficient for the vast majority of use cases.

> **Note:** Note relations and task relations default to `depth=1` rather than `depth=0`, because the relation itself is only useful when it includes the linked record.
