---
name: workflow-actions
description: Add and configure action steps on a Dalil AI workflow version — record CRUD (create/update/delete/find/bulk), communication (email/WhatsApp/LinkedIn), HTTP requests, code execution, control flow (condition/iterator/filter/delay), calculations (formula/aggregate/random/adjust-time), sequences, forms, and visual nodes (comment). Use after setting up a trigger with the `workflow-triggers` skill.
---

# Dalil AI: Workflow Actions Skill

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}`
- **GraphQL endpoint:** `POST https://app.usedalil.ai/graphql`
- **How steps are added:** `createWorkflowVersionStep` GraphQL mutation
- **How steps are updated:** `updateWorkflowVersionStep` GraphQL mutation
- **How steps are removed:** `deleteWorkflowVersionStep` GraphQL mutation
- **How steps are connected:** `createWorkflowVersionEdge` GraphQL mutation

**Critical rules:**
- All step mutations require the version to be in `DRAFT` status
- Every step has a shared base shape — `id`, `name`, `type`, `valid`, `settings`, `nextStepIds`, `position`
- `settings` always contains `input` (action-specific fields), `outputSchema` (computed), and `errorHandlingOptions`
- Variables are referenced inside `input` values as `{{stepId.path}}` or `{{trigger.path}}`
- `nextStepIds` controls flow — one step points to the next via its UUID; for branching nodes (CONDITION) this is handled by `trueNextStepIds` / `falseNextStepIds` inside `settings.input`
- `valid` should be set to `true` once required fields are filled; the system may also compute this

---

## Step Base Shape (all action types share this)

```json
{
  "id": "uuid-of-this-step",
  "name": "Descriptive step name",
  "type": "CREATE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": ["uuid-of-next-step"],
  "settings": {
    "input": { },
    "outputSchema": { },
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

`position.y` increases by ~220 per step downward. `position.x` shifts horizontally for branches.

---

## How to Add a Step

### 1. Create the step (generates an ID, sets default settings)

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateWorkflowVersionStep($input: CreateWorkflowVersionStepInput!) { createWorkflowVersionStep(input: $input) { triggerDiff stepsDiff } }",
    "variables": {
      "input": {
        "workflowVersionId": "version-uuid",
        "stepType": "CREATE_RECORD",
        "parentStepId": "trigger",
        "position": { "x": 0, "y": 220 }
      }
    }
  }'
```

`parentStepId` is `"trigger"` for the first step after the trigger, or the UUID of the preceding step.

### 2. Update the step with your configuration

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateWorkflowVersionStep($input: UpdateWorkflowVersionStepInput!) { updateWorkflowVersionStep(input: $input) { id name type settings valid nextStepIds } }",
    "variables": {
      "input": {
        "workflowVersionId": "version-uuid",
        "step": { ...full step object from examples below... }
      }
    }
  }'
```

### 3. Connect steps (if not handled automatically by create)

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateWorkflowVersionEdge($input: CreateWorkflowVersionEdgeInput!) { createWorkflowVersionEdge(input: $input) { triggerDiff stepsDiff } }",
    "variables": {
      "input": {
        "workflowVersionId": "version-uuid",
        "source": "source-step-uuid",
        "target": "target-step-uuid"
      }
    }
  }'
```

---

## Record CRUD Actions

### CREATE_RECORD — Create a new record

```json
{
  "id": "step-uuid",
  "name": "Create person",
  "type": "CREATE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "person",
      "objectRecord": {
        "name": {
          "firstName": "{{trigger.properties.after.name.firstName}}",
          "lastName": "{{trigger.properties.after.name.lastName}}"
        },
        "emails": {
          "primaryEmail": "{{trigger.properties.after.emails.primaryEmail}}"
        }
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

**Set all fields — including relations, status, assignee, and due date — in a single CREATE_RECORD step.** Do not add a follow-up UPDATE_RECORD step just to set more fields on the same newly created record.

```json
{
  "objectName": "task",
  "objectRecord": {
    "title": "Follow up with {{trigger.properties.after.name.firstName}}",
    "status": "TODO",
    "dueAt": "2026-06-01T09:00:00.000Z",
    "assigneeId": "workspace-member-uuid",
    "taskTargets": [{ "personId": "{{trigger.properties.after.id}}" }]
  }
}
```

Relation fields are passed as inline arrays — no separate step needed. This is what the UI's "Relation" field on a CREATE_RECORD step does under the hood.

**Known inline relation fields:**

| Object | Relation field | Shape |
|---|---|---|
| `task` | `taskTargets` | `[{ "personId": "..." }]` / `[{ "companyId": "..." }]` / `[{ "opportunityId": "..." }]` |
| `note` | `noteTargets` | `[{ "personId": "..." }]` / `[{ "companyId": "..." }]` / `[{ "opportunityId": "..." }]` |

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `objectName` | string | Yes | Singular object name: `"person"`, `"company"`, `"opportunity"`, `"task"`, `"note"`, etc. |
| `objectRecord` | object | Yes | Key-value pairs of fields to set. Scalar values or `{{...}}` expressions. Relation fields (e.g. `taskTargets`, `noteTargets`) can be passed as inline arrays — no separate step needed. |

Output: the full created record — all its fields available as `{{stepId.*}}`.

---

### UPDATE_RECORD — Update an existing record

```json
{
  "id": "step-uuid",
  "name": "Update opportunity stage",
  "type": "UPDATE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 440 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecordId": "{{trigger.id}}",
      "fieldsToUpdate": ["stage", "closeDate"],
      "objectRecord": {
        "stage": "WON",
        "closeDate": "{{step1.value}}"
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

**Before setting fields in UPDATE_RECORD (or CREATE_RECORD), always inspect the object's real field names first:**

```bash
# Fetch a live record to see the actual field keys
curl -s -G "https://app.usedalil.ai/rest/{objectNamePlural}" \
  --data-urlencode "limit=1" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}" | jq '.data.{objectNamePlural}[0] | keys'
```

This avoids assuming field names that don't exist or using the wrong key (e.g. confusing `assigneeId` with `personId`, or `status` with `stage`). Field names on the actual record are the authoritative source — not the UI label.

**Common field name traps:**

| What you might assume | Actual field key | Notes |
|---|---|---|
| task "Assignee" | `assigneeId` | A UUID — verify in the variable picker whether it expects a workspace member or person ID for your object |
| task "related person" | via `taskTargets` (separate object) | `assigneeId` and `taskTargets` are independent |
| opportunity "stage" | `stage` | SELECT value in UPPER_SNAKE_CASE |
| company "website" | `domainName.primaryLinkUrl` | Composite LINKS field, not a flat string |

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `objectName` | string | Yes | Object type |
| `objectRecordId` | string | Yes | UUID of the record to update — can be a `{{...}}` expression |
| `fieldsToUpdate` | string[] | Yes | List of field names being changed — must match keys in `objectRecord` |
| `objectRecord` | object | Yes | New values for each field in `fieldsToUpdate` |

Output: the full updated record.

---

### DELETE_RECORD — Delete a record

```json
{
  "id": "step-uuid",
  "name": "Delete old task",
  "type": "DELETE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 440 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "task",
      "objectRecordId": "{{trigger.id}}"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

Output: none (empty `{}`).

---

### FIND_RECORDS — Search / filter records

```json
{
  "id": "step-uuid",
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
            "value": "WON",
            "stepFilterGroupId": "group-1",
            "fieldMetadataId": "field-uuid"
          }
        ],
        "recordFilterGroups": [
          {
            "id": "group-1",
            "logicalOperator": "AND"
          }
        ]
      },
      "orderBy": { "createdAt": "DescNullsLast" },
      "limit": 20
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `objectName` | string | Yes | Object type to search |
| `filter` | object | No | `recordFilters` + `recordFilterGroups` (see Filter System below) |
| `orderBy` | object | No | e.g. `{ "createdAt": "DescNullsLast" }` |
| `limit` | number | No | Max records to return |

Output: `{ first: {...}, all: [...], totalCount: N }` — use `{{stepId.first.*}}` for the top result, `{{stepId.all}}` for the array.

---

### BULK_UPDATE_RECORDS — Update multiple records matching a filter

```json
{
  "id": "step-uuid",
  "name": "Mark all tasks done",
  "type": "BULK_UPDATE_RECORDS",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "task",
      "fieldsToUpdate": ["status"],
      "objectRecord": { "status": "DONE" },
      "filter": {
        "recordFilters": [...],
        "recordFilterGroups": [...]
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

---

### BULK_DELETE_RECORDS — Delete all records matching a filter

```json
{
  "id": "step-uuid",
  "name": "Delete old notes",
  "type": "BULK_DELETE_RECORDS",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "note",
      "filter": {
        "recordFilters": [...],
        "recordFilterGroups": [...]
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

---

## Communication Actions

### SEND_EMAIL

```json
{
  "id": "step-uuid",
  "name": "Send welcome email",
  "type": "SEND_EMAIL",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "connectedAccountId": "connected-account-uuid",
      "email": "{{trigger.email.primaryEmail}}",
      "subject": "Welcome, {{trigger.name.firstName}}!",
      "body": "<p>Hi {{trigger.name.firstName}},</p><p>Welcome to our platform.</p>"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `connectedAccountId` | UUID | Yes | The connected email account to send from |
| `email` | string | Yes | Recipient email address — can be `{{...}}` expression |
| `subject` | string | No | Email subject line — supports `{{...}}` variables |
| `body` | string | No | HTML email body — supports `{{...}}` variables inline |

Output: none.

---

### SEND_WHATSAPP

```json
{
  "id": "step-uuid",
  "name": "Send WhatsApp message",
  "type": "SEND_WHATSAPP",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "connectedAccountId": "connected-account-uuid",
      "personId": "{{trigger.id}}",
      "message": "Hi {{trigger.name.firstName}}, just following up!"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:** `connectedAccountId` (UUID), `personId` (UUID or expression), `message` (string with variable support).

---

### SEND_LINKEDIN

```json
{
  "id": "step-uuid",
  "name": "Send LinkedIn message",
  "type": "SEND_LINKEDIN",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "connectedAccountId": "connected-account-uuid",
      "personId": "{{trigger.id}}",
      "message": "Hi {{trigger.name.firstName}}, great connecting with you!"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

Same shape as SEND_WHATSAPP — `connectedAccountId`, `personId`, `message`.

---

## HTTP & Code Actions

### HTTP_REQUEST — Call an external API

```json
{
  "id": "step-uuid",
  "name": "Notify Slack",
  "type": "HTTP_REQUEST",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "url": "https://hooks.slack.com/services/xxx/yyy/zzz",
      "method": "POST",
      "headers": {
        "Content-Type": "application/json"
      },
      "body": {
        "text": "New person created: {{trigger.name.firstName}} {{trigger.name.lastName}}"
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

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `url` | string | Yes | Full URL — supports `{{...}}` variables |
| `method` | string | Yes | `"GET"`, `"POST"`, `"PUT"`, `"PATCH"`, or `"DELETE"` |
| `headers` | object | No | Key-value string pairs |
| `body` | object \| string | No | JSON body (object) or raw string body — values support `{{...}}` variables |

Output: the full HTTP response body as JSON.

---

### CODE — Run a serverless function

```json
{
  "id": "step-uuid",
  "name": "Transform data",
  "type": "CODE",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "serverlessFunctionId": "function-uuid",
      "serverlessFunctionVersion": "latest",
      "serverlessFunctionInput": {
        "personId": "{{trigger.id}}",
        "companyName": "{{trigger.company.name}}"
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

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `serverlessFunctionId` | UUID | Yes | ID of the serverless function to call |
| `serverlessFunctionVersion` | string | Yes | Version string — use `"latest"` for the current version |
| `serverlessFunctionInput` | object | No | Key-value map passed as arguments to the function — values support `{{...}}` variables |

Output: whatever the function returns — fields are available as `{{stepId.*}}`.

---

## Control Flow Actions

### CONDITION — If / else branch

Evaluates filters and routes execution to either a `true` path or `false` path.

```json
{
  "id": "step-uuid",
  "name": "Check if VIP",
  "type": "CONDITION",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "stepFilterGroups": [
        {
          "id": "group-1",
          "logicalOperator": "AND"
        }
      ],
      "stepFilters": [
        {
          "id": "filter-1",
          "type": "TEXT",
          "stepOutputKey": "trigger.jobTitle",
          "operand": "CONTAINS",
          "value": "VP",
          "stepFilterGroupId": "group-1"
        }
      ],
      "trueNextStepIds": ["uuid-of-true-branch-step"],
      "falseNextStepIds": ["uuid-of-false-branch-step"]
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:**

| Field | Type | Description |
|---|---|---|
| `stepFilterGroups` | array | Groups of filters with logical operator (`AND` / `OR`) |
| `stepFilters` | array | Individual filter conditions — each references a `stepFilterGroupId` |
| `trueNextStepIds` | string[] | Steps to run when condition matches |
| `falseNextStepIds` | string[] | Steps to run when condition does not match |

Output: none (routing only).

---

### ITERATOR — Loop over an array

Runs a set of steps once for each item in an array.

```json
{
  "id": "step-uuid",
  "name": "Loop over people",
  "type": "ITERATOR",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": ["uuid-of-step-after-loop"],
  "settings": {
    "input": {
      "items": "{{findPeopleStep.all}}",
      "initialLoopStepIds": ["uuid-of-first-step-inside-loop"]
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `items` | array \| string | Yes | The array to iterate — can be a literal array or a `{{...}}` variable pointing to an array |
| `initialLoopStepIds` | string[] | Yes | The first step(s) inside the loop body |

Inside loop steps, use `{{currentItem.*}}` to access the current array element.

Output per iteration: `{ currentItem, currentItemIndex, hasProcessedAllItems }`.

---

### FILTER — Filter an array

Filters items in an array based on conditions and passes the matching subset downstream.

```json
{
  "id": "step-uuid",
  "name": "Keep only enterprise companies",
  "type": "FILTER",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "stepFilterGroups": [
        {
          "id": "group-1",
          "logicalOperator": "AND"
        }
      ],
      "stepFilters": [
        {
          "id": "filter-1",
          "type": "TEXT",
          "stepOutputKey": "currentItem.annualRecurringRevenue.amountMicros",
          "operand": "GREATER_THAN_OR_EQUAL",
          "value": "100000000000",
          "stepFilterGroupId": "group-1"
        }
      ]
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

Output: filtered array — available as `{{stepId.all}}`.

---

### DELAY — Pause execution

Pauses the workflow for a set duration or until a scheduled date.

**Duration-based delay:**
```json
{
  "id": "step-uuid",
  "name": "Wait 3 days",
  "type": "DELAY",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "delayType": "DURATION",
      "duration": {
        "days": 3,
        "hours": 0,
        "minutes": 0,
        "seconds": 0
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

**Scheduled date delay:**
```json
{
  "id": "step-uuid",
  "name": "Wait until follow-up date",
  "type": "DELAY",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "delayType": "SCHEDULED_DATE",
      "scheduledDateTime": "{{trigger.followUpDate}}"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

Output: none.

---

## Calculation Actions

### FORMULA — Evaluate a math expression

```json
{
  "id": "step-uuid",
  "name": "Calculate discount",
  "type": "FORMULA",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "formula": "{{trigger.price}} * 0.9"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

Output: `{ value: <number> }` — reference as `{{stepId.value}}`.

---

### AGGREGATE_VALUES — Sum / average / min / max

```json
{
  "id": "step-uuid",
  "name": "Sum deal values",
  "type": "AGGREGATE_VALUES",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "values": "{{findDealsStep.all}}",
      "operation": "SUM"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`operation` values:** `SUM`, `AVERAGE`, `MINIMUM`, `MAXIMUM`

Output: `{ value: <number> }` — reference as `{{stepId.value}}`.

---

### RANDOM_NUMBER — Generate a random integer

```json
{
  "id": "step-uuid",
  "name": "Random score",
  "type": "RANDOM_NUMBER",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "minimum": 1,
      "maximum": 100
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

Output: `{ value: <number> }` — reference as `{{stepId.value}}`.

---

### ADJUST_TIME — Add or subtract time from a date

```json
{
  "id": "step-uuid",
  "name": "Add 7 days to close date",
  "type": "ADJUST_TIME",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "dateTime": "{{trigger.closeDate}}",
      "today": false,
      "offset": 7,
      "unit": "DAYS"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:**

| Field | Type | Description |
|---|---|---|
| `dateTime` | string | ISO datetime string or `{{...}}` expression |
| `today` | boolean | If `true`, ignores `dateTime` and uses the current date |
| `offset` | number | Number of units to add (positive) or subtract (negative) |
| `unit` | string | `SECONDS`, `MINUTES`, `HOURS`, `DAYS`, `WEEKS`, `MONTHS`, or `YEARS` |

Output: `{ value: "<ISO datetime string>" }` — reference as `{{stepId.value}}`.

---

## Sequence Actions

### CREATE_SEQUENCE_PERSON — Enroll a person in a sequence

```json
{
  "id": "step-uuid",
  "name": "Add to onboarding sequence",
  "type": "CREATE_SEQUENCE_PERSON",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "personId": "{{trigger.id}}",
      "sequenceId": "sequence-uuid"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

### PAUSE_SEQUENCE_PERSON — Pause a person's sequence

Same shape as CREATE_SEQUENCE_PERSON — `personId` and `sequenceId`.

### DELETE_SEQUENCE_PERSON — Remove a person from a sequence

Same shape — `sequenceId` and `personId`.

---

## Form Action

### FORM — Collect user input mid-workflow

Pauses execution and presents a form to a user. Resumes when submitted.

```json
{
  "id": "step-uuid",
  "name": "Approval form",
  "type": "FORM",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": [
      {
        "id": "field-1",
        "name": "approvalDecision",
        "label": "Approve this deal?",
        "type": "TEXT",
        "placeholder": "yes / no"
      },
      {
        "id": "field-2",
        "name": "discount",
        "label": "Discount %",
        "type": "NUMBER",
        "value": 10
      }
    ],
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**Form field types:** `TEXT`, `NUMBER`, `DATE`, `RECORD`

**Form field shape:**

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique field identifier |
| `name` | string | Yes | Variable name used to reference the field's output |
| `label` | string | Yes | Display label shown in the form |
| `type` | string | Yes | `TEXT`, `NUMBER`, `DATE`, or `RECORD` |
| `value` | any | No | Default value pre-filled in the form |
| `placeholder` | string | No | Hint text shown in the empty input |

Output: each field's submitted value is available as `{{stepId.fieldName}}`.

---

## Visual / Special Nodes

### COMMENT — Floating annotation (non-executing)

Adds a documentation node to the canvas — does not execute, has no output.

```json
{
  "id": "step-uuid",
  "name": "Comment",
  "type": "COMMENT",
  "valid": true,
  "position": { "x": -420, "y": 0 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "body": "<p><strong>What this step does</strong></p><p>Explain the logic here.</p>"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

`body` is an HTML string. Position it to the left of the step it annotates (negative x offset). COMMENT nodes are floating — they do not connect into the flow.

---

## Pipeline Actions

Pipeline actions operate on records inside a **Pipeline** — Dalil's kanban/stage-based view over an existing object (e.g. an opportunity pipeline on top of the `opportunity` object). They mirror the standard Record CRUD actions but scope operations to a specific pipeline's field set and stages.

**What makes pipeline actions different from record CRUD:**
- `objectName` refers to the pipeline's `nameSingular` (e.g. `"opportunity"`), fetched via `findManyPipelinesByObjectMetadataId` on the metadata `/graphql` endpoint
- Pipeline fields (stage, position, status) are tracked separately — use `isPipelineField: true` in step filters to target them
- The `pipelineId` is not part of the action `input` — it is resolved at runtime from the object's pipeline configuration

**How to get the pipeline's `nameSingular`:**
```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query FindPipelines($objectMetadataId: String!) { findManyPipelinesByObjectMetadataId(objectMetadataId: $objectMetadataId) { id nameSingular namePlural labelSingular } }",
    "variables": { "objectMetadataId": "opportunity-object-metadata-uuid" }
  }'
```

---

### ADD_PIPELINE_RECORD — Add a record to a pipeline

Adds an existing record into a pipeline (creates a pipeline record entry). Uses the same input shape as `CREATE_RECORD`.

```json
{
  "id": "step-uuid",
  "name": "Add to pipeline",
  "type": "ADD_PIPELINE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecord": {
        "name": "{{trigger.name}}",
        "stageId": "stage-uuid"
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

**Output:** Full created record — same as `CREATE_RECORD`.

---

### UPDATE_PIPELINE_RECORD — Update a record inside a pipeline

Updates fields on a pipeline record. Requires `objectRecordId` and `fieldsToUpdate`. Same input shape as `UPDATE_RECORD`.

```json
{
  "id": "step-uuid",
  "name": "Update pipeline stage",
  "type": "UPDATE_PIPELINE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 440 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecordId": "{{trigger.id}}",
      "objectRecord": {
        "stageId": "closed-won-stage-uuid"
      },
      "fieldsToUpdate": ["stageId"]
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**Output:** Full updated record — same as `UPDATE_RECORD`.

---

### DELETE_PIPELINE_RECORD — Remove a record from a pipeline

Removes a record from the pipeline. Same input shape as `DELETE_RECORD`.

```json
{
  "id": "step-uuid",
  "name": "Remove from pipeline",
  "type": "DELETE_PIPELINE_RECORD",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecordId": "{{trigger.id}}"
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**Output:** Full deleted record — same as `DELETE_RECORD`.

---

### FIND_PIPELINE_RECORDS — Search records in a pipeline

Finds records in a pipeline matching a filter. Same input shape as `FIND_RECORDS`.

```json
{
  "id": "step-uuid",
  "name": "Find open opportunities",
  "type": "FIND_PIPELINE_RECORDS",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "filter": {
        "recordFilterGroups": [
          { "id": "group-uuid", "logicalOperator": "AND", "positionInStepFilterGroup": 0 }
        ],
        "recordFilters": [
          {
            "id": "filter-uuid",
            "fieldMetadataId": "stage-field-metadata-uuid",
            "operand": "isNotEmpty",
            "value": "",
            "stepFilterGroupId": "group-uuid",
            "isPipelineField": true
          }
        ]
      },
      "limit": 50
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**Output:** `{ first, all, totalCount }` — same as `FIND_RECORDS`. Access via `{{stepId.first.*}}`, `{{stepId.totalCount}}`, or pass `{{stepId.all}}` to an ITERATOR.

---

### BULK_UPDATE_PIPELINE_RECORDS — Update all matching pipeline records

Updates all pipeline records matching a filter. Same input shape as `BULK_UPDATE_RECORDS`.

```json
{
  "id": "step-uuid",
  "name": "Close all stale deals",
  "type": "BULK_UPDATE_PIPELINE_RECORDS",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "objectRecord": {
        "stageId": "closed-lost-stage-uuid"
      },
      "fieldsToUpdate": ["stageId"],
      "filter": {
        "recordFilterGroups": [
          { "id": "group-uuid", "logicalOperator": "AND", "positionInStepFilterGroup": 0 }
        ],
        "recordFilters": [
          {
            "id": "filter-uuid",
            "fieldMetadataId": "close-date-field-uuid",
            "operand": "isBefore",
            "value": "{{trigger.date}}",
            "filterValueMode": "VARIABLE",
            "stepFilterGroupId": "group-uuid",
            "isPipelineField": false
          }
        ]
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

**Output:** Bulk record array node — same as `BULK_UPDATE_RECORDS`.

---

### BULK_DELETE_PIPELINE_RECORDS — Delete all matching pipeline records

Removes all pipeline records matching a filter. Same input shape as `BULK_DELETE_RECORDS`.

```json
{
  "id": "step-uuid",
  "name": "Remove stale pipeline entries",
  "type": "BULK_DELETE_PIPELINE_RECORDS",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "objectName": "opportunity",
      "filter": {
        "recordFilterGroups": [
          { "id": "group-uuid", "logicalOperator": "AND", "positionInStepFilterGroup": 0 }
        ],
        "recordFilters": [
          {
            "id": "filter-uuid",
            "fieldMetadataId": "stage-field-metadata-uuid",
            "operand": "eq",
            "value": "archived-stage-uuid",
            "stepFilterGroupId": "group-uuid",
            "isPipelineField": true
          }
        ]
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

**Output:** Bulk record array node — same as `BULK_DELETE_RECORDS`.

---

## AI / Enrichment Actions

### ENRICH — AI-powered enrichment prompt

Runs an AI prompt against data in the workflow. The `body` field is a freeform prompt string — it can contain `{{...}}` variable expressions to inject record data.

```json
{
  "id": "step-uuid",
  "name": "Enrich person bio",
  "type": "ENRICH",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "input": {
      "body": "Summarize the professional background of {{trigger.name.firstName}} {{trigger.name.lastName}} who works at {{trigger.company.name}} as {{trigger.jobTitle}}."
    },
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**`input` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `body` | string | Yes | The AI prompt text; supports `{{...}}` variable expressions |

**Output:** Empty — ENRICH has no structured output schema. Results are applied externally.

---

### EMPTY — No-op placeholder

A placeholder step that performs no action. Used to represent an unfinished branch, a default fallback leg, or as a visual connector while building the workflow.

```json
{
  "id": "step-uuid",
  "name": "Coming soon",
  "type": "EMPTY",
  "valid": true,
  "position": { "x": 0, "y": 220 },
  "nextStepIds": [],
  "settings": {
    "outputSchema": {},
    "errorHandlingOptions": {
      "retryOnFailure": { "value": false },
      "continueOnFailure": { "value": false }
    }
  }
}
```

**Output:** Empty — no output variables.

---

## Filter System (used in CONDITION, FILTER, FIND_RECORDS, BULK actions)

Filters are expressed as a list of `stepFilters` grouped by `stepFilterGroups`.

**Filter group:**
```json
{
  "id": "group-1",
  "logicalOperator": "AND"
}
```

**Filter condition:**
```json
{
  "id": "filter-1",
  "type": "TEXT",
  "stepOutputKey": "trigger.jobTitle",
  "operand": "CONTAINS",
  "value": "VP",
  "stepFilterGroupId": "group-1",
  "fieldMetadataId": "optional-field-uuid"
}
```

**Key filter fields:**

| Field | Description |
|---|---|
| `stepOutputKey` | Dot-path to the value to test: `"trigger.fieldName"`, `"stepId.fieldName"`, `"currentItem.fieldName"` |
| `operand` | Comparison operator — see table below |
| `value` | The value to compare against (always a string, even for numbers) |
| `filterValueMode` | `"fixed"` (literal) or `"expression"` (use `value` as a `{{...}}` variable path) |

**Operand values:**

| Operand | Meaning |
|---|---|
| `IS` / `is` | Equals |
| `IS_NOT` / `isNot` | Not equals |
| `CONTAINS` / `contains` | String contains substring |
| `DOES_NOT_CONTAIN` / `doesNotContain` | String does not contain |
| `GREATER_THAN_OR_EQUAL` / `greaterThan` | ≥ (numbers/dates) |
| `LESS_THAN_OR_EQUAL` / `lessThan` | ≤ (numbers/dates) |
| `IS_BEFORE` / `isBefore` | Date before |
| `IS_AFTER` / `isAfter` | Date after |
| `IS_EMPTY` / `isEmpty` | Field is null/empty |
| `IS_NOT_EMPTY` / `isNotEmpty` | Field is not null/empty |
| `IS_NOT_NULL` / `isNotNull` | Field is not null |
| `IS_TODAY` / `isToday` | Date equals today |

---

## Gotchas

1. **All step mutations require a DRAFT version** — Creating or updating a step on an ACTIVE version will fail. Call `createDraftFromWorkflowVersion` first.
2. **`updateWorkflowVersionStep` replaces the entire step** — Send the full step object including all existing fields. Partial updates are not supported; missing fields will be cleared.
3. **`fieldsToUpdate` is required for UPDATE_RECORD and BULK_UPDATE_RECORDS** — The field names in `fieldsToUpdate` must exactly match the keys in `objectRecord`. If a key is in `objectRecord` but not in `fieldsToUpdate`, it will be ignored.
4. **`parentStepId` in `createWorkflowVersionStep` must be `"trigger"` for the first step** — The literal string `"trigger"` (not a UUID) is the reserved ID for the trigger node.
5. **CONDITION does not use `nextStepIds`** — Routing is controlled by `trueNextStepIds` and `falseNextStepIds` inside `settings.input`, not the top-level `nextStepIds` field.
6. **ITERATOR's `items` can be a string expression** — Pass `"{{stepId.all}}"` as a string (not an array) to reference a previous step's output array.
7. **Inside an ITERATOR loop, use `{{currentItem.*}}`** — The variable `currentItem` is only valid within steps that are `initialLoopStepIds` of the iterator. It does not exist outside the loop.
8. **COMMENT nodes should not be connected to the flow** — Set `nextStepIds: []` and position them off to the side (negative x). Do not create an edge from a COMMENT node.
9. **`operand` values have two forms** — Both `"CONTAINS"` (uppercase) and `"contains"` (camelCase) are valid. Use consistent casing within a workflow to avoid confusion.
10. **DELAY with `SCHEDULED_DATE` expects an ISO datetime** — The `scheduledDateTime` value must be a full ISO 8601 string (e.g. `"2026-06-01T09:00:00.000Z"`). Use ADJUST_TIME to compute dynamic dates before passing them here.
11. **FORM pauses execution until submitted** — The workflow run stays in `RUNNING` status until a user submits the form via the UI or the `submitFormStep` GraphQL mutation. Do not use FORM in fully automated flows.
12. **`outputSchema` starts empty** — After creating a step, call `computeStepOutputSchema` (GraphQL) with the step object and version ID to populate it. Without this, downstream steps cannot reference this step's output in the variable picker.
13. **Inspect actual field keys before writing any CREATE_RECORD or UPDATE_RECORD step** — Field names in `objectRecord` and `fieldsToUpdate` must match the real API keys, not the UI labels. Fetch a live record (`GET /rest/{objectNamePlural}?limit=1`) and call `| jq 'keys'` on it before assuming what fields exist. Common trap: `domainName.primaryLinkUrl` is the actual key for a company website, not `website`.
14. **`createWorkflowVersionStep` default `objectName` is `"workflow"`, not the object you want** — The step is created with a placeholder `objectName: "workflow"` in `settings.input`. Always follow up with `updateWorkflowVersionStep` to set the correct `objectName` and `objectRecord`. Skipping this leaves the step pointing at the wrong object type.
15. **DATABASE_EVENT trigger variables are nested under `properties.after.*`, not at the root** — When referencing trigger output in a DATABASE_EVENT workflow, all record fields are accessed as `{{trigger.properties.after.fieldName}}` (e.g. `{{trigger.properties.after.id}}`, `{{trigger.properties.after.name.firstName}}`). This matches what the variable picker shows under "New [object] created/updated". Using `{{trigger.id}}` or `{{trigger.name}}` directly will not resolve — those paths don't exist on the DATABASE_EVENT output shape.
16. **Set relation fields inline in CREATE_RECORD — do not create a separate step** — Relation fields like `taskTargets` (for tasks) and `noteTargets` (for notes) can be passed as inline arrays inside `objectRecord`: `"taskTargets": [{ "personId": "..." }]` or `"noteTargets": [{ "personId": "..." }]`. Both support `personId`, `companyId`, and `opportunityId`. Creating a separate CREATE_RECORD step for the relation object is unnecessary and adds complexity.
17. **FILTER is a visible canvas node that gates the flow** — FILTER sits directly in the step chain (trigger → FILTER → next step) and stops execution if the condition doesn't match. It is NOT only for arrays. Use it when you want "only proceed if X is true." The `stepOutputKey` in a FILTER step uses `{{trigger.properties.after.fieldName}}` syntax with double curly braces (e.g. `"{{trigger.properties.after.name.firstName}}"`), NOT a plain dot-path string.
18. **Set all record fields in CREATE_RECORD — do not follow up with UPDATE_RECORD** — Every writable field (`status`, `assigneeId`, `dueAt`, relation fields, etc.) can be included directly in `objectRecord` on the CREATE_RECORD step. Adding a separate UPDATE_RECORD step immediately after just to set more fields on the same record is unnecessary. Only use UPDATE_RECORD when modifying a record that already exists independently.
19. **When adding a new step to an existing workflow, do NOT touch other steps** — Adding a FILTER, CONDITION, or any other step only requires creating the new step and fixing the trigger/edge connections. Never re-issue `updateWorkflowVersionStep` on existing steps unless they specifically need to change. Rewriting an existing step "for context" or "to reconnect it" will overwrite its `objectRecord` and silently drop fields the user already configured (e.g. `status`, `assigneeId`, `taskTargets`). Read the existing step first, leave it untouched, and only update connections.
20. **Before activating, verify no ghost placeholder steps remain** — Every `createWorkflowVersionStep` call creates a step with `objectName: "workflow"` and `valid: false` by default. If you call `createWorkflowVersionStep` and then update a different step ID (e.g. due to a copy/paste error), the original placeholder is silently left in the version. It will fail at runtime with `"Object cannot be created by workflow"`. Before activating, always fetch the version and confirm all steps have `valid: true` and no step has `objectName: "workflow"`.
