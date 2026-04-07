# Dalil AI — API Docs & Agent Skills

This repository contains everything you need to build on top of **[Dalil AI](https://app.usedalil.ai)** — a CRM platform for managing contacts, companies, deals, tasks, and notes.

It serves two audiences:

- **Developers** building custom integrations, automations, or applications on top of Dalil AI
- **AI agents** (Claude, GPT, etc.) that need structured, action-ready references to operate the CRM on a user's behalf

---

## What's Inside

```
apiDocs/          # Human-readable API reference for developers
.claude/skills/   # Condensed skill files designed for AI agents
```

### `apiDocs/`

Detailed API reference guides covering every resource, operation, and convention in the Dalil AI API. Written for developers who want to understand the full picture before building.

| Guide | What It Covers |
|-------|----------------|
| [index.md](apiDocs/index.md) | Overview, quick start, common workflows |
| [authentication.md](apiDocs/authentication.md) | API key setup, auth headers, error codes |
| [field-types.md](apiDocs/field-types.md) | Data formats for every field type |
| [pagination-filtering-sorting.md](apiDocs/pagination-filtering-sorting.md) | Cursor pagination, filter syntax, sorting |
| [metadata.md](apiDocs/metadata.md) | Discovering custom fields, pipelines, workspace members |
| [webhooks.md](apiDocs/webhooks.md) | Real-time event notifications |
| [person.md](apiDocs/person.md) | People / contacts |
| [company.md](apiDocs/company.md) | Companies / organizations |
| [opportunity.md](apiDocs/opportunity.md) | Deals / opportunities |
| [note.md](apiDocs/note.md) | Notes |
| [task.md](apiDocs/task.md) | Tasks |
| [note-relation.md](apiDocs/note-relation.md) | Attaching notes to records |
| [task-relation.md](apiDocs/task-relation.md) | Attaching tasks to records |
| [pipeline.md](apiDocs/pipeline.md) | Dynamic CRM pipelines |

### `.claude/skills/`

Self-contained, action-oriented skill files built for AI agents. Each file covers a single entity and is structured so an AI agent can pick it up and immediately know what endpoints to call, what fields are required, and what mistakes to avoid.

Each skill file includes:
- Endpoint signatures with required fields
- Complete example request bodies
- Search patterns (GraphQL + REST two-step)
- Common filter examples
- A **Gotchas** section — edge cases and common mistakes

| Skill | Entity |
|-------|--------|
| [person.md](.claude/skills/person.md) | People / contacts |
| [company.md](.claude/skills/company.md) | Companies / organizations |
| [opportunity.md](.claude/skills/opportunity.md) | Deals / opportunities |
| [note.md](.claude/skills/note.md) | Notes |
| [task.md](.claude/skills/task.md) | Tasks |
| [note-relation.md](.claude/skills/note-relation.md) | Linking notes to records |
| [task-relation.md](.claude/skills/task-relation.md) | Linking tasks to records |
| [pipeline.md](.claude/skills/pipeline.md) | Dynamic pipelines |

---

## How the API Works

Dalil AI exposes two interfaces:

| Interface | Path | Purpose |
|-----------|------|---------|
| REST API | `/rest/...` | All CRUD operations |
| GraphQL | `/graphql` | Full-text search (returns IDs only) |

**Base URL:** `https://app.usedalil.ai`

**Authentication:** Every request requires `Authorization: Bearer YOUR_API_KEY`. Get your key from **Settings → API Keys** in your workspace.

**Search pattern:** Full-text search goes through GraphQL, which returns record IDs. You then fetch full records via REST:

```
Step 1: POST /graphql  →  get list of recordIds
Step 2: GET /rest/{entity}?filter=id[in]:[id1,id2]  →  get full records
```

---

## Quick Start

```bash
# List your contacts
curl -G "https://app.usedalil.ai/rest/people" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "limit=5"

# Create a company
curl -X POST "https://app.usedalil.ai/rest/companies" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Corp", "domainName": {"primaryLinkUrl": "https://acme.com", "primaryLinkLabel": "Website", "secondaryLinks": []}}'

# Search for a person by name (step 1: GraphQL)
curl -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query Search($searchInput: String!) { search(searchInput: $searchInput, includedObjectNameSingulars: [\"person\"], limit: 5) { edges { node { recordId label } } } }",
    "variables": {"searchInput": "Jane Doe"}
  }'
```

---

## Using the Skills Files with AI Agents

The `.claude/skills/` files are designed to be passed directly into an AI agent's context. When you want an AI agent to perform a CRM operation, give it the relevant skill file and your API key.

**Example prompt to an AI agent:**

> Here is the Dalil AI person skill file: [paste contents of person.md]. My API key is `YOUR_KEY`. Find the contact named "Jane Doe" and update her job title to "Head of Product".

The agent will use the skill file to know exactly which endpoints to call, how to format the request, and what to watch out for.

For operations that span multiple entities (e.g., create a person and attach a note), provide both the `person.md` and `note.md` + `note-relation.md` skill files.

---

## Key Conventions

| Convention | Rule |
|------------|------|
| Currency amounts | In micros — multiply by 1,000,000 (e.g., $50 → `50000000`) |
| SELECT / MULTI_SELECT | Values must be `UPPER_SNAKE_CASE` |
| Link fields (LinkedIn, domain) | Always `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }` |
| Person name | Nested object `{ firstName, lastName }`, not flat fields |
| Note / Task body | Use `bodyV2` with `{ markdown, blocknote }` |
| Pagination | Cursor-based via `starting_after` (UUID of last record) |
| Filtering | `filter=field[comparator]:value` — URL-encode when using curl |

---

## About Dalil AI

[Dalil AI](https://app.usedalil.ai) is a CRM platform built for modern sales and relationship workflows. This repository is the official reference for building on top of it.
