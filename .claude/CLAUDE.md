# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository contains API documentation and AI skills files for the **Dalil AI CRM platform** — a system for managing contacts, companies, deals, tasks, and notes.

There are no build commands, tests, or code to run. This is a pure documentation repository.

## Structure

```
apiDocs/
  *.md              # Human-readable API reference guides
.claude/
  skills/
    *.md            # Self-contained skills files for AI agents/automation
```

The two doc types serve different audiences:
- **`apiDocs/*.md`** — detailed API reference for developers integrating with Dalil AI
- **`.claude/skills/*.md`** — condensed, action-oriented references for AI agents to follow directly

## API Architecture

**Base URL:** `https://app.usedalil.ai`

**Two API interfaces:**
- **REST** (`/rest/...`) — all CRUD operations
- **GraphQL** (`/graphql`) — full-text search only (returns IDs; must follow up with REST to get full records)

**Search pattern** (applies to all standard entities):
1. POST to `/graphql` with `searchInput` → get `recordId` list
2. GET `/rest/{entity}?filter=id[in]:[id1,id2]` → get full records

**Pipeline search** uses a different GraphQL query: `searchPipeline` (not `search`), and requires `nameSingular`.

## Key Conventions

### Data Formats
- **Currency**: amounts in micros — multiply actual value by 1,000,000 (e.g., $50 → `50000000`)
- **SELECT / MULTI_SELECT**: values must be `UPPER_SNAKE_CASE`
- **LINKS** (LinkedIn, X, domain): always use `{ primaryLinkUrl, primaryLinkLabel, secondaryLinks: [] }`
- **Person name**: nested object `{ firstName, lastName }`, not flat fields
- **Note/Task body**: `bodyV2` with `{ markdown, blocknote }` — plain text in `body` field is auto-converted

### Query Parameters
- **Pagination**: cursor-based via `starting_after` (UUID of last record); default page size is 60
- **Filtering**: `filter=field[comparator]:value`, comma-separated for multiple; nested fields use dot notation (e.g., `emails.primaryEmail`)
- **Sorting**: `order_by=field[DescNullsLast]`
- **Depth**: `0` = record only, `1` = + relations, `2` = + nested relations

### Pipelines (Dynamic Entities)
Pipeline endpoints are not fixed. Each pipeline has a unique `namePlural` (e.g., `/rest/startupFundraisings`). Always:
1. Discover pipelines first: `GET /rest/metadata/pipelines`
2. Get pipeline fields: `GET /rest/metadata/pipelines/{pipelineId}`
3. Then create/update using the discovered `namePlural`

### Custom Fields
Each workspace can add custom fields to any entity. Discover them via:
```
GET /rest/metadata/objects/standard-id/{standardId}
```
Standard IDs are documented in `apiDocs/metadata.md`. Custom field values follow the same type conventions as standard fields.

### System Fields (Read-Only)
Never set these on create/update: `id`, `createdAt`, `updatedAt`, `deletedAt`, `position`, `createdBy`, `groupId`, `visibilityLevel`, `score`, `lastContactAt`

## Using Skills Files

Before executing any API operation from the skills files, confirm the user has provided a Bearer token (API key). If not already supplied, ask for it — no request can succeed without it.

## Skills Files

The `skills/` files are structured for direct use by AI agents. Each covers a single entity and includes:
- Quick reference (base URL, auth, resource path)
- All endpoint signatures with required fields
- Complete request body examples
- Search patterns (GraphQL + REST two-step)
- Common filter examples
- **Gotchas section** — the most important part for avoiding mistakes

When updating API behavior, both the main doc (`apiDocs/{entity}.md`) and the corresponding skills file (`.claude/skills/{entity}.md`) must be kept in sync.

### Available Skills

Use the table below to select the right skill file for a given operation:

| Skill file | Name | When to use |
|---|---|---|
| `.claude/skills/person.md` | `person` | Creating, searching, updating, or deleting **people/contacts** |
| `.claude/skills/company.md` | `company` | Creating, searching, updating, or deleting **companies/organizations** |
| `.claude/skills/opportunity.md` | `opportunity` | Creating, searching, updating, or deleting **deals/opportunities** |
| `.claude/skills/note.md` | `note` | Creating, searching, updating, or deleting **notes** |
| `.claude/skills/task.md` | `task` | Creating, searching, updating, or deleting **tasks/to-dos** |
| `.claude/skills/pipeline.md` | `pipeline` | Working with **dynamic CRM pipelines** (discover endpoints first) |
| `.claude/skills/note-relation.md` | `note-relation` | **Attaching notes** to people, companies, or opportunities |
| `.claude/skills/task-relation.md` | `task-relation` | **Attaching tasks** to people, companies, or opportunities |

**Selection rules:**
- For note/task operations on a record, use `note` or `task` to create the item, then `note-relation` or `task-relation` to link it.
- For pipeline records (not the pipeline definition itself), use `pipeline` — it handles discovery of dynamic endpoints.
- When an operation spans multiple entities (e.g., create a person and attach a note), read both relevant skills files.
