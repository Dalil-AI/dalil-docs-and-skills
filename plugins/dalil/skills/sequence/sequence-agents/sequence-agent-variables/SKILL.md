---
name: sequence-agent-variables
description: Sub-agent for resolving valid {{...}} variable paths for each step in a Dalil AI sequence, given the enrolled person context and prior step outputs. Called by sequence-agent orchestrator; never invoke directly.
---

# Dalil AI: Sequence Variables Sub-Agent

## Purpose

Produces a `variablesMap` — a lookup of valid `{{...}}` expression paths available at each step. This helps the actions-agent insert correct variable expressions into step settings (message bodies, record fields, conditions) without guessing.

Runs in Phase B (parallel with trigger-agent). Do not invoke directly — spawned by `sequence-agent`.

---

## Inputs (from orchestrator)

```json
{
  "stepsPlanned": [
    { "order": 1, "type": "START", "description": "Entry point" },
    { "order": 2, "type": "SEND_EMAIL", "description": "Intro email" },
    { "order": 3, "type": "DELAY", "description": "Wait 3 days" },
    { "order": 4, "type": "REPLIED_TO_EMAIL", "description": "Did they reply?" },
    { "order": 5, "type": "CREATE_TASK", "description": "Create follow-up task" }
  ]
}
```

No API calls are needed. All variable resolution is deterministic based on step types.

---

## Resolution Rules

### Always-available variables (all steps)

Person fields (from enrolled person record):
```
{{person.id}}
{{person.name.firstName}}
{{person.name.lastName}}
{{person.emails.primaryEmail}}
{{person.phones.primaryPhoneNumber}}
{{person.linkedinLink.url}}
{{person.jobTitle}}
{{person.city}}
{{person.company.name}}
{{person.company.domainName.primaryLinkUrl}}
{{person.avatarUrl}}
```

SequencePerson fields:
```
{{sequencePerson.id}}
{{sequencePerson.status}}
{{sequencePerson.hasReplied}}
{{sequencePerson.hasClickedLink}}
{{sequencePerson.hasUnsubscribed}}
```

### Step output variables (only from steps that produce records)

Steps that produce output and their available paths:

| Step type | Output paths |
|---|---|
| `CREATE_RECORD` | `{{stepId.id}}` + any field returned in the created record |
| `CREATE_TASK` | `{{stepId.id}}`, `{{stepId.title}}`, `{{stepId.status}}` |
| `CREATE_NOTE` | `{{stepId.id}}`, `{{stepId.title}}` |

Steps that produce NO output (no `{{stepId.*}}` variables):
- START, DELAY, EMPTY, COMMENT
- All SEND_* steps (SEND_EMAIL, SEND_LINKEDIN, SEND_WHATSAPP, SEND_LINKEDIN_CONNECTION)
- VISIT_LINKEDIN_PROFILE, LIKE_LINKEDIN_POST, COMMENT_LINKEDIN_POST
- CUSTOM_CONDITION, HAS_* checks
- All event detection steps (OPENED_EMAIL, REPLIED_TO_EMAIL, etc.)
- TRIGGER_WORKFLOW, CREATE_SEQUENCE_PERSON

### Availability by position

A step's output variables are available only to steps that come AFTER it in the flow. Use the `stepsPlanned` order to determine what's available at each step.

---

## Return JSON

Produce a `variablesMap` keyed by step order. The orchestrator will match these to step UUIDs after the actions-agent assigns them.

```json
{
  "variablesMap": {
    "step_1": {
      "type": "START",
      "availableVariables": [
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
        "{{sequencePerson.hasReplied}}",
        "{{sequencePerson.hasClickedLink}}",
        "{{sequencePerson.hasUnsubscribed}}"
      ],
      "producesOutput": false
    },
    "step_2": {
      "type": "SEND_EMAIL",
      "availableVariables": [
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
      ],
      "producesOutput": false
    },
    "step_5": {
      "type": "CREATE_TASK",
      "availableVariables": [
        "{{person.id}}",
        "{{person.name.firstName}}",
        "{{person.name.lastName}}",
        "{{person.emails.primaryEmail}}",
        "{{sequencePerson.id}}"
      ],
      "producesOutput": true,
      "outputPaths": ["{{step_5_uuid.id}}", "{{step_5_uuid.title}}"]
    }
  },
  "status": "success"
}
```

Note: Use `"step_N"` keys (by order) since step UUIDs are not known at Phase B time. The orchestrator will substitute real UUIDs after the actions-agent runs.

---

## Recommendations to Include

Along with the variables map, include recommendations for steps that may fail:

```json
{
  "recommendations": [
    {
      "stepOrder": 2,
      "stepType": "SEND_EMAIL",
      "recommendation": "Add a HAS_EMAIL_ADDRESS check before this step — {{person.emails.primaryEmail}} may be null for some enrolled people"
    },
    {
      "stepOrder": 6,
      "stepType": "SEND_LINKEDIN_CONNECTION",
      "recommendation": "Add a HAS_LINKEDIN_URL check before this step — {{person.linkedinLink.url}} may be null"
    }
  ]
}
```

**Always recommend a property check before:**
- Any SEND_EMAIL step → `HAS_EMAIL_ADDRESS`
- Any SEND_LINKEDIN / SEND_LINKEDIN_CONNECTION / VISIT_LINKEDIN_PROFILE step → `HAS_LINKEDIN_URL`
- Any SEND_WHATSAPP step → `HAS_PHONE_NUMBER`

---

## Gotchas

1. **No `{{trigger.*}}` prefix** — sequence variables never use `trigger`. Person data is always `{{person.*}}`.
2. **Step output UUIDs are not known yet** — return `outputPaths` with placeholder names (`{{step_5_uuid.id}}`). The orchestrator or actions-agent substitutes real UUIDs.
3. **Detection steps (REPLIED_TO_EMAIL etc.) produce no output** — they are listeners, not data sources.
4. **`CREATE_RECORD` step outputs depend on the object** — for a `CREATE_RECORD` with `objectName: "task"`, the outputs are the same as `CREATE_TASK`. For `objectName: "note"`, same as `CREATE_NOTE`. For other objects, output `{{stepId.id}}` plus any fields in `objectRecord`.
5. **Variables are evaluated per person** — a variable that resolves for one person may be null for another. Always recommend guard checks for optional person fields.
