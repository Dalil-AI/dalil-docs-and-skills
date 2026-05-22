---
name: workflow-variables
description: Use the {{...}} variable system in Dalil AI workflow steps — reference trigger outputs, prior step outputs, and iterator current items. Covers syntax, output schema shapes per action type, how to compute output schemas, and nested path access for composite fields.
---

# Dalil AI: Workflow Variables Skill

## Quick Reference

- **Variable syntax:** `{{stepId.path.to.field}}`
- **Trigger variable:** `{{trigger.fieldName}}` — the literal string `trigger` is the reserved step ID for the trigger node
- **Step variable:** `{{stepUuid.outputField}}` — use the step's actual `id` UUID, not its `name`
- **Iterator current item:** `{{stepUuid.currentItem.fieldName}}` — only valid inside the iterator's loop steps
- **Compute output schema:** `POST https://app.usedalil.ai/graphql` → `computeStepOutputSchema` mutation
- **Where variables appear:** inside a step's `settings.input` fields — any string value can contain `{{...}}` expressions

**Critical rules:**
- The trigger's reserved step ID is the literal string `"trigger"` — not a UUID
- Step IDs in `{{stepId.*}}` must be the UUID `id` field from the step object, NOT the step's `name`
- Variables are resolved at run time — they will be empty strings if the referenced step hasn't run yet
- `{{currentItem.*}}` is only valid within an ITERATOR's `initialLoopStepIds` chain — using it outside an iterator returns nothing
- CRON and WEBHOOK (GET) triggers have no output — `{{trigger.*}}` is empty for those trigger types
- CODE and HTTP_REQUEST steps have dynamic output schemas — they must be populated by running a test execution, not via `computeStepOutputSchema`

---

## Variable Syntax

Variables are dot-path expressions wrapped in double curly braces:

```
{{stepId}}                    ← full step output object
{{stepId.field}}              ← top-level field
{{stepId.field.subField}}     ← nested field (e.g. composite types)
{{trigger.name.firstName}}    ← trigger output, composite FULL_NAME field
{{stepUuid.currentItem.id}}   ← iterator current item field
```

The path segments map directly to the keys in the step's `outputSchema`.

**Static string with embedded variables:**
```json
"body": "Hello {{trigger.name.firstName}}, your deal {{stepUuid.name}} is closing soon."
```

**Entire field as variable:**
```json
"recordId": "{{trigger.id}}"
```

---

## Output Schema per Step Type

### Trigger Output Schemas

| Trigger Type | Output | `{{trigger.*}}` fields |
|---|---|---|
| `DATABASE_EVENT` | Full record that fired the event | All fields of the object (e.g. `{{trigger.id}}`, `{{trigger.name.firstName}}`) |
| `MANUAL` (SINGLE_RECORD) | The selected record | All fields of `objectType` |
| `MANUAL` (GLOBAL) | Empty unless payload passed via API | Depends on `payload` sent to `runWorkflowVersion` |
| `MANUAL` (BULK_RECORDS) | Empty by default | Requires iterator to loop over records |
| `CRON` | **Empty** | No `{{trigger.*}}` variables available |
| `WEBHOOK` (POST) | Keys from `expectedBody` | `{{trigger.fieldName}}` per key defined in `expectedBody` |
| `WEBHOOK` (GET) | **Empty** | No body → no variables |
| `SEQUENCE` | The triggered person record | All `person` fields |

### Action Output Schemas

| Action Type | Output Shape | Available Variables |
|---|---|---|
| `CREATE_RECORD` | Full created record | `{{stepId.id}}`, `{{stepId.name}}`, all object fields |
| `UPDATE_RECORD` | Full updated record | Same as CREATE_RECORD |
| `DELETE_RECORD` | Full deleted record | Same as CREATE_RECORD |
| `ADD_PIPELINE_RECORD` | Full created record | Same as CREATE_RECORD |
| `UPDATE_PIPELINE_RECORD` | Full updated record | Same as CREATE_RECORD |
| `DELETE_PIPELINE_RECORD` | Full deleted record | Same as CREATE_RECORD |
| `FIND_RECORDS` | `{ first, all, totalCount }` | `{{stepId.first.id}}`, `{{stepId.totalCount}}` |
| `FIND_PIPELINE_RECORDS` | `{ first, all, totalCount }` | Same as FIND_RECORDS |
| `BULK_UPDATE_RECORDS` | Full record array node | `{{stepId.object.*}}` |
| `BULK_DELETE_RECORDS` | Full record array node | `{{stepId.object.*}}` |
| `BULK_UPDATE_PIPELINE_RECORDS` | Full record array node | Same as BULK_UPDATE_RECORDS |
| `BULK_DELETE_PIPELINE_RECORDS` | Full record array node | Same as BULK_DELETE_RECORDS |
| `SEND_EMAIL` | Link output `{ link }` | `{{stepId.link}}` |
| `SEND_WHATSAPP` | Link output `{ link }` | `{{stepId.link}}` |
| `SEND_LINKEDIN` | Link output `{ link }` | `{{stepId.link}}` |
| `HTTP_REQUEST` | Dynamic — from example response | `{{stepId.anyKeyFromResponse}}` |
| `CODE` | Dynamic — from test execution | `{{stepId.anyReturnedKey}}` |
| `FORM` | One key per form field | `{{stepId.fieldName}}` per field |
| `ITERATOR` | `{ currentItem, currentItemIndex, hasProcessedAllItems }` | `{{stepId.currentItem.*}}`, `{{stepId.currentItemIndex}}`, `{{stepId.hasProcessedAllItems}}` |
| `AGGREGATE_VALUES` | `{ value }` (number) | `{{stepId.value}}` |
| `FORMULA` | `{ value }` (number) | `{{stepId.value}}` |
| `RANDOM_NUMBER` | `{ value }` (number) | `{{stepId.value}}` |
| `ADJUST_TIME` | `{ value }` (ISO string) | `{{stepId.value}}` |
| `CREATE_SEQUENCE_PERSON` | sequencePerson record | `{{stepId.id}}`, sequence person fields |
| `DELETE_SEQUENCE_PERSON` | sequencePerson record | Same as CREATE_SEQUENCE_PERSON |
| `DELAY` | **Empty** | No output variables |
| `FILTER` | **Empty** | No output variables |
| `CONDITION` | **Empty** | No output variables |
| `COMMENT` | **Empty** | No output variables |
| `EMPTY` | **Empty** | No output variables |
| `ENRICH` | **Empty** | No output variables |
| `PAUSE_SEQUENCE_PERSON` | **Empty** | No output variables |

---

## FIND_RECORDS Output Detail

FIND_RECORDS returns three top-level keys:

```
{{stepId.first.*}}       ← the first matching record (full record fields)
{{stepId.all}}           ← array of all matching records (use with ITERATOR)
{{stepId.totalCount}}    ← integer count of matching records
```

Example:
```json
"recordId": "{{findPeopleStepUuid.first.id}}"
"body": "Found {{findPeopleStepUuid.totalCount}} people matching your search."
```

To loop over all results: pass `{{findPeopleStepUuid.all}}` as the ITERATOR's `items` field.

---

## ITERATOR Output Detail

Inside the iterator's loop steps, use `currentItem` to access the current iteration's record:

```
{{iteratorStepUuid.currentItem.id}}
{{iteratorStepUuid.currentItem.name.firstName}}
{{iteratorStepUuid.currentItem.emails.primaryEmail}}
{{iteratorStepUuid.currentItemIndex}}         ← 0-based index
{{iteratorStepUuid.hasProcessedAllItems}}     ← true on final iteration
```

The `currentItem` fields mirror the full record schema of whatever object was passed as `items`.

---

## Composite Field Paths

Fields of composite types (FULL_NAME, EMAILS, PHONES, LINKS, CURRENCY, ADDRESS) expose sub-fields via dot notation:

| Field Type | Sub-field paths |
|---|---|
| `FULL_NAME` | `.firstName`, `.lastName` |
| `EMAILS` | `.primaryEmail`, `.additionalEmails` |
| `PHONES` | `.primaryPhoneNumber`, `.primaryPhoneCountryCode`, `.additionalPhones` |
| `LINKS` | `.primaryLinkUrl`, `.primaryLinkLabel`, `.secondaryLinks` |
| `CURRENCY` | `.amountMicros`, `.currencyCode` |
| `ADDRESS` | `.addressStreet1`, `.addressCity`, `.addressState`, `.addressCountry`, `.addressPostcode` |

Examples:
```
{{trigger.name.firstName}}
{{trigger.emails.primaryEmail}}
{{trigger.amount.amountMicros}}
{{trigger.city.addressCity}}
{{trigger.linkedinLink.primaryLinkUrl}}
```

---

## Compute Output Schema

Call `computeStepOutputSchema` after creating or updating a step to populate `settings.outputSchema`. This is what makes `{{stepId.*}}` variables available to downstream steps.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation ComputeStepOutputSchema($input: ComputeStepOutputSchemaInput!) { computeStepOutputSchema(input: $input) }",
    "variables": {
      "input": {
        "step": {
          "id": "step-uuid-or-trigger",
          "name": "My Step",
          "type": "CREATE_RECORD",
          "settings": {
            "input": { "objectName": "person", "objectRecord": {} },
            "outputSchema": {},
            "errorHandlingOptions": {
              "retryOnFailure": { "value": false },
              "continueOnFailure": { "value": false }
            }
          },
          "valid": true
        },
        "workflowVersionId": "{versionId}"
      }
    }
  }'
```

The mutation returns the `outputSchema` JSON directly. Take the returned value and PATCH it back into the step:

```bash
# 1. Compute the schema
OUTPUT_SCHEMA=$(curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "query": "mutation ComputeStepOutputSchema($input: ComputeStepOutputSchemaInput!) { computeStepOutputSchema(input: $input) }", "variables": { "input": { "step": {...stepObject...}, "workflowVersionId": "{versionId}" } } }' \
  | jq '.data.computeStepOutputSchema')

# 2. Update the step with the computed schema via updateWorkflowVersionStep
```

For triggers specifically, the outputSchema is patched directly onto the version:

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
        "outputSchema": { ...computed schema here... }
      },
      "nextStepIds": ["first-step-uuid"]
    }
  }'
```

---

## Full Variable Usage Example

A workflow that fires when a person is created, finds their company, then sends an email:

**Trigger** (id: `"trigger"`):
```
{{trigger.id}}                    ← person UUID
{{trigger.name.firstName}}        ← person first name
{{trigger.companyId}}             ← company relation ID
```

**Step 1: FIND_RECORDS** (id: `"abc-123-..."`):
```json
{
  "objectName": "company",
  "filter": { "id": { "eq": "{{trigger.companyId}}" } }
}
```
Output:
```
{{abc-123-....first.name}}        ← company name
{{abc-123-....first.domainName.primaryLinkUrl}}
```

**Step 2: SEND_EMAIL** (references both trigger and step 1):
```json
{
  "to": "{{trigger.emails.primaryEmail}}",
  "subject": "Welcome to {{abc-123-....first.name}}!",
  "body": "Hi {{trigger.name.firstName}}, welcome aboard."
}
```

---

## Gotchas

1. **Trigger step ID is the literal string `"trigger"`** — Not a UUID. Use `{{trigger.fieldName}}`, never `{{<some-uuid>.fieldName}}` for the trigger.
2. **Step ID in variables is the step UUID, not the step name** — If your step is named `"Find Company"` with id `"abc-123"`, use `{{abc-123.first.name}}`, not `{{Find Company.first.name}}`.
3. **CRON triggers have no output** — `{{trigger.*}}` is empty for cron-triggered runs. Put a FIND_RECORDS step first to fetch the data you need.
4. **WEBHOOK GET has no output** — Only POST webhooks with a defined `expectedBody` produce `{{trigger.*}}` variables. GET webhooks return nothing.
5. **CODE and HTTP_REQUEST schemas are dynamic** — `computeStepOutputSchema` returns `{}` for these types. Their output schema is populated when you run a test execution (CODE) or define an example response body (HTTP_REQUEST).
6. **`currentItem` only works inside the iterator loop** — Steps outside the iterator's `initialLoopStepIds` chain cannot access `{{iteratorId.currentItem.*}}`.
7. **Skip `computeStepOutputSchema` → downstream steps see no variables** — Without a populated `outputSchema`, the variable picker shows nothing and `{{stepId.*}}` expressions resolve to empty strings at runtime.
8. **FIND_RECORDS `all` requires ITERATOR to be useful** — `{{stepId.all}}` is an array. You can't reference individual items directly — pass it to an ITERATOR's `items` field and use `{{iteratorId.currentItem.*}}` inside the loop.
9. **Composite fields need their sub-path** — `{{trigger.name}}` returns the raw composite object, not a string. Use `{{trigger.name.firstName}}` to get a usable string value.
10. **Variables in filter values need `filterValueMode: "VARIABLE"`** — When using a `{{...}}` expression in a `StepFilter.value`, set `filterValueMode` to `"VARIABLE"`. Without this, the literal string `{{trigger.id}}` is used instead of resolving it.
