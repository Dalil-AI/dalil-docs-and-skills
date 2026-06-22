---
name: sequence
description: Manage Dalil AI sequences — create, read, update, delete, and run outreach sequences including email, WhatsApp, and LinkedIn campaigns with step management and sender configuration.
---

# Dalil AI: Sequence API Reference

## Overview

Sequences are multi-channel outreach automation flows for CRM campaigns (Email, WhatsApp, LinkedIn). Each sequence is a step-graph where a person is enrolled and moves through steps according to `nextStepIds` routing.

**Key concepts:**
- **Sequence** — the container (`status`: DRAFT | ACTIVE | DEACTIVATED)
- **Step** — a node in the flow graph (`type`: one of 33 action types)
- **SequencePerson** — a person enrolled in a sequence
- **SequenceRun** — execution instance per enrolled person
- **SequenceSender** — platform identity (email/WhatsApp/LinkedIn) assigned to workspace members

---

## REST Endpoints

Base URL: `https://app.usedalil.ai`
Auth: `Authorization: Bearer {apiKey}`

### Sequence CRUD

| Operation | Method | Path | Notes |
|---|---|---|---|
| Get one | GET | `/rest/sequences/{id}` | Add `?depth=1` to include steps |
| List | GET | `/rest/sequences` | Use filters — never fetch all |
| Create | POST | `/rest/sequences` | Status defaults to DRAFT |
| Update | PATCH | `/rest/sequences/{id}` | Patch any top-level fields |
| Delete | DELETE | `/rest/sequences/{id}` | Hard delete |

**Create body:**
```json
{
  "name": "Q1 Outreach",
  "status": "DRAFT"
}
```

**Response wrapper:** `.data.sequence` (singular) or `.data.sequences` (list)

### Search by name (GraphQL)
```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { sequences(filter: { name: { like: \"%{name}%\" } }) { edges { node { id name status } } } }"
  }'
```

### Fetch with steps (depth=1)
```bash
curl -s -G "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

---

## GraphQL Mutations

Endpoint: `POST https://app.usedalil.ai/graphql`

### Step Management

#### createSequenceStep
```graphql
mutation CreateSequenceStep($input: CreateSequenceStepInput!) {
  createSequenceStep(input: $input) {
    stepsDiff
  }
}
```
Input:
```json
{
  "sequenceId": "uuid",
  "stepType": "SEND_EMAIL",
  "parentStepId": "uuid-of-parent-or-null",
  "nextStepId": "uuid-to-insert-before-or-null",
  "position": { "x": 0, "y": 200 },
  "id": "optional-preset-uuid",
  "isFirstStep": false
}
```
Returns: `{ stepsDiff: JSON }` — a microdiff array describing changes to the steps graph.

#### updateSequenceStep
```graphql
mutation UpdateSequenceStep($input: UpdateSequenceStepInput!) {
  updateSequenceStep(input: $input) {
    id name type settings valid isFirstStep nextStepIds
    position { x y }
  }
}
```
Input:
```json
{
  "sequenceId": "uuid",
  "step": {
    "id": "step-uuid",
    "name": "Send intro email",
    "type": "SEND_EMAIL",
    "isFirstStep": true,
    "valid": true,
    "nextStepIds": ["next-step-uuid"],
    "position": { "x": 0, "y": 0 },
    "settings": {
      "input": {
        "subject": "Hi {{person.name.firstName}}",
        "body": "<p>Hello {{person.name.firstName}},</p>",
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
}
```

#### deleteSequenceStep
```graphql
mutation DeleteSequenceStep($input: DeleteSequenceStepInput!) {
  deleteSequenceStep(input: $input) {
    stepsDiff
  }
}
```
Input: `{ "sequenceId": "uuid", "stepId": "step-uuid" }`

#### duplicateSequenceStep
```graphql
mutation DuplicateSequenceStep($input: DuplicateSequenceStepInput!) {
  duplicateSequenceStep(input: $input) {
    stepsDiff
  }
}
```
Input: `{ "sequenceId": "uuid", "stepId": "step-uuid" }`

### Sequence Management

#### duplicateSequence
```graphql
mutation DuplicateSequence($input: DuplicateSequenceInput!) {
  duplicateSequence(input: $input) {
    success newSequenceId duplicatedSendersCount duplicatedPeopleCount warnings
  }
}
```
Input: `{ "sequenceId": "uuid", "duplicateType": "SEQUENCE_ONLY" }`
`duplicateType`: `SEQUENCE_ONLY` | `ALL_SEQUENCE` | `SEQUENCE_INCOMPLETE_RUNS`

#### createSequenceFromTemplate
```graphql
mutation CreateSequenceFromTemplate($input: CreateSequenceFromTemplateInput!) {
  createSequenceFromTemplate(input: $input) {
    success sequenceId stepsCreated
  }
}
```
Input: `{ "name": "My Sequence", "templateType": "LINKEDIN_OUTREACH" }`

### Sender Management

#### activateSequenceSenders
```graphql
mutation ActivateSequenceSenders($input: ActivateSequenceSendersInput!) {
  activateSequenceSenders(input: $input)
}
```
Input:
```json
{
  "senderActions": [
    { "sequenceSenderId": "uuid", "actionType": "message" }
  ]
}
```

#### removeSender
```graphql
mutation RemoveSender($input: RemoveSenderInput!) {
  removeSender(input: $input) {
    success deletedSenderId totalAffectedPeople reassignedCount stoppedCount warnings
  }
}
```
Input:
```json
{
  "platform": "EMAIL",
  "sequenceSenderId": "uuid",
  "reassign": false,
  "hasBeenPublished": true
}
```

### Run Management

#### retrySequenceSteps
```graphql
mutation RetrySequenceSteps($input: [RetrySequenceStepInput!]!) {
  retrySequenceSteps(input: $input)
}
```
Input: `[{ "sequenceId": "uuid", "sequenceRunId": "uuid", "stepId": "step-uuid" }]`

#### resetSequencePersonRun
```graphql
mutation ResetSequencePersonRun($input: ResetSequencePersonRunInput!) {
  resetSequencePersonRun(input: $input)
}
```
Input: `{ "sequencePersonId": "uuid" }`

---

## GraphQL Queries

### resolvedSequencePeople
```graphql
query ResolvedSequencePeople($filter: SequencePeopleFilterInput!) {
  resolvedSequencePeople(filter: $filter) {
    edges {
      node {
        sequenceId personId status
        person { name { firstName lastName } emails { primaryEmail } avatarUrl }
        steps { id type resolvedInput invalidVariables }
      }
    }
    pageInfo { hasNextPage endCursor }
    totalCount
  }
}
```
Filter: `{ "sequenceId": "uuid", "first": 20, "after": "cursor" }`

### sequenceErrors
```graphql
query SequenceErrors($filter: SequenceErrorsFilterInput!) {
  sequenceErrors(filter: $filter) {
    peopleErrors { sequencePersonId failedStepId failedStepType failedStepError }
    senderErrors { senderId platformType status actionType reason displayValue }
    completedNoMessage { sequencePersonId status reason }
  }
}
```
Filter: `{ "sequenceId": "uuid" }`

---

## Data Model

### Sequence
```
id: UUID
name: String
status: DRAFT | ACTIVE | DEACTIVATED
steps: RAW_JSON                  ← array of SequenceAction objects
settings: RAW_JSON               ← SequenceLimitSettings
senderAssignStatus: FAILED | PENDING | COMPLETED
```

### SequencePerson
```
id: UUID
status: ACTIVE | PAUSED
hasReplied: Boolean
hasClickedLink: Boolean
hasUnsubscribed: Boolean
```
Relations: `→ Person`, `→ Sequence`

### SequenceRun
```
id: UUID
status: NOT_STARTED | RUNNING | COMPLETED | FAILED | ENQUEUED | STOPPED
state: RAW_JSON                  ← SequenceRunState (flow + stepInfos)
```
Relations: `→ SequencePerson`

### SequenceSender
```
id: UUID
platformType: EMAIL | WHATSAPP | LINKEDIN
identifier: String               ← email address, phone, or LinkedIn URL
messageStatus, profileVisitStatus, commentPostStatus, likePostStatus, inviteStatus: SenderStatus
```
Relations: `→ WorkspaceMember`, `→ ConnectedAccount (nullable)`

---

## Key Enums

| Enum | Values |
|---|---|
| `SequenceStatus` | `DRAFT`, `ACTIVE`, `DEACTIVATED` |
| `SequenceRunStatus` | `NOT_STARTED`, `RUNNING`, `COMPLETED`, `FAILED`, `ENQUEUED`, `STOPPED` |
| `SequencePersonStatus` | `ACTIVE`, `PAUSED` |
| `SequencePlatformType` | `EMAIL`, `WHATSAPP`, `LINKEDIN` |
| `StepStatus` | `NOT_STARTED`, `RUNNING`, `SUCCESS`, `STOPPED`, `FAILED`, `PENDING`, `SKIPPED`, `PAUSED`, `VERIFIED`, `DELAYED`, `RESUMED`, `CONDITION_NOT_MET`, `CONDITION_MET` |
| `DayOfWeek` | `1` (Mon) – `7` (Sun) |

---

## Gotchas

1. **Steps are stored as RAW_JSON on the sequence** — `createSequenceStep` / `updateSequenceStep` mutations manage them; don't PATCH `steps` directly via REST.
2. **Every step has `nextStepIds`** — routing is explicit; unconnected steps are dead ends.
3. **`valid: false` blocks activation** — the UI shows "Please complete all action steps before viewing leads" if any sending step has `valid: false`. This means `settings.input` is missing required content. `CREATE_TASK` is an exception — it persistently returns `valid: false` from the API even when fully configured; this is a known API quirk and does not block activation.
4. **`updateSequenceStep` requires the FULL settings object** — sending only `settings.input` will silently reset other fields (e.g. `delay`, `conditionalOptions`, `outputSchema`) to defaults, and the step body will appear empty. Always fetch the current step via `GET /rest/sequences/{id}?depth=1` first and merge your changes into the existing settings before calling update.
5. **SEND_EMAIL: `body` cannot be empty string** — the UI validates this on activation. Always set a non-empty HTML string (e.g. `"<p>Your message here</p>"`). An empty `""` passes the API but fails UI validation.
6. **SEND_LINKEDIN step type is `SEND_LINKEDIN`** — not `SEND_LINKEDIN_MESSAGE`. The `stepType` field is a plain string, not a typed enum; wrong values return `"SequenceActionType '...' unknown"`.
7. **SEND_LINKEDIN `message` cannot be empty** — same as email: an empty `""` passes the API but the UI blocks activation with "Incomplete steps". Always set a non-empty string.
8. **SEND_LINKEDIN `senderId` must be a `SequenceSender` ID, not a workspace member ID** — fetch `GET /rest/sequenceSenders`, filter by `platformType == LINKEDIN` and `workspaceMemberId`, and use the sender's `id`. New senders added via the UI will appear here; re-fetch after the user connects an account.
7. **No versioning** — unlike workflows, sequences have no version layer. Edits apply to the live record; if status is ACTIVE, changes apply immediately.
8. **Senders are separate records** — `SequenceSender` objects are created per workspace member per platform. They must exist before a sending step can run. Check `GET /rest/sequenceSenders` and filter by `workspaceMemberId` to verify a sender exists for the desired platform before building the sequence. Most workspaces only have senders for a few members.
9. **`CREATE_TASK` assignee format is nested object, not a flat ID** — use `"assignee": { "id": "workspace-member-uuid", "name": "Full Name" }` inside `objectRecord`, not `"assigneeId"`.
10. **`stepsDiff` return** — `createSequenceStep` and `deleteSequenceStep` return a microdiff array, not the step itself. To get the new step's UUID, fetch the sequence at `depth=1` after creation.
11. **Never list all sequences** — use search by name via GraphQL. A full list can be very large.
12. **`depth=1` includes steps** — use `depth=0` (default) for status checks only.
