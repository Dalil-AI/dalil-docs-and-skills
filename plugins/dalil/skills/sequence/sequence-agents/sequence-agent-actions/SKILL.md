---
name: sequence-agent-actions
description: Sub-agent for building all steps in a Dalil AI sequence — creates, configures, and connects each step in order using createSequenceStep and updateSequenceStep mutations. Called by sequence-agent orchestrator; never invoke directly.
---

# Dalil AI: Sequence Actions Sub-Agent

## Purpose

Builds every step in the sequence in the correct order. For each step:
1. `createSequenceStep` — generate the step's UUID
2. Fetch the sequence to get the new step's UUID
3. `updateSequenceStep` — configure the step with full settings, variables, and `nextStepIds`

Runs in Phase C (after lifecycle, metadata, trigger, and variables agents complete). Steps must be built sequentially — each step's UUID is needed to wire `nextStepIds` of the previous step.

Do not invoke directly — spawned by `sequence-agent`.

---

## Inputs (from orchestrator)

```json
{
  "apiKey": "...",
  "sequenceId": "uuid",
  "startStepId": "start-step-uuid",
  "metadataContext": {
    "senders": {
      "email": [{ "id": "sender-uuid", "identifier": "alice@co.com" }],
      "linkedin": [],
      "whatsapp": []
    },
    "fieldMetadataIds": { "person.jobTitle": "field-uuid" },
    "workspaceMembers": [{ "id": "member-uuid", "userEmail": "alice@co.com" }],
    "targetSequences": {}
  },
  "variablesMap": {
    "step_1": { "availableVariables": ["{{person.name.firstName}}", ...] },
    "step_2": { "availableVariables": ["{{person.name.firstName}}", ...] }
  },
  "stepsToBuild": [
    {
      "order": 2,
      "type": "SEND_EMAIL",
      "name": "Intro email",
      "config": {
        "subject": "Hi {{person.name.firstName}}",
        "body": "<p>Hi {{person.name.firstName}},</p><p>I noticed you work at {{person.company.name}}...</p>",
        "days": [1, 2, 3, 4, 5],
        "window": { "start": "09:00", "end": "17:00" }
      }
    },
    {
      "order": 3,
      "type": "DELAY",
      "name": "Wait 3 days",
      "config": { "unit": "DAY", "value": 3 }
    }
  ]
}
```

Note: `stepsToBuilder` starts at order 2 (order 1 is START, already created by trigger-agent).

---

## Step-by-step execution

For each step in `stepsToBuilder` (in order):

### Phase 1: Create the step

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateSequenceStep($input: CreateSequenceStepInput!) { createSequenceStep(input: $input) { stepsDiff } }",
    "variables": {
      "input": {
        "sequenceId": "{sequenceId}",
        "stepType": "{stepType}",
        "isFirstStep": false,
        "position": { "x": 0, "y": {order * 200} }
      }
    }
  }'
```

### Phase 2: Fetch sequence to get new step UUID

```bash
curl -s -G "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

Find the newly created step in `response.data.sequence.steps`. It will be the step with matching `type` and no `name` yet (or the step that doesn't have an ID already assigned in `stepsBuilt`). Save its `id` as the current step's UUID.

### Phase 3: Update the step with full configuration

Build the `step` object according to the step type (see settings schemas below), then call:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateSequenceStep($input: UpdateSequenceStepInput!) { updateSequenceStep(input: $input) { id name type valid nextStepIds } }",
    "variables": {
      "input": {
        "sequenceId": "{sequenceId}",
        "step": { ... full step object ... }
      }
    }
  }'
```

### Phase 4: Wire previous step to this step

After creating step N, update step N-1's `nextStepIds` to point to step N's UUID:

```bash
# Update previous step to add nextStepId pointing to current step
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateSequenceStep($input: UpdateSequenceStepInput!) { updateSequenceStep(input: $input) { id nextStepIds } }",
    "variables": {
      "input": {
        "sequenceId": "{sequenceId}",
        "step": {
          "id": "{previousStepId}",
          "nextStepIds": ["{currentStepId}"],
          ... (other fields unchanged)
        }
      }
    }
  }'
```

For the first step after START: update the START step's `nextStepIds` to point to step 2.

---

## Settings Templates by Step Type

### SEND_EMAIL
```json
{
  "id": "{stepId}",
  "name": "{stepName}",
  "type": "SEND_EMAIL",
  "isFirstStep": false,
  "valid": true,
  "nextStepIds": [],
  "position": { "x": 0, "y": 200 },
  "settings": {
    "input": {
      "subject": "{subject with {{variables}}}",
      "body": "{html body with {{variables}}}",
      "days": [1, 2, 3, 4, 5],
      "window": { "start": "09:00", "end": "17:00" }
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

### SEND_LINKEDIN / SEND_WHATSAPP
```json
{
  "settings": {
    "input": {
      "message": "{message with {{variables}}}",
      "days": [1, 2, 3, 4, 5],
      "window": { "start": "09:00", "end": "17:00" }
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### SEND_LINKEDIN_CONNECTION
```json
{
  "settings": {
    "input": {
      "message": "{optional connection note}",
      "days": [1, 2, 3, 4, 5],
      "window": { "start": "09:00", "end": "17:00" }
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### VISIT_LINKEDIN_PROFILE
```json
{
  "settings": {
    "input": { "days": [1, 2, 3, 4, 5], "window": { "start": "09:00", "end": "17:00" } },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### LIKE_LINKEDIN_POST
```json
{
  "settings": {
    "input": { "type": "like", "days": [1, 2, 3, 4, 5], "window": { "start": "09:00", "end": "17:00" } },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### COMMENT_LINKEDIN_POST
```json
{
  "settings": {
    "input": { "message": "{comment}", "days": [1, 2, 3, 4, 5], "window": { "start": "09:00", "end": "17:00" } },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### DELAY
```json
{
  "settings": {
    "delay": { "unit": "DAY", "value": 3 },
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```
**Critical:** DELAY config goes in `settings.delay`, NOT `settings.input`.

### CUSTOM_CONDITION
```json
{
  "settings": {
    "input": {
      "recordFilterGroups": {},
      "recordFilters": {},
      "trueNextStepIds": ["{step-uuid-if-true}"],
      "falseNextStepIds": ["{step-uuid-if-false}"]
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### HAS_EMAIL_ADDRESS / HAS_PHONE_NUMBER / HAS_LINKEDIN_URL / HAS_LINKEDIN_CONNECTION
```json
{
  "settings": {
    "input": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```
These steps must have `nextStepIds` with TWO entries: `[trueStepId, falseStepId]`.

### Event detection steps (OPENED_EMAIL, REPLIED_TO_EMAIL, etc.)
```json
{
  "settings": {
    "input": {},
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```
Wire as branches from the DELAY that precedes them. They have two `nextStepIds`: `[yesStepId, noStepId]`.

### CREATE_TASK
```json
{
  "settings": {
    "input": {
      "objectRecord": {
        "title": "Follow up with {{person.name.firstName}}",
        "status": "TODO",
        "assigneeId": "{workspaceMemberId}"
      }
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### CREATE_NOTE
```json
{
  "settings": {
    "input": {
      "objectRecord": {
        "title": "Note for {{person.name.firstName}}",
        "body": "{body text}"
      }
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### CREATE_RECORD
```json
{
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecord": {
        "name": "Deal — {{person.company.name}}",
        "stageId": "{stage-fieldMetadataId}"
      }
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### UPDATE_RECORD
```json
{
  "settings": {
    "input": {
      "objectName": "person",
      "objectRecord": {
        "id": "{{person.id}}",
        "jobTitle": "Engaged Lead"
      }
    },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### CREATE_SEQUENCE_PERSON
```json
{
  "settings": {
    "input": { "sequenceId": "{targetSequenceId}" },
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": true } },
    "outputSchema": {}
  }
}
```

### TRIGGER_WORKFLOW
```json
{
  "settings": {
    "input": { "workflowId": "{workflowId}", "workflowVersionId": "{versionId}" },
    "errorHandlingOptions": { "retryOnFailure": { "value": true }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

### COMMENT
```json
{
  "settings": {
    "input": { "body": "{internal note text}" },
    "errorHandlingOptions": { "retryOnFailure": { "value": false }, "continueOnFailure": { "value": false } },
    "outputSchema": {}
  }
}
```

---

## Return JSON

```json
{
  "stepsBuilt": [
    { "order": 1, "id": "start-step-uuid", "type": "START", "name": "Start" },
    { "order": 2, "id": "step-2-uuid", "type": "SEND_EMAIL", "name": "Intro email" },
    { "order": 3, "id": "step-3-uuid", "type": "DELAY", "name": "Wait 3 days" },
    { "order": 4, "id": "step-4-uuid", "type": "REPLIED_TO_EMAIL", "name": "Did they reply?" },
    { "order": 5, "id": "step-5-uuid", "type": "CREATE_TASK", "name": "Create follow-up task" }
  ],
  "errors": [],
  "warnings": [
    "SEND_EMAIL step has no HAS_EMAIL_ADDRESS guard — people without email will fail at this step"
  ],
  "status": "success"
}
```

---

## Handling Branches

For branching steps (HAS_*, CUSTOM_CONDITION, detection steps), both branches must be built before wiring:

1. Build the branch step (e.g., `REPLIED_TO_EMAIL`) — don't set `nextStepIds` yet
2. Build both branch outcome steps (e.g., step-if-yes, step-if-no)
3. Go back and update the branch step with `nextStepIds: [yes-step-id, no-step-id]`

---

## Gotchas

1. **`createSequenceStep` returns `stepsDiff`, not the UUID** — always fetch the sequence after each creation to get the new step's UUID.
2. **DELAY uses `settings.delay`, not `settings.input`** — do not put delay config under `input`.
3. **`nextStepIds` must be set in `updateSequenceStep`** — leaving them empty creates dead-end steps.
4. **Wire backwards too** — after building step N, go back and update step N-1 to point to step N's UUID.
5. **`valid: true` must be set explicitly** — when all required settings fields are filled in, set `valid: true` in the updateSequenceStep call.
6. **Don't re-configure the START step** — it was already built by the trigger-agent. Just update its `nextStepIds` to point to step 2.
7. **HAS_* and detection steps have TWO `nextStepIds`** — both branches must lead somewhere. Wire both to valid steps (or EMPTY if one branch is intentionally a dead end).
8. **Use variables from `variablesMap`** — don't guess variable paths. Use the paths provided by the variables-agent for each step's order position.
