---
name: sequence-agent-trigger
description: Sub-agent for configuring the Dalil AI sequence START step and enrollment trigger context. Called by sequence-agent orchestrator; never invoke directly.
---

# Dalil AI: Sequence Trigger Sub-Agent

## Purpose

Configures the sequence's entry point — the START step and any enrollment-related settings. Also determines what person-context variables will be available to all steps.

Runs in Phase B (after lifecycle-agent has created the sequence). Do not invoke directly — spawned by `sequence-agent`.

---

## Inputs (from orchestrator)

For the full `sequenceSettings` schema, see: **`sequence-skills/sequence-settings/SKILL.md`**

```json
{
  "apiKey": "...",
  "sequenceId": "uuid",
  "enrollmentType": "MANUAL_BULK | AUTO_VIA_WORKFLOW | CROSS_SEQUENCE",
  "sequenceSettings": {
    "pauseOnReply": true,
    "businessDaysOnly": true,
    "activeWindow": { "days": [1,2,3,4,5], "window": { "start": "09:00", "end": "17:00" } },
    "timezone": "Europe/Paris",
    "isEmailSingleThread": true,
    "email": { "dailyEmails": 50 },
    "linkedIn": { "dailyConnectionRequests": 20, "dailyMessages": 15 }
  }
}
```

---

## Step-by-step execution

### Step 1: Create the START step

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateSequenceStep($input: CreateSequenceStepInput!) { createSequenceStep(input: $input) { stepsDiff } }",
    "variables": {
      "input": {
        "sequenceId": "{sequenceId}",
        "stepType": "START",
        "isFirstStep": true,
        "position": { "x": 0, "y": 0 }
      }
    }
  }'
```

After creation, fetch the sequence to get the START step's auto-generated UUID:

```bash
curl -s -G "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

Find the step with `isFirstStep: true` in `response.data.sequence.steps`. That is the START step UUID.

### Step 2: Update the START step with full settings

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateSequenceStep($input: UpdateSequenceStepInput!) { updateSequenceStep(input: $input) { id name type valid } }",
    "variables": {
      "input": {
        "sequenceId": "{sequenceId}",
        "step": {
          "id": "{startStepId}",
          "name": "Start",
          "type": "START",
          "isFirstStep": true,
          "valid": true,
          "nextStepIds": [],
          "position": { "x": 0, "y": 0 },
          "settings": {
            "input": {},
            "errorHandlingOptions": {
              "retryOnFailure": { "value": false },
              "continueOnFailure": { "value": false }
            },
            "outputSchema": {}
          }
        }
      }
    }
  }'
```

Note: `nextStepIds` is left empty here — the actions-agent will connect the START step to the first real step.

### Step 3: Apply sequence-level settings

If `sequenceSettings` was provided, PATCH the sequence record:

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "pauseOnReply": true,
      "businessDaysOnly": true,
      "activeWindow": {
        "days": [1, 2, 3, 4, 5],
        "window": { "start": "09:00", "end": "17:00" }
      },
      "timezone": "Europe/Paris",
      "isEmailSingleThread": true,
      "email": { "dailyEmails": 50 }
    }
  }'
```

---

## Return JSON

```json
{
  "startStepId": "start-step-uuid",
  "startStepCreated": true,
  "sequenceSettingsApplied": true,
  "enrollmentType": "MANUAL_BULK",
  "variableContext": {
    "alwaysAvailable": [
      "{{person.id}}",
      "{{person.name.firstName}}",
      "{{person.name.lastName}}",
      "{{person.emails.primaryEmail}}",
      "{{person.phones.primaryPhoneNumber}}",
      "{{person.linkedinLink.url}}",
      "{{person.jobTitle}}",
      "{{person.city}}",
      "{{person.company.name}}",
      "{{sequencePerson.id}}",
      "{{sequencePerson.status}}",
      "{{sequencePerson.hasReplied}}"
    ]
  },
  "status": "success",
  "error": null
}
```

---

## Enrollment Type Context

The `enrollmentType` from the orchestrator is informational — it doesn't change the START step configuration. It informs the user-facing explanation:

| enrollmentType | Meaning | Impact on steps |
|---|---|---|
| `MANUAL_BULK` | Users manually add people from CRM UI | No special steps needed |
| `AUTO_VIA_WORKFLOW` | A workflow enrolls people on database events | Document in a COMMENT step |
| `CROSS_SEQUENCE` | Another sequence's CREATE_SEQUENCE_PERSON step enrolls people | Document in a COMMENT step |

If `enrollmentType` is `AUTO_VIA_WORKFLOW` or `CROSS_SEQUENCE`, add a COMMENT step after START explaining the enrollment source. This helps future readers of the sequence.

---

## Gotchas

1. **START step `nextStepIds` starts empty** — the trigger-agent leaves it empty. The actions-agent connects it when building the first real step via `updateSequenceStep`.
2. **`isFirstStep: true` must be set in `createSequenceStep` AND in `updateSequenceStep`** — set it in both to ensure it sticks.
3. **Sequence settings are separate from step settings** — apply them via PATCH on the sequence record, not as part of the step mutation.
4. **`createSequenceStep` returns `stepsDiff`, not the step UUID** — always fetch the sequence after creation to get the new step's ID.
5. **`pauseOnReply: true` is almost always desired** — if the user doesn't specify, default to true. Continuing to send to someone who already replied is a common mistake.
