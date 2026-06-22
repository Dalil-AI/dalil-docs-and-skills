---
name: sequence-actions
description: Reference for all 33 Dalil AI sequence action types — settings schemas, required fields, nextStepIds routing, and platform-specific configuration for building sequence step graphs.
---

# Dalil AI: Sequence Action Types Reference

## Base Step Structure

Every step in a sequence conforms to:

```json
{
  "id": "uuid",
  "name": "Human-readable step name",
  "type": "SEND_EMAIL",
  "isFirstStep": false,
  "valid": true,
  "nextStepIds": ["uuid-of-next-step"],
  "position": { "x": 0, "y": 200 },
  "settings": {
    "input": { ... },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

**Routing:** `nextStepIds` is an array. Most steps have one next step. Branching steps (CUSTOM_CONDITION, HAS_* checks) have two: `[trueNextStepId, falseNextStepId]` — but branching is encoded in the step's `settings.input`, not just `nextStepIds`.

---

## Action Categories

### 1. Entry

#### START
The always-first step. Every sequence must have exactly one START step with `isFirstStep: true`.

```json
{
  "type": "START",
  "isFirstStep": true,
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

### 2. Messaging — Email

#### SEND_EMAIL
Sends an email to the enrolled person.

```json
{
  "type": "SEND_EMAIL",
  "settings": {
    "input": {
      "subject": "Hi {{person.name.firstName}}, quick question",
      "body": "<p>Hello {{person.name.firstName}},</p><p>...</p>",
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

Fields:
- `subject` — email subject line (supports `{{...}}` variables)
- `body` — HTML email body (supports `{{...}}` variables)
- `days` — array of DayOfWeek integers: `1`=Mon, `2`=Tue, …, `7`=Sun (optional, defaults to all days)
- `window` — time window `{ start: "HH:mm", end: "HH:mm" }` in workspace timezone (optional)

---

### 3. Messaging — LinkedIn

#### SEND_LINKEDIN
Sends a LinkedIn direct message.

```json
{
  "type": "SEND_LINKEDIN",
  "settings": {
    "input": {
      "message": "Hi {{person.name.firstName}}, ...",
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

#### SEND_LINKEDIN_CONNECTION
Sends a LinkedIn connection request (with optional note).

```json
{
  "type": "SEND_LINKEDIN_CONNECTION",
  "settings": {
    "input": {
      "message": "Hi {{person.name.firstName}}, I'd love to connect.",
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
Note: `message` is optional (max 300 chars for connection notes).

#### VISIT_LINKEDIN_PROFILE
Visits the person's LinkedIn profile (engagement signal).

```json
{
  "type": "VISIT_LINKEDIN_PROFILE",
  "settings": {
    "input": {
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

#### LIKE_LINKEDIN_POST
Likes one of the person's LinkedIn posts.

```json
{
  "type": "LIKE_LINKEDIN_POST",
  "settings": {
    "input": {
      "type": "like",
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
`type` values: `"like"` | `"celebrate"` | `"support"` | `"love"` | `"insightful"` | `"funny"`

#### COMMENT_LINKEDIN_POST
Comments on one of the person's LinkedIn posts.

```json
{
  "type": "COMMENT_LINKEDIN_POST",
  "settings": {
    "input": {
      "message": "Great insights, {{person.name.firstName}}!",
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

---

### 4. Messaging — WhatsApp

#### SEND_WHATSAPP
Sends a WhatsApp message.

```json
{
  "type": "SEND_WHATSAPP",
  "settings": {
    "input": {
      "message": "Hi {{person.name.firstName}}, ...",
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

---

### 5. Flow Control

#### DELAY
Pauses execution for a specified duration.

```json
{
  "type": "DELAY",
  "settings": {
    "delay": {
      "unit": "DAY",
      "value": 3
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```
`unit` values: `"MINUTE"` | `"HOUR"` | `"DAY"`

Note: For DELAY, the delay config is at `settings.delay`, NOT `settings.input.delay`.

#### CUSTOM_CONDITION
Branches based on filter rules applied to the person record.

```json
{
  "type": "CUSTOM_CONDITION",
  "settings": {
    "input": {
      "recordFilterGroups": { ... },
      "recordFilters": { ... },
      "trueNextStepIds": ["step-uuid-if-true"],
      "falseNextStepIds": ["step-uuid-if-false"]
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

---

### 6. Contact Property Checks

These steps branch based on whether the person has a specific contact property. All return true/false routing.

| Action Type | Checks for |
|---|---|
| `HAS_EMAIL_ADDRESS` | Person has a primary email set |
| `HAS_PHONE_NUMBER` | Person has a phone number set |
| `HAS_LINKEDIN_URL` | Person has a LinkedIn URL set |
| `HAS_LINKEDIN_CONNECTION` | Person is connected on LinkedIn with the sender |

Settings template (same for all four):
```json
{
  "type": "HAS_EMAIL_ADDRESS",
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
These steps have TWO next step IDs in `nextStepIds`: `[trueStepId, falseStepId]`.

---

### 7. Event Detection (Engagement Listeners)

These steps wait for an engagement event from the person and branch accordingly. They are placed after a send step and act as listeners.

| Action Type | Listens for |
|---|---|
| `OPENED_EMAIL` | Email was opened |
| `CLICKED_EMAIL_LINK` | Link in email was clicked |
| `REPLIED_TO_EMAIL` | Person replied to the email |
| `UNSUBSCRIBED_EMAIL` | Person unsubscribed |
| `OPENED_LINKEDIN_MESSAGE` | LinkedIn DM was opened |
| `REPLIED_TO_LINKEDIN_MESSAGE` | Person replied to LinkedIn DM |
| `ACCEPTED_LINKEDIN_CONNECTION` | Person accepted connection request |
| `OPENED_WHATSAPP_MESSAGE` | WhatsApp message was opened |
| `REPLIED_TO_WHATSAPP_MESSAGE` | Person replied to WhatsApp message |
| `CREATED_CALENDAR_EVENT` | A calendar event was created with the person |
| `CREATED_AI_MEETING_NOTE` | An AI meeting note was created |

Settings template (same for all):
```json
{
  "type": "OPENED_EMAIL",
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
Typically paired with a DELAY step before them: DELAY → OPENED_EMAIL (did they open in 3 days?).

---

### 8. CRM Record Actions

#### CREATE_RECORD
Creates a record of any object type.

```json
{
  "type": "CREATE_RECORD",
  "settings": {
    "input": {
      "objectName": "task",
      "objectRecord": {
        "title": "Follow up with {{person.name.firstName}}",
        "status": "TODO",
        "assigneeId": "{{workspaceMemberId}}"
      }
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```
`objectRecord` keys must match the object's field names. Use `fieldMetadataIds` from the metadata-agent for fields that require UUID references.

#### UPDATE_RECORD
Updates an existing record.

```json
{
  "type": "UPDATE_RECORD",
  "settings": {
    "input": {
      "objectName": "person",
      "objectRecord": {
        "id": "{{person.id}}",
        "jobTitle": "CTO"
      }
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

#### CREATE_TASK
Creates a task (shorthand, no `objectName` required).

```json
{
  "type": "CREATE_TASK",
  "settings": {
    "input": {
      "objectRecord": {
        "title": "Follow up with {{person.name.firstName}}",
        "status": "TODO",
        "dueAt": "2025-12-31T00:00:00.000Z"
      }
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

#### CREATE_NOTE
Creates a note.

```json
{
  "type": "CREATE_NOTE",
  "settings": {
    "input": {
      "objectRecord": {
        "title": "Sequence note — {{person.name.firstName}}",
        "body": "Person enrolled in outreach sequence."
      }
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

#### CREATE_SEQUENCE_PERSON
Enrolls the person into another sequence.

```json
{
  "type": "CREATE_SEQUENCE_PERSON",
  "settings": {
    "input": {
      "sequenceId": "target-sequence-uuid"
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": true }
    },
    "outputSchema": {}
  }
}
```

---

### 9. Integration

#### TRIGGER_WORKFLOW
Fires a Dalil workflow for this person.

```json
{
  "type": "TRIGGER_WORKFLOW",
  "settings": {
    "input": {
      "workflowId": "workflow-uuid",
      "workflowVersionId": "version-uuid"
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": true },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

---

### 10. Utility

#### COMMENT
Internal annotation on the sequence canvas. Not executed.

```json
{
  "type": "COMMENT",
  "settings": {
    "input": {
      "body": "This branch handles people who haven't replied after 7 days"
    },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    },
    "outputSchema": {}
  }
}
```

#### EMPTY
Placeholder step. Used as a dead-end or pending step. Not executed.

```json
{
  "type": "EMPTY",
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

## Step Graph Construction Pattern

Steps are connected via `nextStepIds`. For a linear sequence:

```
START → SEND_EMAIL → DELAY (3 days) → REPLIED_TO_EMAIL → CREATE_TASK
                                       ↓ (if not replied)
                                      SEND_EMAIL (follow-up)
```

When building a branch, the detection step (`REPLIED_TO_EMAIL`) has:
```json
"nextStepIds": ["create-task-step-uuid", "follow-up-email-step-uuid"]
```
The `trueNextStepIds` / `falseNextStepIds` inside `settings.input` mirror this for CUSTOM_CONDITION and HAS_* checks.

---

## Scheduling Fields (days + window)

All messaging steps (SEND_EMAIL, SEND_LINKEDIN, SEND_WHATSAPP, SEND_LINKEDIN_CONNECTION, VISIT_LINKEDIN_PROFILE, LIKE_LINKEDIN_POST, COMMENT_LINKEDIN_POST) accept:

```json
"days": [1, 2, 3, 4, 5],      // DayOfWeek: 1=Mon through 7=Sun
"window": { "start": "09:00", "end": "17:00" }
```

Both are optional. If omitted, the sequence-level `settings.activeWindow` applies.

---

## Error Handling Options

All steps require:
```json
"errorHandlingOptions": {
  "retryOnFailure": { "value": true },
  "continueOnFailure": { "value": false }
}
```
- `retryOnFailure`: retry the step if it fails (recommended for sending steps)
- `continueOnFailure`: move to next step even if this step fails (useful for non-critical steps like CREATE_NOTE)

---

## Gotchas

1. **DELAY uses `settings.delay`, not `settings.input`** — unlike all other steps where config goes in `settings.input`.
2. **START must be `isFirstStep: true`** — only one step per sequence can have this flag.
3. **Detection steps (OPENED_EMAIL etc.) are not triggers** — they are steps in the flow that wait and branch. They require a preceding send step.
4. **`valid: false` blocks execution** — set all required fields or the step won't run.
5. **HAS_* steps produce two branches** — always wire both `nextStepIds[0]` (true path) and `nextStepIds[1]` (false path).
6. **`outputSchema: {}`** — sequences don't use `computeStepOutputSchema` like workflows. Leave as empty object.
7. **Platform limits apply globally** — even if step settings allow sending any day, the sequence-level `settings` (SequenceLimitSettings) can cap daily volume.
