---
name: workflow-triggers
description: Configure trigger types on a Dalil AI workflow version — set up DATABASE_EVENT (record created/updated/deleted), MANUAL (button-triggered from a record or globally), CRON (scheduled), WEBHOOK (inbound HTTP), or SEQUENCE triggers. Use after creating a workflow version with the `workflow` skill.
---

# Dalil AI: Workflow Triggers Skill

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests.
- **How triggers are set:** PATCH `/rest/workflowVersions/{versionId}` with `{ "trigger": { ... } }`
- **Output schema:** After setting a trigger, call `computeStepOutputSchema` (GraphQL) to populate `settings.outputSchema` — this enables `{{trigger.*}}` variables in downstream steps

**Critical rules:**
- The trigger lives on the **WorkflowVersion**, not the Workflow — always use the version ID
- The version must be in `DRAFT` status — you cannot edit a published (`ACTIVE`) version
- Trigger `type` is always UPPER_SNAKE_CASE: `DATABASE_EVENT`, `MANUAL`, `CRON`, `WEBHOOK`, `SEQUENCE`
- For DATABASE_EVENT, `eventName` is `"objectName.action"` — both parts must be **lowercase** (e.g. `"person.created"`, not `"Person.CREATED"`)
- The trigger's `outputSchema` is what makes `{{trigger.*}}` variables available. If you skip calling `computeStepOutputSchema`, downstream steps won't see trigger fields in the variable picker

---

## How to Set a Trigger

Trigger is set via a REST PATCH on the version — send the full trigger object:

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "trigger": { ...trigger object from examples below... }
  }'
```

Then compute the output schema so variables are populated (optional but recommended):

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation ComputeStepOutputSchema($input: ComputeStepOutputSchemaInput!) { computeStepOutputSchema(input: $input) }",
    "variables": {
      "input": {
        "step": { ...same trigger object... },
        "workflowVersionId": "{versionId}"
      }
    }
  }'
```

Paste the returned JSON into `trigger.settings.outputSchema` in your PATCH above, or re-PATCH the version with the output schema included.

---

## Trigger Types

### 1. DATABASE_EVENT — Record Create / Update / Delete

Fires automatically when a record is created, updated, deleted, or destroyed.

```json
{
  "type": "DATABASE_EVENT",
  "name": "Record created",
  "settings": {
    "eventName": "person.created",
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**`settings` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `eventName` | string | Yes | `"objectName.action"` — see table below |
| `outputSchema` | object | No | Populated by `computeStepOutputSchema` |

**`eventName` format — `"objectName.action"`:**

| Action | `eventName` suffix | When it fires |
|---|---|---|
| Created | `.created` | A new record is saved |
| Updated | `.updated` | Any field on the record changes |
| Deleted | `.deleted` | Record is soft-deleted (moved to trash) |
| Destroyed | `.destroyed` | Record is permanently deleted |

**Common object names** (all lowercase): `person`, `company`, `opportunity`, `note`, `task`, `event`, `call`

**Full examples:**
```
"person.created"         → new contact added
"company.updated"        → any company field edited
"opportunity.deleted"    → deal moved to trash
"person.destroyed"       → contact permanently removed
```

**Trigger output variables** (available as `{{trigger.*}}`):
- The full record that triggered the event — all its fields are nested under `properties.after.*`
- e.g. `{{trigger.properties.after.id}}`, `{{trigger.properties.after.name.firstName}}`, `{{trigger.properties.after.companyId}}`
- These paths match what the variable picker shows when you expand "New [object] created" in the UI

*Example prompts: "Trigger when a new person is created", "Fire whenever a company is updated", "Start workflow when an opportunity is deleted"*

---

### 2. MANUAL — Button-triggered from the UI

Shows a button the user can click to launch the workflow — either globally, from a single record, or from multiple selected records.

```json
{
  "type": "MANUAL",
  "name": "Launch manually",
  "settings": {
    "icon": "IconHandMove",
    "isPinned": false,
    "objectType": "person",
    "availability": {
      "type": "SINGLE_RECORD",
      "objectNameSingular": "person"
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**`settings` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `icon` | string | No | Icon name from Twenty icon set (e.g. `"IconHandMove"`) |
| `isPinned` | boolean | No | Pin the button to the record's action bar |
| `objectType` | string | Conditional | Object the trigger is associated with — required when `availability.type` is not `GLOBAL` |
| `availability` | object | No | Controls where the button appears — see below |
| `outputSchema` | object | No | Populated by `computeStepOutputSchema` |

**`availability` options:**

| Type | Shape | Button location |
|---|---|---|
| `GLOBAL` | `{ "type": "GLOBAL" }` | Appears in the global command menu (no specific record) |
| `SINGLE_RECORD` | `{ "type": "SINGLE_RECORD", "objectNameSingular": "person" }` | Appears when viewing or selecting one record of that type |
| `BULK_RECORDS` | `{ "type": "BULK_RECORDS", "objectNameSingular": "person" }` | Appears when multiple records of that type are selected |

**Global trigger (no object):**
```json
{
  "type": "MANUAL",
  "name": "Run from anywhere",
  "settings": {
    "icon": "IconPlayerPlay",
    "isPinned": false,
    "availability": { "type": "GLOBAL" },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**Trigger output variables:**
- For `SINGLE_RECORD`: the selected record's fields are available as `{{trigger.*}}`
- For `GLOBAL` or `BULK_RECORDS`: trigger output is empty unless a `payload` is passed when running via API (`runWorkflowVersion`)

*Example prompts: "Add a manual trigger for person records", "Create a button that appears on company pages", "Make a globally available action button"*

---

### 3. CRON — Scheduled (time-based)

Runs the workflow on a recurring schedule. Supports simple intervals (every N days/hours/minutes) or a custom cron expression.

**Every N days at a specific time:**
```json
{
  "type": "CRON",
  "name": "Run daily",
  "settings": {
    "type": "DAYS",
    "schedule": {
      "day": 1,
      "hour": 9,
      "minute": 0
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**Every N hours at a specific minute:**
```json
{
  "type": "CRON",
  "name": "Run every 2 hours",
  "settings": {
    "type": "HOURS",
    "schedule": {
      "hour": 2,
      "minute": 30
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**Every N minutes:**
```json
{
  "type": "CRON",
  "name": "Run every 15 minutes",
  "settings": {
    "type": "MINUTES",
    "schedule": {
      "minute": 15
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**Custom cron expression:**
```json
{
  "type": "CRON",
  "name": "Run every weekday at 8am",
  "settings": {
    "type": "CUSTOM",
    "pattern": "0 8 * * 1-5",
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**`settings` fields by schedule type:**

| `type` | Fields | Description |
|---|---|---|
| `DAYS` | `schedule.day`, `schedule.hour`, `schedule.minute` | Every `day` days, at `hour`:`minute` |
| `HOURS` | `schedule.hour`, `schedule.minute` | Every `hour` hours, at `minute` past |
| `MINUTES` | `schedule.minute` | Every `minute` minutes |
| `CUSTOM` | `pattern` | Standard 5-field cron expression (`* * * * *`) |

**Cron pattern reminder** (for CUSTOM): `minute hour day-of-month month day-of-week`

**Trigger output variables:** CRON triggers have no record-based output — `{{trigger.*}}` is empty. Use FIND_RECORDS actions in subsequent steps to fetch data.

*Example prompts: "Run every morning at 9am", "Schedule a weekly check every Monday", "Run every 30 minutes"*

---

### 4. WEBHOOK — Inbound HTTP request

Exposes a unique URL that external systems can POST or GET to trigger the workflow.

**POST webhook (accepts a JSON body):**
```json
{
  "type": "WEBHOOK",
  "name": "Inbound webhook",
  "settings": {
    "httpMethod": "POST",
    "authentication": null,
    "expectedBody": {
      "name": "John Doe",
      "email": "john@example.com",
      "source": "website"
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**GET webhook (no body):**
```json
{
  "type": "WEBHOOK",
  "name": "Ping trigger",
  "settings": {
    "httpMethod": "GET",
    "authentication": null,
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**`settings` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `httpMethod` | string | Yes | `"GET"` or `"POST"` |
| `authentication` | string \| null | Yes | `"API_KEY"` or `null` (no auth) |
| `expectedBody` | object | POST only | Example JSON body — defines the output schema for `{{trigger.*}}` variables |
| `outputSchema` | object | No | Computed from `expectedBody` via `computeStepOutputSchema` |

**Webhook URL format:**
```
{serverBaseUrl}/webhooks/workflows/{workspaceId}/{workflowId}
```

The URL is fixed per workflow — it does not change between versions. Retrieve it from the UI or construct it from the workspace and workflow IDs.

**Trigger output variables:**
- For POST: all keys in the incoming JSON body are available as `{{trigger.fieldName}}`
- The `expectedBody` object you define acts as the schema — keys in it become the available variable names
- For GET: no body, so no `{{trigger.*}}` variables are available

*Example prompts: "Create a webhook trigger to receive form submissions", "Set up an inbound HTTP trigger for Zapier", "Make a GET webhook to ping the workflow"*

---

### 5. SEQUENCE — Email sequence event

Fires when a person enters a sequence step (used within Dalil's built-in email sequence feature). This trigger is managed internally by the sequence engine — you generally don't set it manually.

```json
{
  "type": "SEQUENCE",
  "name": "Sequence trigger",
  "settings": {
    "objectType": "person",
    "availability": {
      "type": "SINGLE_RECORD",
      "objectNameSingular": "person"
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

**`settings` fields:**

| Field | Type | Value | Description |
|---|---|---|---|
| `objectType` | string | `"person"` | Always `"person"` — sequences only apply to people |
| `availability.type` | string | `"SINGLE_RECORD"` | Always single record |
| `availability.objectNameSingular` | string | `"person"` | Always `"person"` |

**Trigger output variables:** The triggered person record is available as `{{trigger.*}}` — same shape as a DATABASE_EVENT trigger on `person`.

---

## Full PATCH Example — Set a Trigger on a Version

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "trigger": {
      "type": "DATABASE_EVENT",
      "name": "New person created",
      "settings": {
        "eventName": "person.created",
        "outputSchema": {}
      },
      "nextStepIds": []
    }
  }'
```

---

## Gotchas

1. **Trigger lives on the version, not the workflow** — Always PATCH `/rest/workflowVersions/{versionId}`, not `/rest/workflows/{id}`.
2. **Version must be DRAFT** — Patching the trigger on an ACTIVE or ARCHIVED version will fail. Use `createDraftFromWorkflowVersion` first.
3. **`eventName` must be fully lowercase** — `"person.created"` works; `"Person.Created"` does not.
4. **CRON triggers produce no output** — `{{trigger.*}}` is empty for cron-triggered runs. Fetch data using FIND_RECORDS in your first action step.
5. **Webhook URL is per workflow, not per version** — The URL is tied to the `workflowId`, not the `workflowVersionId`. Activating a new version doesn't change the URL.
6. **POST webhook `expectedBody` defines the variable schema** — If you set `expectedBody: {}`, no `{{trigger.*}}` variables will be available in steps. Add representative fields to unlock them.
7. **`outputSchema` should be computed after setting the trigger** — Call `computeStepOutputSchema` with the trigger object to generate the schema, then include it in `settings.outputSchema`. Skipping this means steps won't show trigger fields in the variable picker.
8. **`nextStepIds` must be an array** — Even if empty, include `"nextStepIds": []`. Omitting it may cause the trigger to not connect to the first step.
9. **SEQUENCE trigger is managed by the system** — Don't attach it manually unless building a sequence workflow. Use DATABASE_EVENT on person records instead for most automation needs.
10. **MANUAL trigger `objectType` must match `availability.objectNameSingular`** — If you set `availability.type: "SINGLE_RECORD"` with `objectNameSingular: "company"`, also set `objectType: "company"` to match.
11. **DATABASE_EVENT trigger fields are nested under `properties.after.*`, not at the root** — All record field variables use the path `{{trigger.properties.after.fieldName}}`. Using `{{trigger.id}}` or `{{trigger.name}}` directly will not resolve. Always use the full `properties.after` prefix, which mirrors what the variable picker shows under "New [object] created/updated".
