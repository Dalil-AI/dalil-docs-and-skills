# Dalil AI API Documentation

Dalil AI is a CRM platform for managing contacts, companies, deals, tasks, and notes. This API gives you full programmatic access to all CRM data — create, read, update, delete, and search any record in your workspace.

**Base URL:** `https://app.usedalil.ai`

---

## Quick Start

**1. Get your API key** from your Dalil AI workspace: **Settings → API Keys**

**2. Make your first request:**

```bash
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=5"
```

**3. Explore the guides below** to start building your integration.

> **Every request requires** `Authorization: Bearer YOUR_API_KEY` in the header. Without it, you'll get a `401 Unauthorized` response.

---

## How the API Works

Dalil AI exposes two interfaces:

| Interface | Path | Purpose |
|-----------|------|---------|
| **REST API** | `/rest/...` | All CRUD operations — create, read, update, delete, list |
| **GraphQL** | `/graphql` | Full-text search only (returns record IDs) |

### The Search Pattern

Full-text search (searching by name, title, etc.) goes through GraphQL, but GraphQL only returns IDs. You then fetch the full records via REST. This two-step approach applies to all standard entities:

```
Step 1: POST /graphql  →  get list of recordIds
Step 2: GET /rest/{entity}?filter=id[in]:[id1,id2]  →  get full records
```

Pipelines use a different GraphQL query (`searchPipeline`) — see the [Pipeline guide](pipeline.md) for details.

### Important Conventions at a Glance

| Convention | Rule |
|------------|------|
| **Currency amounts** | In micros — multiply by 1,000,000 (e.g., $50 → `50000000`) |
| **SELECT / MULTI_SELECT** | Values must be `UPPER_SNAKE_CASE` |
| **Link fields** (LinkedIn, domain, X) | Always `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }` |
| **Person name** | Nested object `{ firstName, lastName }`, not flat fields |
| **Note / Task body** | Use `bodyV2` with `{ markdown, blocknote }` — or send plain `body` text for auto-conversion |
| **Filter parameters** | URL-encode them — they contain `[`, `]`, `:` characters that break raw URLs |

---

## Guides

| Guide | What It Covers |
|-------|----------------|
| [Authentication](authentication.md) | API key setup, auth headers, error codes |
| [Pagination, Filtering & Sorting](pagination-filtering-sorting.md) | Cursor pagination, filter syntax, comparators, sorting |
| [Field Types](field-types.md) | Data formats for every field type — TEXT, CURRENCY, LINKS, FULL_NAME, etc. |
| [Metadata](metadata.md) | Discovering custom fields, pipelines, and workspace members |
| [Webhooks](webhooks.md) | Real-time event notifications |

---

## Resources

### Core Entities

| Resource | Endpoint | Operations |
|----------|----------|------------|
| [Person](person.md) | `/rest/people` | Create, Get, List, Update, Delete, Search |
| [Company](company.md) | `/rest/companies` | Create, Get, List, Update, Delete, Search |
| [Opportunity](opportunity.md) | `/rest/opportunities` | Create, Get, List, Update, Delete, Search |

### Activity Entities

| Resource | Endpoint | Operations |
|----------|----------|------------|
| [Note](note.md) | `/rest/notes` | Create, Get, List, Update, Delete, Search |
| [Task](task.md) | `/rest/tasks` | Create, Get, List, Update, Delete, Search |

### Relationship Entities

Notes and tasks are standalone records. Use these endpoints to link them to people, companies, or opportunities.

| Resource | Endpoint | Operations |
|----------|----------|------------|
| [Note Relation](note-relation.md) | `/rest/noteTargets` | Create, Get, List, Update, Delete |
| [Task Relation](task-relation.md) | `/rest/taskTargets` | Create, Get, List, Update, Delete |

### Dynamic Entities

| Resource | Endpoint | Operations |
|----------|----------|------------|
| [Pipeline](pipeline.md) | `/rest/{namePlural}` | Create, Get, List, Update, Delete, Search |

Pipeline endpoints are dynamic — you must discover the `namePlural` for each pipeline via the [Metadata API](metadata.md) before using them.

---

## Common Workflows

### Create a person and attach a note

```bash
# Step 1: Create the person
curl -X POST "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": {"firstName": "Jane", "lastName": "Doe"}}'

# Step 2: Create the note
curl -X POST "https://app.usedalil.ai/rest/notes" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Intro call", "body": "Discussed product fit."}'

# Step 3: Link note to person
curl -X POST "https://app.usedalil.ai/rest/noteTargets" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"noteId": "note-uuid", "personId": "person-uuid"}'
```

### Search for a company by name

```bash
# Step 1: GraphQL search
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"company\"], limit: 5) { edges { node { recordId label } } } }",
    "variables": {"searchInput": "Acme"}
  }'

# Step 2: Fetch full records (use IDs from step 1)
curl -G "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=id[in]:[id1,id2]" \
  --data-urlencode "depth=1"
```

### Paginate through all people

```bash
# Page 1
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=60" \
  --data-urlencode "order_by=createdAt[DescNullsLast]"

# Page 2 (pass the last record's ID as starting_after)
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=60" \
  --data-urlencode "order_by=createdAt[DescNullsLast]" \
  --data-urlencode "starting_after=LAST-RECORD-UUID"
```
