---
name: relation
description: Create and delete relation fields between two Dalil AI objects using the RelationMetadata API — covers relation types (ONE_TO_ONE, ONE_TO_MANY, MANY_TO_ONE), required payload fields, the mandatory name validation and object relation-readiness checks that prevent CRM crashes, and all relation-specific gotchas.
---

# Dalil AI: Relation Field Creation API Skills

## Quick Reference

- **Base URL:** `https://api.usedalil.ai/rest/metadata`
- **Auth:** `Authorization: Bearer {apiKey}`
- **Content-Type:** `application/json`
- **Resource path:** `/rest/metadata/relations`

> **This skill is for creating RELATION-type fields between objects.** It uses a different API endpoint than regular field creation (`/rest/metadata/fields`). Do NOT use the fields endpoint for relation fields — it will be rejected.

---

## Endpoints

| Operation | Method | Path | Notes |
|-----------|--------|------|-------|
| Create a relation | POST | `/rest/metadata/relations` | Main creation endpoint |
| Get one relation | GET | `/rest/metadata/relations/{id}` | Inspect an existing relation |
| Delete a relation | DELETE | `/rest/metadata/relations/{id}` | Removes field from both objects — irreversible |

---

## Pre-Flight: Gather Requirements Before Acting

**Complete every step before making any API call. If anything is missing or fails validation, stop and ask the user.**

### Step 1 — Identify both objects

Ask if not already known: *"Which two objects should be linked? (e.g., Person → Company, Person → Test)"*

Verify both exist:
```http
GET /rest/metadata/objects HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
```

Confirm both `fromObject` and `toObject` appear in the response with `isActive: true`. If either is missing or inactive, stop and tell the user.

### Step 2 — Confirm relation type

Ask if not stated: *"Should this be one-to-many, many-to-one, or one-to-one?"*

| Type | Meaning |
|------|---------|
| `ONE_TO_MANY` | One `from` record links to many `to` records |
| `MANY_TO_ONE` | Many `from` records link to one `to` record (mirror of ONE_TO_MANY) |
| `ONE_TO_ONE` | One `from` record links to exactly one `to` record |

### Step 3 — Confirm field names and labels

You need four values:
- `fromName` — camelCase field name placed on the `from` object
- `fromLabel` — display label for the field on the `from` side
- `toName` — camelCase field name placed on the `to` object (the reverse/back-reference field)
- `toLabel` — display label for the field on the `to` side

If the user only gives you one side (e.g., "call it testRelation on Person"), derive the reverse name by suggestion and confirm: *"I'll name the reverse field on Test `linkedPeople` — does that work?"*

### Step 4 — Run name validation (MANDATORY)

**This step must not be skipped. Invalid names silently break the CRM schema.**

Run the following validation for **both** `fromName` (against `fromObject`) and `toName` (against `toObject`):

#### 4a. Check for existing field name conflicts

```http
GET /rest/metadata/objects/{objectId} HTTP/1.1
```

Collect every `name` value from the `fields` array. The proposed name must not appear in this list.

#### 4b. Check against all object singular and plural names

```http
GET /rest/metadata/objects HTTP/1.1
```

Collect every object's `nameSingular` and `namePlural`. The proposed name must not match any of them.

> **This is the exact cause of the `people` incident.** `people` was `namePlural` of the Person object. Using it as a field name on Test caused a GraphQL schema collision that broke the relation immediately after creation.

#### 4c. Check against reserved field names

The proposed name must not be any of these:

**Core system:** `user`, `users`, `workspace`, `workspaces`, `role`, `roles`, `event`, `events`, `field`, `fields`, `object`, `objects`, `pipeline`, `pipelines`, `relation`, `relations`, `link`, `links`, `currency`, `currencies`, `fullName`, `fullNames`, `address`, `addresses`, `type`, `types`, `index`

**GraphQL keywords:** `one`, `many`, `input`, `args`, `data`, `filter`, `sort`, `pagination`, `connection`, `edge`, `node`, `query`, `mutation`, `subscription`

#### 4d. Check camelCase format

The name must:
- Start with a **lowercase letter**
- Contain only alphanumeric characters (no underscores, dashes, or spaces)
- Be 63 characters or fewer

#### Validation result

If **any check fails** for either name:
- Stop immediately
- Tell the user exactly which check failed and what it clashed with (e.g., *"`people` matches the `namePlural` of the Person object — please choose a different name for the reverse field on Test"*)
- Do NOT proceed until the user provides a valid replacement

If all checks pass for both names, proceed to Step 5.

### Step 5 — Verify both objects can support relations (MANDATORY)

> **This step prevents a server-side crash that breaks the entire CRM.** Creating a relation on an object that cannot support it returns `"Cannot return null for non-nullable field RelationMetadata.id"` — a platform-level error that corrupts the schema for both objects even though the API accepted the request.

For **each** object (`fromObject` and `toObject`), verify it already has at least one existing RELATION-type field:

```http
GET /rest/metadata/objects/{objectId} HTTP/1.1
```

In the response, check `fields[]` for any entry where `"type": "RELATION"`. Standard objects (Person, Company, Opportunity, Note, Task) always have these. Custom objects may not if they were just created or are structurally minimal.

**If an object has zero RELATION-type fields:**
- Do NOT proceed with relation creation.
- Tell the user: *"The [object name] object does not appear to support custom relations yet — creating one causes a platform error that crashes the CRM. Please contact Dalil support to confirm this object can have relations added before proceeding."*

**If both objects have at least one existing RELATION field:** proceed to Step 6.

### Step 6 — Confirm before creating

Summarize and ask for approval:

> *"I'll create a **[relationType]** relation:*
> *- Field `[fromName]` ("[fromLabel]") on **[fromObject]***
> *- Field `[toName]` ("[toLabel]") on **[toObject]***
>
> *Shall I go ahead?"*

Only proceed after confirmation.

---

## Create a Relation

### Endpoint

```http
POST /rest/metadata/relations HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
Content-Type: application/json
```

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `relationType` | string | `ONE_TO_ONE`, `ONE_TO_MANY`, or `MANY_TO_ONE` |
| `fromObjectMetadataId` | UUID | ID of the object the `fromName` field is added to |
| `toObjectMetadataId` | UUID | ID of the object the `toName` field is added to |
| `fromName` | string | camelCase field name on the `from` object — must pass name validation |
| `toName` | string | camelCase field name on the `to` object — must pass name validation |
| `fromLabel` | string | Display label on the `from` object |
| `toLabel` | string | Display label on the `to` object |

### Example Request

```json
{
  "relationType": "MANY_TO_ONE",
  "fromObjectMetadataId": "4d973ba5-a8a7-47af-8684-a1900fe816db",
  "toObjectMetadataId": "082a44a8-511c-44a7-8719-71bef8c16ebc",
  "fromName": "linkedTest",
  "toName": "linkedPeople",
  "fromLabel": "Linked Test",
  "toLabel": "Linked People"
}
```

### Example Response

```json
{
  "data": {
    "createOneRelationMetadata": {
      "id": "adf0b4e6-20fe-445b-9a22-33b252b82681",
      "relationType": "MANY_TO_ONE",
      "fromFieldMetadataId": "904678c5-d814-442f-8cbc-bad49c7e0ff1",
      "toFieldMetadataId": "7b02e5f4-af5c-4d68-919f-cbc482df5ea9",
      "fromObjectMetadata": {
        "id": "4d973ba5-a8a7-47af-8684-a1900fe816db",
        "nameSingular": "person",
        "namePlural": "people"
      },
      "toObjectMetadata": {
        "id": "082a44a8-511c-44a7-8719-71bef8c16ebc",
        "nameSingular": "test",
        "namePlural": "tests"
      }
    }
  }
}
```

Save the returned `id` — you'll need it to delete the relation if required.

---

## Delete a Relation

```http
DELETE /rest/metadata/relations/{relationId} HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
```

### Response

```json
{
  "data": {
    "deleteOneRelation": {
      "id": "adf0b4e6-20fe-445b-9a22-33b252b82681"
    }
  }
}
```

**Note:** Deletion removes the field from **both** objects simultaneously. It is irreversible — there is no soft-delete.

---

## Gotchas

1. **`fromName`/`toName` must not match any object's `nameSingular` or `namePlural`** — this is the highest-risk mistake. The API may accept the payload, but the relation silently becomes invalid and breaks the GraphQL schema for the entire workspace. Always run Step 4b validation. Example of what went wrong: using `people` as `toName` clashed with Person's `namePlural` and corrupted the relation on creation.

2. **The API can accept an invalid name without returning an error** — schema corruption is silent. There is no error at POST time; the damage only becomes visible when the CRM breaks. Name validation in pre-flight is your only protection.

3. **Deleting a relation removes the field from both sides** — there is no way to remove just one side. Both `fromName` on `fromObject` and `toName` on `toObject` are deleted together.

4. **Both objects must be `isActive: true` and `isSystem: false`** — system objects (e.g., `auditLog`, `timelineActivity`) are not valid targets for custom relations.

5. **`fromName` and `toName` must be unique on their respective objects** — same uniqueness constraint as regular field creation. Colliding with an existing field name returns a `"duplicate key value violates unique constraint"` error.

6. **Reserved field names apply here too** — the full reserved list from the field skill applies to relation field names. Common traps: `relation`, `relations`, `link`, `links`, `type`, `data`, `filter`.

7. **camelCase rules are identical to regular fields** — `fromName` and `toName` must start lowercase, contain only alphanumeric characters, and be 63 characters or fewer.

8. **`MANY_TO_ONE` and `ONE_TO_MANY` are mirrors** — creating `MANY_TO_ONE` from A to B is the same schema result as `ONE_TO_MANY` from B to A. The directionality determines which object gets the FK-side field.

9. **CRITICAL — Objects without existing relations cause a server crash that breaks the CRM.** If either object has no RELATION-type fields, the API returns `"Cannot return null for non-nullable field RelationMetadata.id"` — a platform bug where the relation record fails to persist server-side. Despite this, the schema is partially mutated, causing `"Invalid relation type"` errors on both objects' REST endpoints until the relation is deleted. Always run Step 5 before creating. Custom objects created with minimal configuration are the most common trigger. Standard objects (Person, Company, Opportunity, Note, Task) are safe — they always have built-in relations.

10. **If a crash does occur, delete the relation immediately.** Use `DELETE /rest/metadata/relations/{id}` with the ID from the failed POST response (if one was returned) or by listing relations filtered to the affected objects. Do not leave a broken relation in place — it will keep the REST endpoints for both objects in a broken state.
