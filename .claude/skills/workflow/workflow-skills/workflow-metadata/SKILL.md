---
name: workflow-metadata
description: Discover the objects and fields available in a Dalil AI workspace — query the metadata API to list objects, get field definitions (id, name, type, label), fetch fieldMetadataIds for step filters, and list connectedAccounts and serverless functions needed by workflow actions.
---

# Dalil AI: Workflow Metadata Skill

## Quick Reference

- **Metadata GraphQL endpoint:** `POST https://app.usedalil.ai/metadata` ← different from `/graphql`
- **Main GraphQL endpoint:** `POST https://app.usedalil.ai/graphql` ← used for serverless functions
- **REST endpoint:** `GET https://app.usedalil.ai/rest/{objectNamePlural}` ← used for connectedAccounts
- **Auth:** `Authorization: Bearer {apiKey}`
- **Why this matters:** Workflow actions (`workflow-actions` skill) require exact `objectName` values, `fieldMetadataId` UUIDs for filters, and `connectedAccountId` UUIDs for communication steps. All of these come from the metadata API.

**Critical rules:**
- The metadata API lives at `/metadata`, NOT `/graphql` — using the wrong endpoint will return a 404 or "operation not found" error
- Object names used in workflow actions (`objectName`, `eventName`, `objectNameSingular`) must match `nameSingular` exactly — they are lowercase and camelCase (e.g. `person`, `company`, `connectedAccount`)
- Always filter by `isActive: true` when listing fields to exclude soft-deleted or deactivated fields
- `fieldsList` (not `fields`) is the correct subfield name for fetching an object's fields in this API

---

## List All Objects

Returns all objects in the workspace — standard (built-in) and custom.

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { objects(paging: { first: 1000 }) { edges { node { id nameSingular namePlural labelSingular labelPlural isCustom isActive isSystem } } } }"
  }'
```

Response shape:
```json
{
  "data": {
    "objects": {
      "edges": [
        {
          "node": {
            "id": "uuid",
            "nameSingular": "person",
            "namePlural": "people",
            "labelSingular": "Person",
            "labelPlural": "People",
            "isCustom": false,
            "isActive": true,
            "isSystem": false
          }
        }
      ]
    }
  }
}
```

**Common standard objects** (use `nameSingular` in workflow actions):

| `nameSingular` | `namePlural` | Description |
|---|---|---|
| `person` | `people` | Contacts |
| `company` | `companies` | Company records |
| `opportunity` | `opportunities` | Deals/pipeline records |
| `note` | `notes` | Notes |
| `task` | `tasks` | Tasks |
| `event` | `events` | Events |
| `call` | `calls` | Call records |
| `connectedAccount` | `connectedAccounts` | Email/messaging integrations |
| `message` | `messages` | Emails/messages |
| `workspaceMember` | `workspaceMembers` | Team members |

Filter to only active, non-system objects for user-facing selectors:
```graphql
objects(filter: { isActive: { eq: true }, isSystem: { eq: false } }, paging: { first: 1000 })
```

---

## Get Fields for an Object

Returns all fields defined on a specific object, including their `id` (the `fieldMetadataId` used in step filters).

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetObjectFields($nameSingular: String!) { objects(filter: { nameSingular: { eq: $nameSingular } }, paging: { first: 1 }) { edges { node { id nameSingular labelSingular fieldsList { id name label type isActive isNullable isSystem isCustom options settings objectMetadataId } } } } }",
    "variables": { "nameSingular": "person" }
  }'
```

Response shape — one field node:
```json
{
  "id": "field-metadata-uuid",
  "name": "firstName",
  "label": "First Name",
  "type": "TEXT",
  "isActive": true,
  "isNullable": true,
  "isSystem": false,
  "isCustom": false,
  "options": null,
  "settings": null
}
```

**Key field properties:**

| Property | Use |
|---|---|
| `id` | The `fieldMetadataId` to use in `StepFilter.fieldMetadataId` |
| `name` | The field key used in record payloads and `{{trigger.name}}` variable paths |
| `type` | Determines which filter operands are valid (see table below) |
| `options` | Present for SELECT / MULTI_SELECT / STATUS — lists available option values |
| `isActive` | Filter to `true` to exclude deactivated fields |

---

## FieldMetadataType Reference

| Type | Description | Example fields |
|---|---|---|
| `TEXT` | Plain text | `jobTitle`, `city` |
| `FULL_NAME` | Composite: `{ firstName, lastName }` | `name` on `person` |
| `EMAILS` | Composite: `{ primaryEmail, additionalEmails }` | `emails` on `person` |
| `PHONES` | Composite: `{ primaryPhoneNumber, ... }` | `phones` on `person` |
| `LINKS` | Composite: `{ primaryLinkUrl, primaryLinkLabel }` | `linkedinLink`, `xLink` |
| `ADDRESS` | Composite address object | `city`, `street`, etc. |
| `NUMBER` | Integer or decimal | `annualRecurringRevenue` |
| `NUMERIC` | Precise decimal | financial values |
| `CURRENCY` | Composite: `{ amountMicros, currencyCode }` | `amount` on `opportunity` |
| `BOOLEAN` | true / false | `isDeal` |
| `DATE` | Date only (no time) | `birthDate` |
| `DATE_TIME` | Full ISO timestamp | `createdAt`, `closeDate` |
| `SELECT` | Single enum from options list | `industry`, `stage` |
| `STATUS` | Single enum with status semantics | `status` on `task` |
| `MULTI_SELECT` | Array of enum values | `technologies` |
| `RELATION` | Foreign key to another object | `company`, `assignee` |
| `UUID` | Raw UUID | `id` |
| `RATING` | Star rating (1–5) | `performanceRating` |
| `ARRAY` | Array of strings | `tags` |
| `ACTOR` | Who created/updated a record | `createdBy` |
| `RICH_TEXT` | HTML rich text | `body` on `note` |
| `RICH_TEXT_V2` | Block-based rich text | newer body fields |
| `RAW_JSON` | Arbitrary JSON | `trigger`, `steps` on versions |
| `POSITION` | Float for drag-order | `position` |
| `TS_VECTOR` | Full-text search index | internal use |
| `MORPH_RELATION` | Polymorphic relation | internal use |

---

## Get a Single Object's Full Field List (Practical Query)

Use this to get all `fieldMetadataId`s for building FIND_RECORDS filters or BULK action conditions:

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { objects(filter: { nameSingular: { eq: \"person\" } }, paging: { first: 1 }) { edges { node { fieldsList { id name label type options } } } } }"
  }' | jq '.data.objects.edges[0].node.fieldsList[] | select(.name) | { id, name, label, type }'
```

---

## List All Objects with Their Fields (Full Dump)

Used when you need to browse all available entity types and their fields at once — this is the same query the UI uses to populate its object/field pickers:

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { objects(paging: { first: 1000 }) { edges { node { id nameSingular namePlural labelSingular labelPlural isCustom isActive isSystem fieldsList { id name label type isActive isSystem isCustom options settings objectMetadataId } } } } }"
  }'
```

---

## Fetch Connected Accounts (for SEND_EMAIL / SEND_WHATSAPP / SEND_LINKEDIN)

Connected accounts are regular CRM records, not metadata — fetch them via the REST API:

```bash
curl -s -G "https://app.usedalil.ai/rest/connectedAccounts" \
  -H "Authorization: Bearer {apiKey}" \
  --data-urlencode "filter=accountOwnerId[eq]:{workspaceMemberId}" \
  --data-urlencode "limit=50"
```

Response — extract the `id` for use as `connectedAccountId` in communication actions:
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

**`provider` values:** `google` (Gmail/Google Calendar), `microsoft` (Outlook), `linkedin`, `whatsapp`

To list all connected accounts in the workspace (admin use):
```bash
curl -s -G "https://app.usedalil.ai/rest/connectedAccounts" \
  -H "Authorization: Bearer {apiKey}" \
  --data-urlencode "limit=100"
```

---

## Fetch Serverless Functions (for CODE action)

Serverless functions are queried via the **main GraphQL** endpoint (`/graphql`), not the metadata endpoint:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { findManyServerlessFunctions { id name description runtime } }"
  }'
```

Response — extract `id` and use as `serverlessFunctionId` in the CODE action:
```json
[
  {
    "id": "function-uuid",
    "name": "enrichContact",
    "description": "Enrich a contact via Clearbit",
    "runtime": "nodejs18.x"
  }
]
```

A specific function:
```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query FindOneServerlessFunction($input: ServerlessFunctionIdInput!) { findOneServerlessFunction(input: $input) { id name description runtime } }",
    "variables": { "input": { "id": "function-uuid" } }
  }'
```

---

## How fieldMetadataId Maps to Workflow Filters

When building `StepFilter` objects (used in CONDITION, FILTER, FIND_RECORDS), the `fieldMetadataId` field must contain the `id` from the field's metadata record:

```json
{
  "id": "filter-uuid",
  "type": "LEAF",
  "stepOutputKey": "",
  "fieldMetadataId": "field-metadata-uuid",
  "operand": "eq",
  "value": "{{trigger.city}}",
  "filterValueMode": "VARIABLE",
  "stepFilterGroupId": "group-uuid",
  "isFullRecord": false,
  "isPipelineField": false
}
```

**Workflow:** query the object's `fieldsList` → pick the field by `name` → copy its `id` as `fieldMetadataId` in the filter.

---

## Endpoint Summary

| What you need | Endpoint | Method |
|---|---|---|
| All objects (names, labels) | `/metadata` GraphQL — `objects` query | POST |
| Fields for an object | `/metadata` GraphQL — `objects(filter: { nameSingular })` with `fieldsList` | POST |
| Connected account IDs | `/rest/connectedAccounts` | GET |
| Serverless function IDs | `/graphql` GraphQL — `findManyServerlessFunctions` | POST |
| Workspace member IDs | `/rest/workspaceMembers` | GET |

---

## Gotchas

1. **Metadata endpoint is `/metadata`, not `/graphql`** — Sending object/field queries to `/graphql` will fail. The `/graphql` endpoint handles workflow mutations and serverless functions; `/metadata` handles object and field schema.
2. **Use `fieldsList`, not `fields`** — The subfield for listing an object's fields is `fieldsList`. Using `fields` as a connection returns a paginated `CursorConnection` shape and requires different pagination syntax.
3. **`fieldMetadataId` is the field's `id`, not its `name`** — Step filters require the UUID from `fieldsList[].id`, not the string name. If you put the name there, the filter will silently fail or error at runtime.
4. **Filter `isActive: true`** — Deleted or deactivated fields still appear in the response. Always filter them out before presenting options or using them in filters.
5. **Custom objects have the same API** — Custom objects created by the user appear alongside standard ones. Their `nameSingular` is whatever the user set (e.g. `project`, `contract`). Use the same query.
6. **SELECT/STATUS/MULTI_SELECT fields have `options`** — To get the valid values for enum fields, read `options` from the field response. Each option has `value` (the key to use in filters/payloads) and `label` (the display name).
7. **RELATION fields point to another object** — A RELATION field's `objectMetadataId` tells you which object it points to. The field `name` becomes the nested key in record payloads (e.g. setting `companyId` on a `person` record).
8. **Connected accounts are per workspace member** — A `connectedAccountId` belongs to a specific `workspaceMember`. In automations, use an account owned by the automation's service user or the account set up for the workspace.
9. **Serverless functions use `/graphql`, not `/metadata`** — `findManyServerlessFunctions` and `findOneServerlessFunction` are on the main GraphQL endpoint, not the metadata endpoint.
10. **COMPOSITE fields have sub-paths in variables** — Fields of type `FULL_NAME`, `EMAILS`, `PHONES`, `LINKS`, `CURRENCY`, `ADDRESS` return nested objects. Access them as `{{trigger.name.firstName}}`, `{{trigger.emails.primaryEmail}}`, `{{trigger.amount.amountMicros}}` etc. in workflow variable expressions.
