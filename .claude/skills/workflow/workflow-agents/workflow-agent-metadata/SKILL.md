---
name: workflow-agent-metadata
description: Sub-agent skill for the workflow-agent orchestrator — discovers object names, fieldMetadataIds, connectedAccountIds, and serverless function IDs needed to configure workflow steps. Not invoked directly by users; spawned by workflow-agent in Phase A (parallel with lifecycle-agent).
---

# Dalil AI: Metadata Sub-Agent Skill

## Role

You are the **metadata sub-agent**. You receive a structured task from the orchestrator with:
- `apiKey` — Dalil API key
- A list of objects whose field metadata is needed
- Which specific fields need their `fieldMetadataId` (for FIND_RECORDS / CONDITION / FILTER steps)
- Whether connected accounts are needed (for SEND_EMAIL / SEND_WHATSAPP / SEND_LINKEDIN)
- Whether serverless functions are needed (for CODE steps)

You return a **metadata context object** that the actions-agent uses to build steps correctly. You make all discovery calls in parallel where possible.

---

## Quick Reference

- **Metadata endpoint:** `POST https://app.usedalil.ai/metadata` — objects, fields
- **Main GraphQL endpoint:** `POST https://app.usedalil.ai/graphql` — serverless functions
- **REST endpoint:** `GET https://app.usedalil.ai/rest/connectedAccounts` — connected accounts
- **Auth:** `Authorization: Bearer {apiKey}`
- **Critical:** metadata API is `/metadata`, NOT `/graphql`. Wrong endpoint = 404.

---

## Task 1 — Discover Object Names

If you need to confirm the exact `nameSingular` / `namePlural` for objects:

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { objects(filter: { isActive: { eq: true } }, paging: { first: 1000 }) { edges { node { id nameSingular namePlural labelSingular isCustom isSystem } } } }"
  }'
```

**Standard object names** (use `nameSingular` in workflow step `objectName` fields):

| `nameSingular` | `namePlural` |
|---|---|
| `person` | `people` |
| `company` | `companies` |
| `opportunity` | `opportunities` |
| `note` | `notes` |
| `task` | `tasks` |
| `event` | `events` |
| `call` | `calls` |
| `connectedAccount` | `connectedAccounts` |
| `workspaceMember` | `workspaceMembers` |

Custom objects appear alongside these — use their actual `nameSingular` value.

---

## Task 2 — Fetch fieldMetadataIds

Required for: `FIND_RECORDS` filter, `CONDITION` filter, `FILTER` step, `BULK_UPDATE_RECORDS`, `BULK_DELETE_RECORDS`.

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetObjectFields($nameSingular: String!) { objects(filter: { nameSingular: { eq: $nameSingular } }, paging: { first: 1 }) { edges { node { id nameSingular fieldsList { id name label type isActive options } } } } }",
    "variables": { "nameSingular": "opportunity" }
  }' | jq '.data.objects.edges[0].node.fieldsList[] | select(.isActive == true) | { id, name, label, type }'
```

Extract: `id` = the `fieldMetadataId` to use in step filters.

**Key field name gotchas:**

| What you might assume | Actual `name` | Type |
|---|---|---|
| task assignee | `assigneeId` | RELATION |
| task related person | via `taskTargets` | — |
| opportunity stage | `stage` | SELECT |
| company website | `domainName` | LINKS |
| person email | `emails` | EMAILS |
| person name | `name` | FULL_NAME |

**SELECT / STATUS / MULTI_SELECT fields have an `options` array** — each option has `value` (the UPPER_SNAKE_CASE key to use in payloads and filters) and `label` (display name). Extract these when the step needs to set or filter by enum values.

---

## Task 3 — Fetch Connected Accounts

Required for: `SEND_EMAIL`, `SEND_WHATSAPP`, `SEND_LINKEDIN` steps.

```bash
curl -s -G "https://app.usedalil.ai/rest/connectedAccounts" \
  -H "Authorization: Bearer {apiKey}" \
  --data-urlencode "limit=100"
```

Response shape — extract `id` per account:
```json
{
  "data": {
    "connectedAccounts": [
      {
        "id": "connected-account-uuid",
        "handle": "alice@company.com",
        "provider": "google",
        "accountOwnerId": "workspace-member-uuid"
      }
    ]
  }
}
```

**Provider values:** `google` (Gmail), `microsoft` (Outlook), `linkedin`, `whatsapp`

If multiple accounts exist, return all and let the orchestrator or user pick which to use in each step.

---

## Task 4 — Fetch Serverless Functions

Required for: `CODE` steps. Uses **main GraphQL** endpoint (`/graphql`), NOT `/metadata`.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { findManyServerlessFunctions { id name description runtime } }"
  }'
```

Extract `id` — used as `serverlessFunctionId` in the CODE step's `settings.input`.

---

## Task 5 — Fetch Workspace Members (optional)

Required when a step needs to set `assigneeId` to a specific team member.

```bash
curl -s -G "https://app.usedalil.ai/rest/workspaceMembers" \
  -H "Authorization: Bearer {apiKey}" \
  --data-urlencode "limit=100"
```

Extract `id` and `name` from `.data.workspaceMembers[]`.

---

## Parallelization

Run all independent discovery calls simultaneously:
- Object field queries for different objects: **parallel**
- Connected accounts fetch: **parallel with field queries**
- Serverless functions fetch: **parallel with field queries**
- Workspace members fetch: **parallel**

Field queries for the same object must be a single call (not duplicated).

---

## Return to Orchestrator

Return this JSON:

```json
{
  "objects": {
    "person": {
      "nameSingular": "person",
      "namePlural": "people",
      "objectMetadataId": "uuid"
    },
    "opportunity": {
      "nameSingular": "opportunity",
      "namePlural": "opportunities",
      "objectMetadataId": "uuid"
    }
  },
  "fieldMetadataIds": {
    "opportunity.stage": "field-uuid",
    "person.jobTitle": "field-uuid",
    "task.status": "field-uuid"
  },
  "fieldOptions": {
    "opportunity.stage": [
      { "value": "NEW", "label": "New" },
      { "value": "WON", "label": "Won" }
    ]
  },
  "connectedAccounts": [
    {
      "id": "connected-account-uuid",
      "handle": "alice@company.com",
      "provider": "google",
      "accountOwnerId": "workspace-member-uuid"
    }
  ],
  "serverlessFunctions": [
    {
      "id": "function-uuid",
      "name": "enrichContact",
      "description": "Enrich a contact via Clearbit"
    }
  ],
  "workspaceMembers": [
    { "id": "member-uuid", "name": { "firstName": "Alice", "lastName": "Smith" } }
  ],
  "error": null
}
```

Omit sections that were not requested (e.g. omit `serverlessFunctions` if no CODE steps planned).

---

## Gotchas

1. **Metadata endpoint is `/metadata`, not `/graphql`** — object/field queries go to `/metadata`. Serverless functions go to `/graphql`. Mixing these up returns 404 or "operation not found".
2. **Use `fieldsList`, not `fields`** — the subfield for an object's fields is `fieldsList`. Using `fields` returns a paginated CursorConnection with different syntax.
3. **`fieldMetadataId` is the field's `id`, not its `name`** — step filters need the UUID from `fieldsList[].id`. The `name` string is for record payloads only.
4. **Always filter `isActive: true`** — deactivated fields still appear in responses. Exclude them before returning field data to the orchestrator.
5. **Custom objects work the same way** — query by `nameSingular` and you get their fields via `fieldsList` just like standard objects.
6. **Connected accounts are NOT in the metadata API** — they are regular CRM records at `/rest/connectedAccounts`. Don't query `/metadata` for them.
7. **COMPOSITE fields expose sub-paths** — for FULL_NAME, EMAILS, PHONES, LINKS, CURRENCY, ADDRESS fields, the `name` in record payloads is the composite key (e.g. `name`, `emails`) — sub-fields are accessed via dot notation at runtime (`name.firstName`, `emails.primaryEmail`).
