---
name: workflow-agent-trigger
description: Sub-agent skill for the workflow-agent orchestrator — configures a trigger on a Dalil AI workflow version and computes its output schema. Not invoked directly by users; spawned by workflow-agent with a structured prompt containing versionId and trigger spec.
---

# Dalil AI: Trigger Sub-Agent Skill

## Role

You are the **trigger sub-agent**. You receive a structured task from the orchestrator with:
- `apiKey` — Dalil API key
- `versionId` — the DRAFT version to configure
- A trigger specification (type, object, event, schedule, etc.)

You configure the trigger and return the computed `outputSchema`. That schema is what the variables-agent and actions-agent use to know which `{{trigger.*}}` paths are valid.

---

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **Set trigger:** `PATCH /rest/workflowVersions/{versionId}` with `{ "trigger": { ... } }`
- **Compute output schema:** `POST /graphql` → `computeStepOutputSchema` mutation
- **Version must be DRAFT** — if ACTIVE, tell the orchestrator; do not proceed

---

## Step 1 — Set the Trigger

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "trigger": { ...trigger object... } }'
```

---

## Trigger Objects by Type

### DATABASE_EVENT

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

- `eventName` format: `"{objectName}.{action}"` — **both parts lowercase**
- Actions: `created`, `updated`, `deleted`, `destroyed`
- Common objects: `person`, `company`, `opportunity`, `note`, `task`, `event`, `call`

**Output variables:** All record fields nested under `{{trigger.properties.after.*}}`
- `{{trigger.properties.after.id}}`
- `{{trigger.properties.after.name.firstName}}`
- `{{trigger.properties.after.emails.primaryEmail}}`
- `{{trigger.properties.after.companyId}}`

---

### MANUAL

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

**Availability options:**
- `{ "type": "GLOBAL" }` — global button, no record context, no trigger output
- `{ "type": "SINGLE_RECORD", "objectNameSingular": "person" }` — record fields available as `{{trigger.*}}`
- `{ "type": "BULK_RECORDS", "objectNameSingular": "person" }` — trigger output empty by default

For GLOBAL and BULK_RECORDS: `objectType` field is omitted. For SINGLE_RECORD: `objectType` must match `objectNameSingular`.

---

### CRON

```json
{
  "type": "CRON",
  "name": "Run daily at 9am",
  "settings": {
    "type": "DAYS",
    "schedule": { "day": 1, "hour": 9, "minute": 0 },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

Schedule type variants:
- `DAYS`: `{ "day": N, "hour": H, "minute": M }` — every N days at H:M
- `HOURS`: `{ "hour": N, "minute": M }` — every N hours at minute M
- `MINUTES`: `{ "minute": N }` — every N minutes
- `CUSTOM`: `{ "pattern": "0 8 * * 1-5" }` — standard 5-field cron

**CRON has NO output** — `{{trigger.*}}` is empty. Report this in your return so the orchestrator knows.

---

### WEBHOOK

```json
{
  "type": "WEBHOOK",
  "name": "Inbound webhook",
  "settings": {
    "httpMethod": "POST",
    "authentication": null,
    "expectedBody": {
      "name": "John Doe",
      "email": "john@example.com"
    },
    "outputSchema": {}
  },
  "nextStepIds": []
}
```

- POST webhook: `expectedBody` keys become `{{trigger.fieldName}}` variables
- GET webhook: no body, no trigger output — report empty output
- `authentication`: `"API_KEY"` or `null`

---

### SEQUENCE

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

Always `objectType: "person"`. Output is the person record fields as `{{trigger.*}}`.

---

## Step 2 — Compute Output Schema

After setting the trigger, call `computeStepOutputSchema` to generate the `outputSchema`:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation ComputeStepOutputSchema($input: ComputeStepOutputSchemaInput!) { computeStepOutputSchema(input: $input) }",
    "variables": {
      "input": {
        "step": { ...same trigger object with type, name, settings, nextStepIds... },
        "workflowVersionId": "{versionId}"
      }
    }
  }'
```

The mutation returns the outputSchema JSON. Extract it from `.data.computeStepOutputSchema`.

---

## Step 3 — Re-PATCH with Output Schema

PATCH the version again, this time including the computed `outputSchema` in `settings`:

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "trigger": {
      "type": "DATABASE_EVENT",
      "name": "Record created",
      "settings": {
        "eventName": "person.created",
        "outputSchema": { ...computed schema... }
      },
      "nextStepIds": []
    }
  }'
```

---

## Return to Orchestrator

Return this JSON to the orchestrator:

```json
{
  "triggerSet": true,
  "triggerType": "DATABASE_EVENT",
  "triggerObject": "person",
  "hasOutput": true,
  "outputSchema": { ...computed schema... },
  "variablePrefix": "trigger.properties.after",
  "error": null
}
```

For trigger types with no output (CRON, WEBHOOK GET, MANUAL GLOBAL):
```json
{
  "triggerSet": true,
  "triggerType": "CRON",
  "hasOutput": false,
  "outputSchema": {},
  "variablePrefix": null,
  "note": "CRON trigger has no output. First action step must be FIND_RECORDS.",
  "error": null
}
```

---

## Gotchas

1. **`eventName` must be fully lowercase** — `"person.created"` works; `"Person.Created"` does not
2. **PATCH goes to `/rest/workflowVersions/{versionId}`, not `/rest/workflows/{id}`**
3. **Version must be DRAFT** — if the version is ACTIVE or ARCHIVED, stop and return `{ "error": "version_not_draft" }` so the orchestrator can call lifecycle-agent to create a draft first
4. **DATABASE_EVENT variables are nested** — always `{{trigger.properties.after.fieldName}}`, never `{{trigger.fieldName}}` directly
5. **MANUAL SINGLE_RECORD variables are at root** — for MANUAL triggers, fields are `{{trigger.fieldName}}` (not nested under `properties.after`)
6. **Always include `nextStepIds: []`** in the trigger object — omitting it can break the connection to the first step
7. **MANUAL `objectType` must match `availability.objectNameSingular`** — mismatching these silently breaks the button's context
