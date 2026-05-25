---
name: sequence-triggers
description: Reference for Dalil AI sequence enrollment triggers and START step configuration — how people enter sequences and how the entry step is structured.
---

# Dalil AI: Sequence Enrollment & Triggers Reference

## How People Enter a Sequence

Unlike workflows (which are triggered by database events, webhooks, or schedules), sequences are enrolled manually or via automation. There is no external "trigger" object on the sequence itself — enrollment is the trigger.

### Enrollment Methods

| Method | How it works |
|---|---|
| **Manual bulk-add** | User selects people in the CRM UI and adds them to a sequence |
| **CREATE_SEQUENCE_PERSON step** | A step in another sequence enrolls a person automatically |
| **TRIGGER_WORKFLOW + workflow** | A workflow step fires which then enrolls a person |
| **REST API direct** | POST to `/rest/sequencePeople` with `personId` + `sequenceId` |

There is no "auto-enroll when person is created" trigger natively on sequences. To achieve that, create a workflow with a DATABASE_EVENT trigger on `person.created` and add a step that calls the sequence enrollment.

---

## The START Step

Every sequence must have exactly one `START` step. It is:
- Always the first step (`isFirstStep: true`)
- Has no input fields (empty settings)
- Has one `nextStepId` pointing to the first real action step
- Its UUID becomes the entry point when a `SequenceRun` begins

### START step structure
```json
{
  "id": "start-step-uuid",
  "name": "Start",
  "type": "START",
  "isFirstStep": true,
  "valid": true,
  "nextStepIds": ["first-action-step-uuid"],
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
```

---

## Sequence-Level Settings

For the full `SequenceLimitSettings` schema — platform daily limits, send windows, timezone, pause behavior, threading, and workflow integrations — see:
**`sequence-skills/sequence-settings/SKILL.md`**

---

## Enrollment via REST (direct)

To manually enroll a person into a sequence:

```bash
curl -s -X POST "https://app.usedalil.ai/rest/sequencePeople" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "personId": "person-uuid",
    "sequenceId": "sequence-uuid"
  }'
```

Response: `.data.sequencePerson` with `id`, `status: "ACTIVE"`.

To pause or stop a person:
```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequencePeople/{sequencePersonId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "status": "PAUSED" }'
```

---

## Sequence Status vs Person Status

| Status | Level | Meaning |
|---|---|---|
| `DRAFT` | Sequence | Being built — no runs execute |
| `ACTIVE` | Sequence | Running — new enrollments start processing |
| `DEACTIVATED` | Sequence | Stopped — no new or continuing runs |
| `ACTIVE` | SequencePerson | This person is actively progressing |
| `PAUSED` | SequencePerson | This person's run is paused (e.g., they replied) |

To activate a sequence (make it live):
```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "status": "ACTIVE" }'
```

---

## Sender Assignment

Before a sequence can send messages, senders must be assigned. Each platform (Email, WhatsApp, LinkedIn) needs at least one `SequenceSender` record.

Fetch available senders:
```bash
curl -s -G "https://app.usedalil.ai/rest/sequenceSenders" \
  -H "Authorization: Bearer {apiKey}"
```

Response per sender:
```json
{
  "id": "uuid",
  "platformType": "EMAIL",
  "identifier": "alice@company.com",
  "messageStatus": "ACTIVE",
  "workspaceMember": { "id": "uuid", "name": { "firstName": "Alice" } }
}
```

Activate a sender's permission for a specific action:
```graphql
mutation {
  activateSequenceSenders(input: {
    senderActions: [
      { sequenceSenderId: "uuid", actionType: "message" }
    ]
  })
}
```

---

## Common Enrollment Patterns

### Pattern 1: Manual outreach campaign
1. Build sequence (DRAFT)
2. Add senders
3. Activate sequence (`status: ACTIVE`)
4. Bulk-enroll people from CRM UI

### Pattern 2: Auto-enroll on workflow event
1. Create a workflow with DATABASE_EVENT trigger on `person.created`
2. Add a CREATE_SEQUENCE_PERSON or REST call step in the workflow
3. Sequence processes new people automatically

### Pattern 3: Cross-sequence enrollment
Use `CREATE_SEQUENCE_PERSON` step in sequence A to enroll the person in sequence B when they reach a certain step.

---

## Gotchas

1. **Sequence must be ACTIVE to execute** — a DRAFT sequence will not run even if people are enrolled.
2. **START step is required** — sequences without a START step will fail validation. Always create START as the first step.
3. **No global trigger type** — unlike workflows, there's no `trigger.type` field on the sequence. Enrollment method is external to the sequence definition.
4. **`senderAssignStatus`** — after activating senders, check `sequence.senderAssignStatus === "COMPLETED"` before enrolling people. If `FAILED` or `PENDING`, senders aren't ready.
5. **Sequence settings live in `sequence-settings/SKILL.md`** — for all `SequenceLimitSettings` fields including `pauseOnReply`, `businessDaysOnly`, threading, and workflow trigger integrations.
