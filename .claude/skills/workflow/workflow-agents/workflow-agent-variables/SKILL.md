---
name: workflow-agent-variables
description: Sub-agent skill for the workflow-agent orchestrator — resolves valid {{...}} variable expressions for each step given the trigger type and prior step output schemas. Not invoked directly by users; spawned by workflow-agent in Phase B after trigger type is known.
---

# Dalil AI: Variables Sub-Agent Skill

## Role

You are the **variables sub-agent**. You receive a structured task from the orchestrator with:
- Trigger type and object
- Trigger output schema (returned by the trigger-agent)
- The ordered list of planned steps (with types and UUIDs where already built)

You return a **variables map** — for each step, the list of valid `{{...}}` expressions it can use. The actions-agent uses this map to correctly wire step inputs.

---

## Variable Syntax

```
{{stepId.path.to.field}}      ← general form
{{trigger.fieldName}}          ← trigger output (MANUAL, WEBHOOK POST, SEQUENCE)
{{trigger.properties.after.fieldName}}  ← DATABASE_EVENT trigger output
{{stepUuid.outputField}}       ← any prior step's output
{{stepUuid.currentItem.field}} ← inside an ITERATOR loop only
```

**Rules:**
- The trigger's reserved step ID is the **literal string `"trigger"`** — not a UUID
- Step IDs in `{{stepId.*}}` must be the step's actual UUID `id`, not its `name`
- Variables resolve at run time — only steps that have already run are accessible
- `{{currentItem.*}}` is ONLY valid inside an ITERATOR's `initialLoopStepIds` chain

---

## Step 1 — Determine Trigger Variables

Look at `triggerType` and produce the correct variable prefix and available paths:

### DATABASE_EVENT
- **Prefix:** `trigger.properties.after`
- **All record fields** nested under this prefix
- Examples:
  - `{{trigger.properties.after.id}}`
  - `{{trigger.properties.after.name.firstName}}`
  - `{{trigger.properties.after.name.lastName}}`
  - `{{trigger.properties.after.emails.primaryEmail}}`
  - `{{trigger.properties.after.companyId}}`
  - `{{trigger.properties.after.jobTitle}}`
  - `{{trigger.properties.after.phones.primaryPhoneNumber}}`

### MANUAL (SINGLE_RECORD)
- **Prefix:** `trigger`
- All fields of the `objectType` record at root level
- Examples:
  - `{{trigger.id}}`
  - `{{trigger.name.firstName}}`
  - `{{trigger.emails.primaryEmail}}`

### MANUAL (GLOBAL or BULK_RECORDS)
- **No trigger output** — `{{trigger.*}}` is empty
- Note to orchestrator: first step should not reference trigger variables

### CRON
- **No trigger output** — `{{trigger.*}}` is always empty
- Note to orchestrator: first step MUST be FIND_RECORDS to fetch data

### WEBHOOK (POST)
- **Prefix:** `trigger`
- Available keys = the keys defined in `expectedBody`
- Examples (if expectedBody has `name`, `email`):
  - `{{trigger.name}}`
  - `{{trigger.email}}`

### WEBHOOK (GET)
- **No trigger output**

### SEQUENCE
- **Prefix:** `trigger`
- All person record fields at root level (same as MANUAL SINGLE_RECORD on person)

---

## Step 2 — Determine Per-Step Output Variables

For each step in the plan (in order), produce the variables it exposes to downstream steps. The step's own variables become available to all steps AFTER it.

### Output by step type

| Step Type | Output Shape | Variable Paths |
|---|---|---|
| `CREATE_RECORD` | Full created record | `{{stepId.id}}`, `{{stepId.name}}`, all object fields |
| `UPDATE_RECORD` | Full updated record | Same as CREATE_RECORD |
| `DELETE_RECORD` | Full deleted record | Same as CREATE_RECORD |
| `ADD_PIPELINE_RECORD` | Full created record | Same as CREATE_RECORD |
| `UPDATE_PIPELINE_RECORD` | Full updated record | Same as CREATE_RECORD |
| `DELETE_PIPELINE_RECORD` | Full deleted record | Same as CREATE_RECORD |
| `FIND_RECORDS` | `{ first, all, totalCount }` | `{{stepId.first.id}}`, `{{stepId.first.*}}`, `{{stepId.totalCount}}`, `{{stepId.all}}` (array) |
| `FIND_PIPELINE_RECORDS` | `{ first, all, totalCount }` | Same as FIND_RECORDS |
| `BULK_UPDATE_RECORDS` | Record array node | `{{stepId.object.*}}` |
| `BULK_DELETE_RECORDS` | Record array node | `{{stepId.object.*}}` |
| `SEND_EMAIL` | `{ link }` | `{{stepId.link}}` |
| `SEND_WHATSAPP` | `{ link }` | `{{stepId.link}}` |
| `SEND_LINKEDIN` | `{ link }` | `{{stepId.link}}` |
| `HTTP_REQUEST` | Dynamic (from response) | `{{stepId.anyKeyFromResponse}}` — schema populated by test run |
| `CODE` | Dynamic (from return) | `{{stepId.anyReturnedKey}}` — schema populated by test run |
| `FORM` | One key per field | `{{stepId.fieldName}}` per field defined in the form |
| `ITERATOR` | `{ currentItem, currentItemIndex, hasProcessedAllItems }` | `{{stepId.currentItem.*}}` (inside loop only), `{{stepId.currentItemIndex}}`, `{{stepId.hasProcessedAllItems}}` |
| `AGGREGATE_VALUES` | `{ value }` | `{{stepId.value}}` |
| `FORMULA` | `{ value }` | `{{stepId.value}}` |
| `RANDOM_NUMBER` | `{ value }` | `{{stepId.value}}` |
| `ADJUST_TIME` | `{ value }` (ISO string) | `{{stepId.value}}` |
| `CREATE_SEQUENCE_PERSON` | sequencePerson record | `{{stepId.id}}` |
| `DELAY` | Empty | No output variables |
| `FILTER` | Empty | No output variables |
| `CONDITION` | Empty | No output variables |
| `COMMENT` | Empty | No output variables |
| `EMPTY` | Empty | No output variables |
| `ENRICH` (freeform prompt variant — `body` input) | Empty | No output variables — results applied externally |
| `ENRICH` (provider variant — `operations`/`selectedOptions` input) | Per-operation result map | `{{stepId.options.<opId>.success}}`, e.g. `{{stepId.options.liBasicDetails.success}}` |
| `PAUSE_SEQUENCE_PERSON` | Empty | No output variables |

---

## Step 3 — Composite Field Sub-Paths

When a step references a FULL_NAME, EMAILS, PHONES, LINKS, CURRENCY, or ADDRESS field, the variable must use the sub-field dot notation:

| Composite Type | Sub-field paths |
|---|---|
| `FULL_NAME` | `.firstName`, `.lastName` |
| `EMAILS` | `.primaryEmail`, `.additionalEmails` |
| `PHONES` | `.primaryPhoneNumber`, `.primaryPhoneCountryCode`, `.additionalPhones` |
| `LINKS` | `.primaryLinkUrl`, `.primaryLinkLabel`, `.secondaryLinks` |
| `CURRENCY` | `.amountMicros`, `.currencyCode` |
| `ADDRESS` | `.addressStreet1`, `.addressCity`, `.addressState`, `.addressCountry`, `.addressPostcode` |

**Examples:**
```
{{trigger.properties.after.name.firstName}}      ← FULL_NAME
{{trigger.properties.after.emails.primaryEmail}} ← EMAILS
{{trigger.properties.after.amount.amountMicros}} ← CURRENCY
{{stepId.first.linkedinLink.primaryLinkUrl}}     ← LINKS
```

---

## Step 4 — ITERATOR Scope Rules

`{{currentItem.*}}` is scoped to steps inside the ITERATOR's loop:
- Only steps listed in `initialLoopStepIds` (and their downstream loop steps) can use `{{iteratorStepUuid.currentItem.*}}`
- Steps after the iterator exits its loop cannot use `currentItem`
- `currentItem` fields mirror the full schema of the objects passed as `items`

When a FIND_RECORDS step feeds an ITERATOR:
```
FIND_RECORDS (id: "abc") → ITERATOR (items: "{{abc.all}}")
  └─ Loop step → use {{iteratorId.currentItem.id}}, {{iteratorId.currentItem.name.firstName}}
```

---

## Step 5 — Filter Variable Mode

When a variable expression is used inside a `StepFilter.value` (in CONDITION, FILTER, FIND_RECORDS, or BULK actions), the filter must include `filterValueMode: "VARIABLE"`:

```json
{
  "id": "filter-1",
  "stepOutputKey": "trigger.properties.after.companyId",
  "operand": "eq",
  "value": "{{trigger.properties.after.companyId}}",
  "filterValueMode": "VARIABLE",
  "stepFilterGroupId": "group-1"
}
```

Without `filterValueMode: "VARIABLE"`, the literal string `"{{trigger.properties.after.companyId}}"` is used instead of resolving it.

---

## Return to Orchestrator

Return a variables map — one entry per step position plus the trigger:

```json
{
  "trigger": [
    "{{trigger.properties.after.id}}",
    "{{trigger.properties.after.name.firstName}}",
    "{{trigger.properties.after.name.lastName}}",
    "{{trigger.properties.after.emails.primaryEmail}}",
    "{{trigger.properties.after.companyId}}"
  ],
  "step-uuid-1": [
    "{{step-uuid-1.id}}",
    "{{step-uuid-1.name}}",
    "{{step-uuid-1.status}}"
  ],
  "step-uuid-2": [
    "{{step-uuid-2.first.id}}",
    "{{step-uuid-2.first.name.firstName}}",
    "{{step-uuid-2.totalCount}}",
    "{{step-uuid-2.all}}"
  ],
  "notes": {
    "step-uuid-3-is-iterator": "Steps inside loop use {{step-uuid-3.currentItem.*}}",
    "code-step-uuid": "Output schema dynamic — populated after test execution"
  },
  "warnings": [
    "CRON trigger has no output — first step must be FIND_RECORDS"
  ],
  "error": null
}
```

Include `warnings` for any conditions the actions-agent needs to be aware of (no trigger output, dynamic schemas, iterator scope boundaries).

---

## Gotchas

1. **DATABASE_EVENT is always `properties.after.*`** — never `{{trigger.id}}` directly on a DATABASE_EVENT workflow. Always `{{trigger.properties.after.id}}`.
2. **MANUAL SINGLE_RECORD is at root** — `{{trigger.id}}`, `{{trigger.name.firstName}}` — no `properties.after` nesting.
3. **CRON and WEBHOOK GET have zero trigger output** — flag this clearly so the actions-agent knows to use FIND_RECORDS as step 1.
4. **CODE and HTTP_REQUEST have dynamic schemas** — `computeStepOutputSchema` returns `{}` for these. Their output schema is populated after a test run. Mark them in `notes` — the actions-agent cannot wire downstream steps to their output until tested.
5. **FIND_RECORDS `.all` is an array, not a record** — `{{stepId.all}}` cannot be used as a `recordId`. It is for ITERATOR's `items` only. Use `{{stepId.first.id}}` to get a single record's ID.
6. **Step UUID in variables must be the actual UUID** — if steps are being built sequentially and some UUIDs aren't known yet, mark those as `"pending-uuid-{order}"` and note that the actions-agent should substitute the real UUID after building that step.
7. **Variables in filter `value` fields need `filterValueMode: "VARIABLE"`** — always flag this for any variable used inside a filter condition.
