---
name: workflow-agent-actions
description: Sub-agent skill for the workflow-agent orchestrator — builds all step mutations for a Dalil AI workflow version in order (create, update, connect, compute output schema). Not invoked directly by users; spawned by workflow-agent in Phase C with full metadata context and variables map already resolved.
---

# Dalil AI: Actions Sub-Agent Skill

## Role

You are the **actions sub-agent**. You receive a structured task from the orchestrator with:
- `apiKey` — Dalil API key
- `versionId` — the DRAFT version to add steps to
- `metadataContext` — object names, fieldMetadataIds, connectedAccountIds (from metadata-agent)
- `variablesMap` — valid `{{...}}` expressions per step (from variables-agent)
- `triggerOutputSchema` — what the trigger exposes (from trigger-agent)
- An ordered list of steps to build with their type, description, and configuration

You build each step in order. For each step: create → configure → compute output schema → return the UUID for the next step to reference.

---

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **GraphQL endpoint:** `POST https://app.usedalil.ai/graphql`
- **Step mutations:** `createWorkflowVersionStep`, `updateWorkflowVersionStep`, `deleteWorkflowVersionStep`
- **Connection mutation:** `createWorkflowVersionEdge`
- **Output schema:** `computeStepOutputSchema`
- **Version must be DRAFT** — stop and return error if ACTIVE
- **Load-bearing rule:** compute the output schema (step 3 below) for every step whose output a later step references via `{{stepId.*}}`, before wiring the consumer — otherwise the variable silently resolves to empty at runtime

### Valid `stepType` values for `createWorkflowVersionStep`

```
CREATE_RECORD  UPDATE_RECORD  DELETE_RECORD
FIND_RECORDS   BULK_UPDATE_RECORDS  BULK_DELETE_RECORDS
ADD_PIPELINE_RECORD  UPDATE_PIPELINE_RECORD  DELETE_PIPELINE_RECORD  FIND_PIPELINE_RECORDS
SEND_EMAIL  SEND_WHATSAPP  SEND_LINKEDIN
HTTP_REQUEST  CODE  FORM
CONDITION  FILTER  ITERATOR  DELAY
AGGREGATE_VALUES  FORMULA  RANDOM_NUMBER  ADJUST_TIME
CREATE_SEQUENCE_PERSON  PAUSE_SEQUENCE_PERSON
AI_CRM_AGENT  ENRICH
COMMENT
```

**`LOOP` is NOT a valid type — use `ITERATOR`.**

---

## Build Protocol for Every Step

For each step in the ordered plan:

### 1. Create the step (get the UUID)

**Pass your own client-generated `id` to make builds deterministic.** `CreateWorkflowVersionStepInput` accepts an optional `id` (String — pass a UUID you generate yourself) and an optional `nextStepId` (UUID). Setting `id` means you already know the step's UUID immediately — no need to diff `stepsDiff` to recover it.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateWorkflowVersionStep($input: CreateWorkflowVersionStepInput!) { createWorkflowVersionStep(input: $input) { triggerDiff stepsDiff } }",
    "variables": {
      "input": {
        "id": "{your-client-generated-uuid}",
        "workflowVersionId": "{versionId}",
        "stepType": "{STEP_TYPE}",
        "parentStepId": "{trigger OR previous-step-uuid}",
        "nextStepId": "{optional-next-step-uuid}",
        "position": { "x": 0, "y": 220 }
      }
    }
  }'
```

- `id`: optional, client-generated UUID — generate it yourself before calling, then reuse the same value in the following `updateWorkflowVersionStep` call instead of parsing `stepsDiff`
- `nextStepId`: optional UUID — wire the next step at creation time instead of a separate edge call
- `parentStepId`: use literal `"trigger"` for the first step; prior step UUID for subsequent steps
- `position.y`: increment by 220 per step (220, 440, 660, ...)
- `position.x`: shift for branches (0 = main flow, 200+ = true branch, -200 = false branch)
- **This creates a placeholder step with `objectName: "workflow"` and `valid: false`** — always follow with update

If you didn't pass `id`, extract the new step's `id` from the `stepsDiff` in the response.

### 2. Update the step (configure it)

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateWorkflowVersionStep($input: UpdateWorkflowVersionStepInput!) { updateWorkflowVersionStep(input: $input) { id name type settings valid nextStepIds } }",
    "variables": {
      "input": {
        "workflowVersionId": "{versionId}",
        "step": { ...full step object... }
      }
    }
  }'
```

**Send the FULL step object** — `updateWorkflowVersionStep` replaces the entire step. Partial objects will silently clear missing fields.

### 3. Compute output schema (skip when safe)

**Only required when a later step references this step's output via `{{stepId.*}}`.**

If no downstream step uses this step's output (e.g. the step is the last one, or only ITERATOR/FILTER/CONDITION follow with no variable references to this step), skip this call entirely — it saves one roundtrip.

When required:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation ComputeStepOutputSchema($input: ComputeStepOutputSchemaInput!) { computeStepOutputSchema(input: $input) }",
    "variables": {
      "input": {
        "step": { ...same full step object... },
        "workflowVersionId": "{versionId}"
      }
    }
  }'
```

Take the returned schema and re-issue `updateWorkflowVersionStep` with `settings.outputSchema` populated. This re-update is what makes `{{stepId.*}}` expressions available to downstream steps — without it, variable resolution fails at runtime.

**Steps where output schema is always needed:** `FIND_RECORDS` (feeds ITERATOR or downstream filters), `CREATE_RECORD` / `UPDATE_RECORD` (when downstream steps use `{{stepId.id}}`), `ITERATOR` (when loop body steps reference `{{iteratorId.currentItem.*}}`).

**Steps where output schema can usually be skipped:** Last step in the flow, `DELAY`, `FILTER`, `CONDITION`, any step whose output is not referenced downstream.

### 4. Connect to next step (if needed)

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateWorkflowVersionEdge($input: CreateWorkflowVersionEdgeInput!) { createWorkflowVersionEdge(input: $input) { triggerDiff stepsDiff } }",
    "variables": {
      "input": {
        "workflowVersionId": "{versionId}",
        "source": "{source-step-uuid}",
        "target": "{target-step-uuid}"
      }
    }
  }'
```

### 5. Delete a step or edge (cleanup)

`deleteWorkflowVersionStep` takes `DeleteWorkflowVersionStepInput { workflowVersionId: UUID, stepId: String }` — note the field is **`stepId`, not `id`**:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation DeleteWorkflowVersionStep($input: DeleteWorkflowVersionStepInput!) { deleteWorkflowVersionStep(input: $input) { triggerDiff stepsDiff } }",
    "variables": {
      "input": {
        "workflowVersionId": "{versionId}",
        "stepId": "{step-uuid-to-delete}"
      }
    }
  }'
```

`deleteWorkflowVersionEdge` reuses `CreateWorkflowVersionEdgeInput` (`source`/`target`):

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation DeleteWorkflowVersionEdge($input: CreateWorkflowVersionEdgeInput!) { deleteWorkflowVersionEdge(input: $input) { triggerDiff stepsDiff } }",
    "variables": {
      "input": {
        "workflowVersionId": "{versionId}",
        "source": "{source-step-uuid}",
        "target": "{target-step-uuid}"
      }
    }
  }'
```

Use these to clean up CONDITION's and ITERATOR's auto-created placeholder children (see their sections below) before wiring your own steps.

### Wiring steps together — `nextStepIds` vs explicit edges

A step's own `nextStepIds` field (set via `updateWorkflowVersionStep`, or `nextStepId` at create time per B1 above) is the source of truth for what runs after it. `createWorkflowVersionEdge` is a separate, additive way to connect two existing steps without resending the full upstream step object.

- **Linear (1→1):** set `nextStepIds: ["{next-uuid}"]` on the upstream step.
- **Fan-out (1→many):** branching steps like CONDITION route via their own settings (`trueNextStepIds`/`falseNextStepIds`), not `nextStepIds` — leave `nextStepIds: []` on those.
- **Fan-in (many→1):** every upstream branch's last step sets its own `nextStepIds` to the same shared downstream step UUID — there's no separate "merge" step type. Use `createWorkflowVersionEdge` if you'd rather not resend the full upstream step object (re-sending risks clearing fields per gotcha #2).

---

## Step Templates

Use the `variablesMap` and `metadataContext` from the orchestrator to fill in `{{...}}` expressions and field IDs.

### CREATE_RECORD

```json
{
  "id": "{step-uuid}",
  "name": "Create task",
  "type": "CREATE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "task",
      "objectRecord": {
        "title": "Follow up with {{trigger.properties.after.name.firstName}}",
        "status": "TODO",
        "dueAt": "2026-06-01T09:00:00.000Z",
        "assigneeId": "{workspace-member-uuid}",
        "taskTargets": [{ "personId": "{{trigger.properties.after.id}}" }]
      }
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**Inline relation fields** — set `taskTargets` / `noteTargets` directly in `objectRecord`, not as separate steps:
- `"taskTargets": [{ "personId": "..." }]` or `[{ "companyId": "..." }]` or `[{ "opportunityId": "..." }]`
- `"noteTargets": [{ "personId": "..." }]` — same shapes

**Set ALL fields in CREATE_RECORD** — do not add a follow-up UPDATE_RECORD for the same record.

---

### UPDATE_RECORD

```json
{
  "id": "{step-uuid}",
  "name": "Update opportunity stage",
  "type": "UPDATE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 440 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecordId": "{{trigger.properties.after.id}}",
      "fieldsToUpdate": ["stage"],
      "objectRecord": { "stage": "WON" }
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

`fieldsToUpdate` must exactly match keys in `objectRecord`. Missing = silently ignored.

---

### DELETE_RECORD

```json
{
  "id": "{step-uuid}",
  "name": "Delete old task",
  "type": "DELETE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 440 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "task",
      "objectRecordId": "{{trigger.properties.after.id}}"
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

---

### FIND_RECORDS

**`recordFilters` alone is not enough — `gqlOperationFilter` is required.** `recordFilters`/`recordFilterGroups` is the human-readable display filter; `gqlOperationFilter` (a GraphQL-style filter object) is what the engine actually executes the query with. Omitting it means the step silently doesn't filter as expected even though `recordFilters` looks correct.

**SELECT/MULTI_SELECT `value` must be a JSON-encoded array string** (e.g. `"[\"WON\"]"`), not a plain string like `"WON"`.

```json
{
  "id": "{step-uuid}",
  "name": "Find open opportunities",
  "type": "FIND_RECORDS",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "filter": {
        "recordFilters": [
          {
            "id": "filter-1",
            "type": "SELECT",
            "stepOutputKey": "stage",
            "operand": "IS_NOT",
            "value": "[\"WON\"]",
            "stepFilterGroupId": "group-1",
            "fieldMetadataId": "{stage-fieldMetadataId-from-metadataContext}"
          }
        ],
        "recordFilterGroups": [{ "id": "group-1", "logicalOperator": "AND" }],
        "gqlOperationFilter": { "and": [{ "stage": { "neq": "WON" } }] }
      },
      "orderBy": { "createdAt": "DescNullsLast" },
      "limit": 20
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

To filter on a composite sub-field (EMAILS, PHONES, LINKS, ADDRESS, CURRENCY, FULL_NAME), add `compositeFieldSubFieldName` to the filter (e.g. `"primaryEmail"` to target `emails.primaryEmail`).

This filter shape (`filter.recordFilters` + `recordFilterGroups` + `fieldMetadataId` + `gqlOperationFilter`) is different from CONDITION/FILTER's shape (`stepFilters` + `stepFilterGroups` + `stepOutputKey`, no `gqlOperationFilter`) — don't mix them up.

Output: `{{stepId.first.*}}` for top result, `{{stepId.all}}` for array, `{{stepId.totalCount}}`.

---

### SEND_EMAIL

```json
{
  "id": "{step-uuid}",
  "name": "Send welcome email",
  "type": "SEND_EMAIL",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "connectedAccountId": "{connectedAccountId-from-metadataContext}",
      "email": "{{trigger.properties.after.emails.primaryEmail}}",
      "subject": "Welcome, {{trigger.properties.after.name.firstName}}!",
      "body": "<p>Hi {{trigger.properties.after.name.firstName}}, welcome aboard.</p>"
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

---

### CONDITION

```json
{
  "id": "{step-uuid}",
  "name": "Check if VIP",
  "type": "CONDITION",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "stepFilterGroups": [{ "id": "group-1", "logicalOperator": "AND" }],
      "stepFilters": [
        {
          "id": "filter-1",
          "type": "TEXT",
          "stepOutputKey": "trigger.properties.after.jobTitle",
          "operand": "CONTAINS",
          "value": "VP",
          "stepFilterGroupId": "group-1"
        }
      ],
      "trueNextStepIds": ["{true-branch-step-uuid}"],
      "falseNextStepIds": ["{false-branch-step-uuid}"]
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

**CONDITION does not use top-level `nextStepIds`** — routing is via `trueNextStepIds` / `falseNextStepIds` inside `settings.input`.

**Creating a CONDITION auto-spawns two placeholder branch steps.** Calling `createWorkflowVersionStep` with `stepType: "CONDITION"` causes the API to create not one but *three* steps — the CONDITION plus an EMPTY placeholder for each branch, already wired as `trueNextStepIds`/`falseNextStepIds`. Either reuse the two placeholders as your true/false targets, or delete them (`deleteWorkflowVersionStep`, see Build Protocol step 5) before wiring your own steps. If you're diffing `stepsDiff` to count new steps, expect three, not one.

---

### FILTER

```json
{
  "id": "{step-uuid}",
  "name": "Only proceed if VP",
  "type": "FILTER",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "stepFilterGroups": [{ "id": "group-1", "logicalOperator": "AND" }],
      "stepFilters": [
        {
          "id": "filter-1",
          "type": "TEXT",
          "stepOutputKey": "{{trigger.properties.after.jobTitle}}",
          "operand": "CONTAINS",
          "value": "VP",
          "stepFilterGroupId": "group-1"
        }
      ]
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

**FILTER `stepOutputKey` uses `{{...}}` syntax with double curly braces** — different from CONDITION which uses a plain dot-path string. FILTER sits in the main step chain and stops execution if the condition doesn't match.

---

### ITERATOR

```json
{
  "id": "{step-uuid}",
  "name": "Loop over people",
  "type": "ITERATOR",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": ["{step-after-loop-uuid}"],
  "settings": {
    "input": {
      "items": "{{findPeopleStepUuid.all}}",
      "initialLoopStepIds": ["{first-step-inside-loop-uuid}"]
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

Inside loop steps: use `{{iteratorStepUuid.currentItem.id}}`, `{{iteratorStepUuid.currentItem.name.firstName}}`.

---

### DELAY

```json
{
  "id": "{step-uuid}",
  "name": "Wait 3 days",
  "type": "DELAY",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "delayType": "DURATION",
      "duration": { "days": 3, "hours": 0, "minutes": 0, "seconds": 0 }
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

For date-based delay: `"delayType": "SCHEDULED_DATE"` + `"scheduledDateTime": "{{stepId.value}}"` (must be ISO 8601).

---

### FORMULA / AGGREGATE_VALUES / RANDOM_NUMBER / ADJUST_TIME

```json
{ "type": "FORMULA", "settings": { "input": { "formula": "{{trigger.properties.after.amount.amountMicros}} * 0.9" } } }
{ "type": "AGGREGATE_VALUES", "settings": { "input": { "values": "{{findDealsStep.all}}", "operation": "SUM" } } }
{ "type": "RANDOM_NUMBER", "settings": { "input": { "minimum": 1, "maximum": 100 } } }
{ "type": "ADJUST_TIME", "settings": { "input": { "dateTime": "{{trigger.properties.after.closeDate}}", "today": false, "offset": 7, "unit": "DAYS" } } }
```

All return `{{stepId.value}}`.

---

### HTTP_REQUEST

```json
{
  "id": "{step-uuid}",
  "name": "Notify Slack",
  "type": "HTTP_REQUEST",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "url": "https://hooks.slack.com/services/xxx/yyy/zzz",
      "method": "POST",
      "headers": { "Content-Type": "application/json" },
      "body": { "text": "New person: {{trigger.properties.after.name.firstName}} {{trigger.properties.after.name.lastName}}" }
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

**`body` type is unconfirmed — both object and stringified-JSON-string forms appear in real exported steps** (e.g. provider integrations like SmartLead use a stringified string). The example above uses an object; if a step's `body` fails to send correctly, try a stringified JSON string instead. `{{...}}` interpolation works as a substring either way.

Output schema is dynamic — cannot be computed until a test run. Note this in return.

---

### CODE

```json
{
  "id": "{step-uuid}",
  "name": "Run enrichment",
  "type": "CODE",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "serverlessFunctionId": "{serverlessFunctionId-from-metadataContext}",
      "serverlessFunctionVersion": "latest",
      "serverlessFunctionInput": {
        "personId": "{{trigger.properties.after.id}}"
      }
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

Output schema is dynamic — populated after test execution.

---

### AI_CRM_AGENT

An AI agent step that reads/writes a target CRM record, guided by a prompt. Reverse-engineered from exported workflows — two open questions are noted below, don't invent answers for them.

```json
{
  "id": "{step-uuid}",
  "name": "Qualify lead",
  "type": "AI_CRM_AGENT",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "main": { "id": "{step-uuid}", "type": "entity", "nameSingular": "person" },
      "input": [
        { "{fieldMetadataId-from-metadataContext}": true }
      ],
      "prompt": "Review this person's recent activity and decide if they qualify as a warm lead.",
      "recordId": "{{trigger.properties.after.id}}",
      "includeReason": true,
      "businessContextDocumentIds": []
    },
    "outputSchema": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } }
  }
}
```

**`input` fields:**

| Field | Type | Description |
|---|---|---|
| `main` | object | `{ id, type: "entity", nameSingular }` — the target object the agent operates on |
| `input` | array | Per-field boolean maps keyed by `fieldMetadataId` — **unclear whether these gate read access, write access, or both; verify against a live run before relying on this to control writes** |
| `prompt` | string | The instruction given to the agent; supports `{{...}}` variables |
| `recordId` | string | UUID (or `{{...}}` expression) of the record the agent acts on |
| `includeReason` | boolean | Whether the agent's reasoning is included in its output |
| `businessContextDocumentIds` | string[] | IDs of business-context documents attached to the agent — **how these documents are created/managed is undocumented; confirm before depending on it** |

---

### ENRICH

ENRICH has **two distinct variants** depending on `input` shape:

**Variant A — freeform AI prompt:**
```json
{ "type": "ENRICH", "settings": { "input": { "body": "Summarize {{trigger.properties.after.name.firstName}}'s background at {{trigger.properties.after.company.name}}." } } }
```
Output: empty — no structured schema, results applied externally.

**Variant B — provider enrichment** (LinkedIn, email finder, etc.) — a completely different shape:
```json
{
  "type": "ENRICH",
  "settings": {
    "input": {
      "operations": ["LINKEDIN_BASIC_DETAILS"],
      "enrichOptions": {},
      "selectedOptions": ["LINKEDIN_BASIC_DETAILS"],
      "overwrite": false,
      "objectRecordId": "{{trigger.properties.after.id}}",
      "objectNameSingular": "person"
    }
  }
}
```
Valid `operations` values include `LINKEDIN_BASIC_DETAILS`, `LINKEDIN_POSTS_PERSON`, `LINKEDIN_REACTIONS`, `EMAIL_FINDER`. Output **is** structured here — read `{{stepId.options.<opId>.success}}` (e.g. `{{enrichStepId.options.liBasicDetails.success}}`) downstream to check whether an operation succeeded.

---

## Return to Orchestrator

```json
{
  "stepsBuilt": [
    { "order": 1, "id": "step-uuid-1", "type": "CREATE_RECORD", "name": "Create task", "valid": true },
    { "order": 2, "id": "step-uuid-2", "type": "SEND_EMAIL", "name": "Send welcome email", "valid": true }
  ],
  "dynamicOutputSteps": ["step-uuid-X"],
  "errors": [],
  "warnings": [
    "step-uuid-X is HTTP_REQUEST — output schema requires a test run before downstream steps can reference its output"
  ]
}
```

---

## Critical Gotchas

1. **`createWorkflowVersionStep` always creates a ghost** — `objectName: "workflow"`, `valid: false`. Always immediately follow with `updateWorkflowVersionStep` with the correct config. Never skip this.
2. **`updateWorkflowVersionStep` replaces the entire step** — always send the full step object. Partial updates silently clear omitted fields.
3. **`parentStepId` is `"trigger"` (string) for step 1** — not the trigger's UUID. Only for the very first step.
4. **CONDITION routing is in `settings.input`** — `trueNextStepIds` and `falseNextStepIds` are inside settings. The top-level `nextStepIds` must be `[]` for CONDITION.
5. **FILTER uses `{{...}}` in `stepOutputKey`** — unlike CONDITION which uses a plain dot-path. Use double curly braces in FILTER's `stepOutputKey`.
6. **Set relation fields inline — never a separate CREATE_RECORD** — `taskTargets` and `noteTargets` go inside `objectRecord` directly.
7. **`fieldMetadataId` must come from the metadata-agent, not from `computeStepOutputSchema`** — use the `/metadata` endpoint with `fieldsList` to get field UUIDs. Never use `computeStepOutputSchema` as a workaround to discover field IDs — that adds an extra roundtrip and returns only the schema shape, not the metadata you need.
8. **CONDITION auto-creates two EMPTY child steps** — `createWorkflowVersionStep` with `stepType: "CONDITION"` creates the condition plus two placeholder branch steps (already set as `trueNextStepIds`/`falseNextStepIds`). Reuse or delete them before wiring real branches; expect three new steps, not one, when diffing `stepsDiff`.
9. **ITERATOR auto-creates an EMPTY child step** — when `createWorkflowVersionStep` is called with `stepType: "ITERATOR"`, the API automatically creates an EMPTY placeholder step inside the loop body and sets it as `initialLoopStepIds[0]`. You must either: (a) delete it and replace with your real first loop step, or (b) set `initialLoopStepIds` in your `updateWorkflowVersionStep` to point to the real first loop step instead. The EMPTY step has `valid: true` but does nothing — leaving it is not harmful but wastes a slot.
10. **`fieldsToUpdate` in UPDATE_RECORD must match `objectRecord` keys** — missing = silently ignored. Extra = error.
11. **Do not touch existing steps** — when adding a step to a workflow that already has steps, only create the new step and update edges. Never re-issue `updateWorkflowVersionStep` on steps that should not change.
12. **Before returning, verify no ghost steps remain** — check that all created steps have `valid: true` and `objectName != "workflow"`. Report any found ghosts in `errors`.
13. **Check for stale ghost steps before starting** — when editing an existing DRAFT version, fetch the version first and delete any pre-existing invalid steps (`valid: false`) before adding new ones. Stale ghosts accumulate from previous interrupted builds and will block activation.
