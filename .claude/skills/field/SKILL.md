---
name: field
description: Create, update, and delete custom fields on Dalil AI objects and pipelines — covers all field types (TEXT, NUMBER, BOOLEAN, CURRENCY, SELECT, MULTI_SELECT, DATE, DATE_TIME, EMAILS, PHONES, LINKS, FULL_NAME, ADDRESS, RATING, ARRAY, RAW_JSON, UUID, RICH_TEXT_V2, ACTOR), naming rules, type-specific settings, SELECT options format, composite default values, Object vs Pipeline scoping, update constraints (name/type are immutable, options replace-all semantics), and delete safety rules (system fields, relation fields, irreversibility).
---

# Dalil AI: Field Creation API Skills

## Quick Reference

- **Base URL:** `https://api.usedalil.ai/rest/metadata`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*

**Critical:** Fields belong to either an **Object** (`objectMetadataId`) or a **Pipeline** (`pipelineId`). You must resolve this parent before creating a field. Never send both. Never omit both.

## Endpoints

| Operation | Method | Path | Notes |
|-----------|--------|------|-------|
| List all objects | GET | `/rest/metadata/objects` | Find objectMetadataId |
| Get one object | GET | `/rest/metadata/objects/{id}` | Confirm object exists |
| List all pipelines | GET | `/rest/metadata/pipelines` | Find pipelineId |
| Get one pipeline | GET | `/rest/metadata/pipelines/{id}` | Confirm pipeline + fields |
| List all fields | GET | `/rest/metadata/fields` | Check existing field names |
| Get one field | GET | `/rest/metadata/fields/{id}` | Inspect a specific field |
| **Create a field** | POST | `/rest/metadata/fields` | Main creation endpoint |
| Update a field | PATCH | `/rest/metadata/fields/{id}` | Modify label, description, options |
| Delete a field | DELETE | `/rest/metadata/fields/{id}` | Irreversible |

---

## Pre-Flight: Gather Requirements Before Acting

**Before making any API call, verify you have all required inputs. If anything is missing, ask the user — do not assume or proceed with placeholders.**

### Required inputs checklist

Work through this checklist top-to-bottom. Stop and ask as soon as you find a gap.

#### 1. Field label / name
- Do you have a clear label for the field (e.g., "Priority", "Lead Source")?
- If yes, derive `name` from it (camelCase). If no, ask: *"What should this field be called?"*

#### 2. Field type
- Is the type explicitly stated (e.g., "a dropdown", "a number", "a date")?
- If ambiguous, map intent to the closest type and confirm: *"I'll create this as a SELECT field — is that right?"*
- If not stated at all, ask: *"What type of field should this be — text, number, dropdown (select), date, boolean, etc.?"*

#### 3. Target entity (object or pipeline) — ask if not specified
- Which object or pipeline should this field be added to?
- Common objects: **Person**, **Company**, **Opportunity**, **Note**, **Task**
- If not specified, ask: *"Which object or pipeline should this field be added to — Person, Company, Opportunity, or a custom pipeline?"*
- Never assume Person just because it was the last entity mentioned.

#### 4. Type-specific prerequisites — ask before proceeding

| Field type | What to ask if missing |
|------------|------------------------|
| **SELECT** | *"What are the dropdown options? Please provide each option's label and optionally a color (green, blue, red, orange, yellow, purple, pink, gray, turquoise, sky)."* |
| **MULTI_SELECT** | Same as SELECT — options are required |
| **NUMBER** | *"Should this be a plain number or a percentage? How many decimal places?"* |
| **CURRENCY** | *"What currency should be the default — USD, EUR, GBP, or other?"* |
| **BOOLEAN** | *"What should the default be — true or false?"* (optional, but good to ask) |
| **DATE / DATE_TIME** | *"Should this auto-fill with today's date/time when a record is created, or stay empty by default?"* |
| **RICH_TEXT_V2** | No extra inputs needed |
| **LINKS / EMAILS / PHONES / FULL_NAME / ADDRESS / ARRAY / RAW_JSON / UUID / RATING / TEXT** | No extra inputs needed |

#### 5. Confirm before creating
Once you have all inputs, summarize what you're about to create and ask for confirmation:

> *"I'll create a **[type]** field called **[label]** (`[name]`) on the **[entity]** object[, with options: X, Y, Z]. Shall I go ahead?"*

Only proceed after the user confirms.

---

## Step 1: Resolve the Parent (Required Before Creating)

### Option A — Object Field

Use when adding a field to a standard or custom object (Person, Company, Opportunity, etc.).

```http
GET /rest/metadata/objects HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
```

Response (abbreviated):
```json
{
  "data": {
    "objects": [
      {
        "id": "4d973ba5-a8a7-47af-8684-a1900fe816db",
        "nameSingular": "person",
        "labelSingular": "Person",
        "isCustom": false,
        "isActive": true
      }
    ]
  }
}
```

Use the `id` as `objectMetadataId` in your field creation payload.

### Option B — Pipeline Field

Use when adding a field scoped to a specific pipeline (e.g., "Customer Prospect").

```http
GET /rest/metadata/pipelines HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
```

Response (abbreviated):
```json
{
  "data": {
    "pipelines": [
      {
        "id": "e6a23049-bdc4-4257-bd10-add82ed23c04",
        "nameSingular": "customerProspect",
        "labelSingular": "Customer Prospect",
        "objectMetadataId": "4d973ba5-...",
        "fields": []
      }
    ]
  }
}
```

Use the `id` as `pipelineId` in your field creation payload.

---

## Step 2: Check for Name Conflicts

Before creating, list existing fields on the target object or pipeline to avoid duplicate name errors.

**`GET /rest/metadata/fields?filter=objectMetadataId[eq]:{objectMetadataId}` does not scope to the object** — despite the filter, it returns a global union of every field in the workspace (thousands of rows). Do not rely on it to check one object's fields. Instead use:

```http
GET /rest/metadata/objects/{objectMetadataId} HTTP/1.1
```

and read the embedded `fields[]` array, or query the `/metadata` GraphQL endpoint's `fieldsList` (see the `workflow-metadata` skill for the exact query).

Scan `name` values in the response. Your new field's `name` must not already exist on that object/pipeline — and if you're creating a **pipeline-scoped** field, it must also not already exist on the pipeline's **parent object** (see gotcha #12 below).

---

## Field Naming Rules

The `name` field is the internal identifier. It must:

- Be **camelCase** — e.g., `myCustomField`, `dealScore`, `linkedinUrl`
- Start with a **lowercase letter** only
- Contain only alphanumeric characters (no underscores, dashes, spaces)
- Be **63 characters or fewer** (database identifier limit)
- Be **unique** within the same object or pipeline
- **Not be a reserved name** (see list below)

**Auto-deriving `name` from `label`:** Take the label, lowercase each word, join in camelCase.
- `"My Field"` → `myField`
- `"LinkedIn URL"` → `linkedinUrl`
- `"Deal Score"` → `dealScore`

**Reserved names that will cause errors:**

Core system: `user`, `users`, `workspace`, `workspaces`, `role`, `roles`, `event`, `events`, `field`, `fields`, `object`, `objects`, `pipeline`, `pipelines`, `relation`, `relations`, `link`, `links`, `currency`, `currencies`, `fullName`, `fullNames`, `address`, `addresses`, `type`, `types`, `index`

GraphQL keywords: `one`, `many`, `input`, `args`, `data`, `filter`, `sort`, `pagination`, `connection`, `edge`, `node`, `query`, `mutation`, `subscription`

---

## Step 3: Create a Field

### Endpoint

```http
POST /rest/metadata/fields HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
Content-Type: application/json
```

### Required Parameters (all types)

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Field type enum (see Field Types Reference) |
| `name` | string | camelCase internal name — unique per object/pipeline |
| `label` | string | Display label shown in the UI |
| `objectMetadataId` | UUID | Parent object — **required for object fields, omit for pipeline fields** |
| `pipelineId` | UUID | Parent pipeline — **required for pipeline fields, omit for object fields** |

### Optional Parameters (all types)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `description` | string | `""` | Human-readable description |
| `icon` | string | varies | Icon name (e.g., `IconTypography`, `IconNumber`) |
| `defaultValue` | any | varies | Type-specific default value |
| `isNullable` | boolean | `true` | Whether field can be null |
| `isUnique` | boolean | `false` | Creates unique partial index (TEXT, NUMBER, UUID only) |
| `isIndexed` | boolean | `false` | Creates database index |
| `settings` | object | `{}` | Type-specific settings (required for NUMBER) |
| `options` | array | `null` | Required for SELECT and MULTI_SELECT only |

### Response

```json
{
  "data": {
    "createOneField": {
      "id": "uuid-of-created-field",
      "type": "TEXT",
      "name": "myField",
      "label": "My Field",
      "objectMetadataId": "...",
      "isCustom": true,
      "isActive": true,
      "createdAt": "2025-01-01T00:00:00.000Z"
    }
  }
}
```

---

## Field Types Reference

### TEXT

Short single-line text.

| Property | Value |
|----------|-------|
| Default icon | `IconTypography` |
| `defaultValue` | `"''"` (empty string — note single-quote wrapping) |
| `isNullable` | `false` recommended |
| `settings` | Not required |

```json
{
  "type": "TEXT",
  "name": "linkedinUrl",
  "label": "LinkedIn URL",
  "description": "The person's LinkedIn profile URL",
  "icon": "IconBrandLinkedin",
  "objectMetadataId": "4d973ba5-a8a7-47af-8684-a1900fe816db",
  "defaultValue": "''",
  "isNullable": false
}
```

---

### NUMBER

Integer or decimal numeric value.

| Property | Value |
|----------|-------|
| Default icon | `IconNumber` |
| `defaultValue` | `null` |
| `isNullable` | `true` |
| `settings` | **Required:** `{ "decimals": 0, "type": "number" }` |

`settings.type` options: `"number"` (standard) or `"percentage"` (display as %).

```json
{
  "type": "NUMBER",
  "name": "employeeCount",
  "label": "Employee Count",
  "icon": "IconNumber",
  "objectMetadataId": "4d973ba5-...",
  "settings": {
    "decimals": 0,
    "type": "number"
  }
}
```

Decimal example:
```json
{
  "type": "NUMBER",
  "name": "conversionRate",
  "label": "Conversion Rate",
  "objectMetadataId": "4d973ba5-...",
  "settings": {
    "decimals": 2,
    "type": "percentage"
  }
}
```

---

### BOOLEAN

True/false toggle.

| Property | Value |
|----------|-------|
| Default icon | `IconCheckbox` |
| `defaultValue` | `false` |
| `isNullable` | `true` |

```json
{
  "type": "BOOLEAN",
  "name": "isVerified",
  "label": "Is Verified",
  "icon": "IconCheckbox",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": false
}
```

---

### DATE_TIME

Date and time (ISO 8601).

| Property | Value |
|----------|-------|
| Default icon | `IconCalendar` |
| `defaultValue` | `null` or `"now"` (auto-fills on record creation) |
| `isNullable` | `true` |

```json
{
  "type": "DATE_TIME",
  "name": "lastContactedAt",
  "label": "Last Contacted At",
  "icon": "IconCalendar",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": null
}
```

---

### DATE

Date-only (no time component).

| Property | Value |
|----------|-------|
| Default icon | `IconCalendar` |
| `defaultValue` | `null` |
| `isNullable` | `true` |

```json
{
  "type": "DATE",
  "name": "birthDate",
  "label": "Birth Date",
  "icon": "IconCalendar",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": null
}
```

---

### CURRENCY

Monetary amount with currency code. Composite field.

| Property | Value |
|----------|-------|
| Default icon | `IconCurrencyDollar` |
| `defaultValue` | `{ "amountMicros": null, "currencyCode": "'USD'" }` |
| `isNullable` | `true` |

**Important:** amounts are stored in **micros** (1 USD = 1,000,000 micros). Currency code string defaults use single-quote wrapping: `"'USD'"`.

```json
{
  "type": "CURRENCY",
  "name": "dealValue",
  "label": "Deal Value",
  "icon": "IconCurrencyDollar",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": {
    "amountMicros": null,
    "currencyCode": "'USD'"
  }
}
```

---

### SELECT

Single-option dropdown. **Requires `options` array.**

| Property | Value |
|----------|-------|
| Default icon | `IconTag` |
| `defaultValue` | `null` or `"'OPTION_VALUE'"` (single-quote wrapped) |
| `isNullable` | `true` |
| `options` | **Required** |

**Options schema:**

| Property | Required | Description |
|----------|----------|-------------|
| `label` | Yes | Display text |
| `value` | Yes | Internal value — `UPPER_SNAKE_CASE`, max ~20 chars, `[A-Z0-9_]+` only |
| `color` | Yes | One of: `green`, `turquoise`, `sky`, `blue`, `purple`, `pink`, `red`, `orange`, `yellow`, `gray` |
| `position` | Yes | Display order (0-indexed) |
| `id` | No | UUID — auto-generated if omitted |

```json
{
  "type": "SELECT",
  "name": "priority",
  "label": "Priority",
  "icon": "IconTag",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": "'MEDIUM'",
  "options": [
    { "label": "Low", "value": "LOW", "color": "gray", "position": 0 },
    { "label": "Medium", "value": "MEDIUM", "color": "orange", "position": 1 },
    { "label": "High", "value": "HIGH", "color": "red", "position": 2 }
  ]
}
```

---

### MULTI_SELECT

Multi-option tag field. Same options format as SELECT, allows multiple values selected.

| Property | Value |
|----------|-------|
| Default icon | `IconTag` |
| `defaultValue` | `null` |
| `options` | **Required** |

```json
{
  "type": "MULTI_SELECT",
  "name": "tags",
  "label": "Tags",
  "icon": "IconTag",
  "objectMetadataId": "4d973ba5-...",
  "options": [
    { "label": "VIP", "value": "VIP", "color": "purple", "position": 0 },
    { "label": "Partner", "value": "PARTNER", "color": "blue", "position": 1 },
    { "label": "Churned", "value": "CHURNED", "color": "red", "position": 2 }
  ]
}
```

---

### EMAILS

Email address field. Composite type.

| Property | Value |
|----------|-------|
| Default icon | `IconMail` |
| `defaultValue` | `{ "primaryEmail": "''", "additionalEmails": null }` |
| `isNullable` | `true` |

```json
{
  "type": "EMAILS",
  "name": "workEmails",
  "label": "Work Emails",
  "icon": "IconMail",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": {
    "primaryEmail": "''",
    "additionalEmails": null
  }
}
```

---

### PHONES

Phone number field. Composite type.

| Property | Value |
|----------|-------|
| Default icon | `IconPhone` |
| `defaultValue` | `{ "primaryPhoneNumber": "''", "primaryPhoneCountryCode": "''" }` |
| `isNullable` | `true` |

```json
{
  "type": "PHONES",
  "name": "mobilePhone",
  "label": "Mobile Phone",
  "icon": "IconPhone",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": {
    "primaryPhoneNumber": "''",
    "primaryPhoneCountryCode": "''"
  }
}
```

---

### LINKS

URL/website links field. Composite type.

| Property | Value |
|----------|-------|
| Default icon | `IconLink` |
| `defaultValue` | `{ "primaryLinkUrl": "''", "primaryLinkLabel": "''" }` |
| `isNullable` | `true` |

```json
{
  "type": "LINKS",
  "name": "websiteUrl",
  "label": "Website URL",
  "icon": "IconLink",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": {
    "primaryLinkUrl": "''",
    "primaryLinkLabel": "''"
  }
}
```

---

### FULL_NAME

First and last name. Composite type.

| Property | Value |
|----------|-------|
| Default icon | `IconUser` |
| `defaultValue` | `{ "firstName": "''", "lastName": "''" }` |
| `isNullable` | `true` |

```json
{
  "type": "FULL_NAME",
  "name": "contactName",
  "label": "Contact Name",
  "icon": "IconUser",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": {
    "firstName": "''",
    "lastName": "''"
  }
}
```

---

### ADDRESS

Full postal address. Composite type.

| Property | Value |
|----------|-------|
| Default icon | `IconMap` |
| `defaultValue` | All sub-fields empty strings (see example) |
| `isNullable` | `true` |

```json
{
  "type": "ADDRESS",
  "name": "officeAddress",
  "label": "Office Address",
  "icon": "IconMap",
  "objectMetadataId": "4d973ba5-...",
  "defaultValue": {
    "addressStreet1": "''",
    "addressStreet2": "''",
    "addressCity": "''",
    "addressState": "''",
    "addressPostcode": "''",
    "addressCountry": "''"
  }
}
```

---

### RATING

Star rating field. Do NOT send `options` — they are auto-generated.

| Property | Value |
|----------|-------|
| Default icon | `IconStar` |
| `defaultValue` | `null` |
| `isNullable` | `true` |

```json
{
  "type": "RATING",
  "name": "satisfactionScore",
  "label": "Satisfaction Score",
  "icon": "IconStar",
  "objectMetadataId": "4d973ba5-..."
}
```

---

### ARRAY

List of string values.

| Property | Value |
|----------|-------|
| Default icon | `IconList` |
| `defaultValue` | `null` |
| `isNullable` | `true` |

```json
{
  "type": "ARRAY",
  "name": "industries",
  "label": "Industries",
  "icon": "IconList",
  "objectMetadataId": "4d973ba5-..."
}
```

---

### RAW_JSON

Structured JSON data.

| Property | Value |
|----------|-------|
| Default icon | `IconBraces` |
| `defaultValue` | `null` |
| `isNullable` | `true` |

```json
{
  "type": "RAW_JSON",
  "name": "metadata",
  "label": "Metadata",
  "icon": "IconBraces",
  "objectMetadataId": "4d973ba5-..."
}
```

---

### UUID

Unique identifier field.

| Property | Value |
|----------|-------|
| Default icon | `IconKey` |
| `defaultValue` | `null` |
| `isNullable` | `true` |
| Supports constraints | `isUnique`, `isIndexed` |

```json
{
  "type": "UUID",
  "name": "externalId",
  "label": "External ID",
  "icon": "IconKey",
  "objectMetadataId": "4d973ba5-..."
}
```

---

### RICH_TEXT_V2

Formatted rich text (BlockNote + Markdown). Use for long-form content.

| Property | Value |
|----------|-------|
| Default icon | `IconCreativeCommons` |
| `defaultValue` | `null` |
| `isNullable` | `true` |
| `settings` | `{ "displayedMaxRows": 0 }` recommended |

```json
{
  "type": "RICH_TEXT_V2",
  "name": "bio",
  "label": "Bio",
  "icon": "IconCreativeCommons",
  "objectMetadataId": "4d973ba5-...",
  "settings": {
    "displayedMaxRows": 0
  }
}
```

---

### ACTOR

Tracks who performed an action — stores source, name, and context. Used for assigned user or owner fields.

| Property | Value |
|----------|-------|
| Default icon | `IconUsers` |
| `defaultValue` | `{ "source": "'MANUAL'", "name": "'System'", "context": {} }` |
| `isNullable` | `true` |
| `settings` | Not required |

**Important:** `actor` is a reserved name on any object that already has an actor field (e.g., Person). Use a scoped name like `pipelineActor`, `assignedActor` to avoid conflicts.

```json
{
  "type": "ACTOR",
  "name": "pipelineActor",
  "label": "Actor",
  "icon": "IconUsers",
  "description": "Use for assigned user or owner.",
  "pipelineId": "4f45acb6-b824-4bc7-855f-15b8ad8f25dc"
}
```

---

## Object vs Pipeline Field Creation

| Aspect | Object Field | Pipeline Field |
|--------|-------------|----------------|
| Identifier to send | `objectMetadataId` | `pipelineId` |
| Scope | All views of the object | Scoped to that pipeline only |
| Omit the other | Omit `pipelineId` entirely | Omit `objectMetadataId` entirely |
| Name uniqueness | Unique among the object's own fields | Unique among the pipeline's fields **AND** the pipeline's parent object's fields (shared namespace — see gotcha #12) |

**Object field example:**
```json
{
  "type": "TEXT",
  "name": "nickname",
  "label": "Nickname",
  "objectMetadataId": "4d973ba5-a8a7-47af-8684-a1900fe816db"
}
```

**Pipeline field example:**
```json
{
  "type": "TEXT",
  "name": "nickname",
  "label": "Nickname",
  "pipelineId": "e6a23049-bdc4-4257-bd10-add82ed23c04"
}
```

---

## Constraints (Unique / Indexed)

Available for: **TEXT**, **NUMBER**, **UUID** only.

When `isUnique: true`:
- A unique partial index is created in the database
- `isNullable` is automatically forced to `true` (allows multiple NULLs, prevents duplicate non-NULLs)
- `defaultValue` should be `null`

```json
{
  "type": "TEXT",
  "name": "externalRef",
  "label": "External Reference",
  "objectMetadataId": "4d973ba5-...",
  "isUnique": true,
  "isNullable": true,
  "defaultValue": null
}
```

---

## Default Icons by Field Type

| Field Type | Default Icon |
|------------|-------------|
| TEXT | `IconTypography` |
| RICH_TEXT_V2 | `IconCreativeCommons` |
| NUMBER | `IconNumber` |
| BOOLEAN | `IconCheckbox` |
| CURRENCY | `IconCurrencyDollar` |
| DATE_TIME | `IconCalendar` |
| DATE | `IconCalendar` |
| EMAILS | `IconMail` |
| PHONES | `IconPhone` |
| LINKS | `IconLink` |
| SELECT | `IconTag` |
| MULTI_SELECT | `IconTag` |
| ADDRESS | `IconMap` |
| FULL_NAME | `IconUser` |
| RATING | `IconStar` |
| ARRAY | `IconList` |
| RAW_JSON | `IconBraces` |
| UUID | `IconKey` |
| ACTOR | `IconUsers` |

---

## Gotchas

1. **`name` must be camelCase** — API rejects snake_case (`my_field`), spaces (`my field`), or starting uppercase (`MyField`). Always derive from label.

2. **Never send both `objectMetadataId` and `pipelineId`** — causes `INVALID_INPUT`. Pick exactly one.

3. **Never omit both `objectMetadataId` and `pipelineId`** — also `INVALID_INPUT`. Every field needs a parent.

4. **SELECT/MULTI_SELECT requires `options`** — omitting options causes a backend crash. At minimum one option is required.

5. **NUMBER requires `settings`** — always send `{ "decimals": 0, "type": "number" }`. Omitting may produce unexpected behavior.

6. **String default values use single-quote wrapping** — `"''"` for empty string, `"'USD'"` for the string "USD". This is intentional metadata serialization, not a typo. Do NOT use `""` or `"USD"` directly.

7. **RELATION fields cannot be created here — and getting this wrong breaks the CRM.**
   - The `/rest/metadata/fields` endpoint rejects RELATION type entirely.
   - Use `POST /rest/metadata/relations` (the RelationMetadata API) instead.
   - **Critical:** the `fromName` and `toName` you choose must NOT match any object's `nameSingular` or `namePlural` across the entire workspace — doing so causes a GraphQL schema collision that silently invalidates the relation and can break CRM functionality. Always run the name validation sub-routine in the `relation` skill before calling the API.
   - See the **"Creating Relation Fields"** section below, or load the `relation` skill for full details.

8. **RATING options are auto-generated** — do NOT send `options` for RATING fields. They are created automatically.

9. **`isUnique: true` forces `isNullable: true`** — the backend overrides your `isNullable` value. Unique partial indexes require NULLs to be allowed.

10. **Option `value` must be UPPER_SNAKE_CASE** — pattern `[A-Z0-9_]+`, max ~20 characters. Lowercase, spaces, or dashes will fail validation.

11. **All option values must be unique within a field** — duplicate `value` entries in `options` are rejected.

12. **Field names must be unique per object/pipeline — AND pipeline-scoped names share the parent object's namespace** — API returns `"duplicate key value violates unique constraint"` on collision. Always check Step 2 first. A pipeline-scoped field's `name` must not already exist on the pipeline's **parent object** either: e.g. if `person` already has a `sourceDetails` field, creating a `sourceDetails` field scoped to any `person`-based pipeline (via `pipelineId`) will fail with the same duplicate-key error, even though the pipeline itself has no `sourceDetails` field yet.

13. **Reserved names will fail** — see the Reserved Names list above. These include common words like `type`, `data`, `filter`, `index`, `address`, `links`, `currency`.

14. **Composite sub-property names are reserved** — adding an `address` field reserves `addressCity`, `addressStreet1`, etc. as names in that object. Adding `currency` reserves `amountMicros`, `currencyCode`.

15. **`defaultValue: "now"` for DATE/DATE_TIME means auto-fill, but only works on objects — not pipelines.** On object fields, `"now"` auto-populates with the current date/time when a record is created. On pipeline fields, sending `"now"` causes a `"Cannot return null for non-nullable field Field.id"` server crash. Always use `null` as the default for DATE/DATE_TIME fields on pipelines.

16. **Non-nullable fields require `defaultValue`** — if you set `isNullable: false`, you must also provide a valid `defaultValue`. The API will reject non-nullable fields without a default.

17. **`pipelineId` field: omit entirely for object fields** — do not send `"pipelineId": null`. Omit the key from the payload altogether.

---

## Updating a Field

### Endpoint

```http
PATCH /rest/metadata/fields/{fieldId} HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
Content-Type: application/json
```

To find a field's ID: the `filter=objectMetadataId[eq]:{objectId}` filter does not scope (see Step 2 above) — use `GET /rest/metadata/objects/{objectId}` and read embedded `fields[]`, or the `/metadata` GraphQL `fieldsList`, and scan for the field by `name`.

### What can be updated

| Property | Updatable | Notes |
|----------|-----------|-------|
| `label` | Yes | Safe to change — display only |
| `description` | Yes | Safe to change |
| `icon` | Yes | Safe to change |
| `defaultValue` | Yes | Does not backfill existing records |
| `options` | Yes | SELECT/MULTI_SELECT only — see rules below |
| `isActive` | Yes | Set `false` to disable without deleting |
| `settings` | Yes | NUMBER: decimals and type |
| `name` | **No** | Immutable after creation |
| `type` | **No** | Immutable after creation |
| `objectMetadataId` | **No** | Cannot move a field between objects |

### Example — rename label and change description

```json
PATCH /rest/metadata/fields/abc12345-...

{
  "label": "New Label",
  "description": "Updated description"
}
```

### Example — update SELECT options

Send the **full** options array, including existing options you want to keep. Options are replaced entirely, not merged.

```json
PATCH /rest/metadata/fields/abc12345-...

{
  "options": [
    { "id": "existing-option-uuid", "label": "Low", "value": "LOW", "color": "gray", "position": 0 },
    { "id": "existing-option-uuid-2", "label": "Medium", "value": "MEDIUM", "color": "orange", "position": 1 },
    { "id": "existing-option-uuid-3", "label": "High", "value": "HIGH", "color": "red", "position": 2 },
    { "label": "Critical", "value": "CRITICAL", "color": "pink", "position": 3 }
  ]
}
```

> **Include `id` for existing options to preserve them.** If you omit the `id`, the platform treats it as a new option. Omitting an existing option from the array removes it.

### Response

```json
{
  "data": {
    "updateOneField": {
      "id": "abc12345-...",
      "label": "New Label",
      "description": "Updated description",
      "updatedAt": "2025-06-11T00:00:00.000Z"
    }
  }
}
```

### Update gotchas

- **`name` and `type` are immutable** — the API will ignore or reject attempts to change them. If the name or type is wrong, you must delete and recreate the field.
- **Changing `defaultValue` does not backfill** — existing records keep their current values. Only new records will use the new default.
- **SELECT/MULTI_SELECT: always send the full options array** — partial updates are not supported. If you only send new options, existing options are deleted.
- **Fetch existing option IDs before updating** — use `GET /rest/metadata/fields/{id}` to retrieve the current `options` array (with `id` values) before PATCHing, so you can include the IDs and preserve existing options.
- **Disabling a field (`isActive: false`) hides it from the UI** but preserves all data. Re-enable with `isActive: true`.

---

## Deleting a Field

### Endpoint

```http
DELETE /rest/metadata/fields/{fieldId} HTTP/1.1
Host: api.usedalil.ai
Authorization: Bearer {apiKey}
```

### Finding the field ID

If you don't have the field ID, fetch the object's fields first. The `filter=objectMetadataId[eq]:{objectId}` filter does not scope (returns a global union of every field in the workspace — see Step 2 above), so use the object-detail endpoint instead:

```http
GET /rest/metadata/objects/{objectId} HTTP/1.1
```

Read the embedded `fields[]` array (or use the `/metadata` GraphQL `fieldsList`), scan for the field by `name`, then copy its `id`.

### Response

```json
{
  "data": {
    "deleteOneField": {
      "id": "abc12345-..."
    }
  }
}
```

### Delete gotchas

- **Deletion is permanent and irreversible** — all data stored in the field is lost for every record. There is no soft-delete or undo. Consider `isActive: false` (disable) instead if you might want the data later.
- **System fields cannot be deleted** — fields where `isSystem: true` or `isCustom: false` are read-only. The API returns `"Cannot delete non-custom fields"` or similar. Only fields with `isCustom: true` can be deleted.
- **Deleting a RELATION field** — do NOT delete relation fields via this endpoint. Use `DELETE /rest/metadata/relations/{relationId}` (see the `relation` skill). Deleting one side of a relation via the fields endpoint leaves the other side orphaned and may break the schema.
- **Confirm before deleting** — always tell the user what data will be lost and ask for confirmation before calling DELETE.

---

## Creating Relation Fields (RELATION type)

> **RELATION fields are not created via this endpoint.** Use `POST /rest/metadata/relations`. See the `relation` skill for the full workflow including the mandatory name validation sub-routine.

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
| `fromObjectMetadataId` | UUID | The object the relation field is placed on |
| `toObjectMetadataId` | UUID | The object being linked to |
| `fromName` | string | camelCase field name added to the `from` object |
| `toName` | string | camelCase field name added to the `to` object (reverse side) |
| `fromLabel` | string | Display label on the `from` object |
| `toLabel` | string | Display label on the `to` object |

### Example

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

### CRITICAL: Run Name Validation Before Creating

Before submitting, you MUST verify that `fromName` and `toName` do not clash with existing names. Skipping this step can silently break the CRM schema.

**For each proposed name (`fromName` on `fromObject`, `toName` on `toObject`):**

1. `GET /rest/metadata/objects/{objectId}` — collect all existing field `name` values on that object. The proposed name must not be in this list.
2. `GET /rest/metadata/objects` — collect every object's `nameSingular` and `namePlural`. The proposed name must not match any of these.
3. Check against the reserved names list in this skill. The proposed name must not be reserved.
4. The proposed name must be camelCase (start lowercase, alphanumeric only).

If any check fails, stop and ask the user for a different name. Do not proceed until all checks pass for both `fromName` and `toName`.

> For the complete workflow, delete operation, and all relation-specific gotchas, load the **`relation` skill**.
