---
name: sequence-variables
description: Reference for {{...}} variable syntax in Dalil AI sequences — available paths for person, sequencePerson, and step outputs at each point in the flow.
---

# Dalil AI: Sequence Variables Reference

## Overview

Variables use double-brace syntax: `{{person.name.firstName}}`

Variables in sequences come from three sources:
1. **Person context** — fields on the enrolled `Person` record (always available)
2. **SequencePerson context** — fields on the enrollment record (always available)
3. **Step outputs** — output from prior steps that produce records (e.g., CREATE_RECORD)

Variables are evaluated at execution time per enrolled person. If a path resolves to `null` or `undefined`, the variable is flagged as `invalidVariables` in `resolvedSequencePeople`.

---

## 1. Person Variables — `{{person.*}}`

These are always available from step 1 onward (the enrolled person's record).

### Name
```
{{person.name.firstName}}          → First name
{{person.name.lastName}}           → Last name
```

### Contact
```
{{person.emails.primaryEmail}}     → Primary email address
{{person.phones.primaryPhoneNumber}}  → Primary phone number
{{person.linkedinLink.url}}        → LinkedIn profile URL
```

### Professional
```
{{person.jobTitle}}                → Job title
{{person.city}}                    → City
{{person.company.name}}            → Company name (via relation)
{{person.company.domainName.primaryLinkUrl}}  → Company website
```

### Identity
```
{{person.id}}                      → Person UUID (use in UPDATE_RECORD, CREATE_TASK relations)
{{person.avatarUrl}}               → Profile photo URL
```

### Custom fields
```
{{person.{fieldName}}}             → Any custom field on the person object
```
Use the exact `name` value of the field from the metadata API.

---

## 2. SequencePerson Variables — `{{sequencePerson.*}}`

Fields on the `SequencePerson` enrollment record.

```
{{sequencePerson.id}}              → SequencePerson UUID
{{sequencePerson.status}}          → ACTIVE | PAUSED
{{sequencePerson.hasReplied}}      → Boolean
{{sequencePerson.hasClickedLink}}  → Boolean
{{sequencePerson.hasUnsubscribed}} → Boolean
```

---

## 3. Step Output Variables — `{{stepId.*}}`

Steps that produce a new record output their result. Only these step types produce output usable in subsequent steps:

| Step Type | Output paths |
|---|---|
| `CREATE_RECORD` | `{{stepId.id}}`, `{{stepId.{fieldName}}}` |
| `CREATE_TASK` | `{{stepId.id}}`, `{{stepId.title}}`, `{{stepId.status}}` |
| `CREATE_NOTE` | `{{stepId.id}}`, `{{stepId.title}}` |

Where `stepId` is the UUID of that specific step (set when the step is created).

Example: if a CREATE_TASK step has `id: "abc-123"`, the created task's ID is available as `{{abc-123.id}}` in later steps.

**Steps that do NOT produce output** (no variables from these):
- SEND_EMAIL, SEND_LINKEDIN, SEND_WHATSAPP, SEND_LINKEDIN_CONNECTION
- VISIT_LINKEDIN_PROFILE, LIKE_LINKEDIN_POST, COMMENT_LINKEDIN_POST
- DELAY, CUSTOM_CONDITION, HAS_* checks
- OPENED_EMAIL, REPLIED_TO_EMAIL, and all other event detection steps
- TRIGGER_WORKFLOW, CREATE_SEQUENCE_PERSON, COMMENT, EMPTY

---

## 4. Usage in Step Settings

### In message bodies (SEND_EMAIL, SEND_LINKEDIN, etc.)
Variables go directly in strings:
```json
{
  "subject": "Hi {{person.name.firstName}}, quick question",
  "body": "<p>Hi {{person.name.firstName}},</p><p>I noticed you work at {{person.company.name}}...</p>"
}
```

### In record fields (CREATE_RECORD, UPDATE_RECORD, CREATE_TASK)
Variables go inside field values:
```json
{
  "objectRecord": {
    "title": "Follow up with {{person.name.firstName}} {{person.name.lastName}}",
    "assigneeId": "{{workspaceMemberId}}"
  }
}
```

### In conditions (CUSTOM_CONDITION)
Variables go inside filter values:
```json
{
  "recordFilters": {
    "field": "jobTitle",
    "operator": "eq",
    "value": "{{person.jobTitle}}"
  }
}
```

---

## 5. Variable Availability by Step Position

Variables are available from the moment the person is enrolled (step 1):

| Step position | Available `{{person.*}}` | Available `{{sequencePerson.*}}` | Available `{{stepId.*}}` |
|---|---|---|---|
| START (step 1) | Yes — all person fields | Yes | No prior steps |
| SEND_EMAIL (step 2) | Yes | Yes | Only if step 1 produced output |
| Any step N | Yes | Yes | Any prior step (1..N-1) that produced output |

**Key difference from workflows:** There is no `{{trigger.*}}` prefix. Person data is directly at `{{person.*}}`.

---

## 6. Invalid Variables

If a variable path doesn't resolve at execution time (e.g., the person has no LinkedIn URL), it becomes an "invalid variable." The `resolvedSequencePeople` query returns:

```json
{
  "steps": [
    {
      "invalidVariables": ["{{person.linkedinLink.url}}"]
    }
  ]
}
```

Use `HAS_LINKEDIN_URL` / `HAS_EMAIL_ADDRESS` / `HAS_PHONE_NUMBER` check steps before sending steps that depend on those fields.

---

## 7. Nested Object Paths

For composite fields on the person, use dot notation:

```
{{person.name.firstName}}          ✓ correct
{{person.firstName}}               ✗ wrong — name is a composite field
{{person.emails.primaryEmail}}     ✓ correct
{{person.email}}                   ✗ wrong — emails is a composite field
{{person.phones.primaryPhoneNumber}} ✓ correct
```

Company relation (one level deep):
```
{{person.company.name}}            ✓ works for direct relation fields
{{person.company.employees[0].name}} ✗ nested arrays not supported
```

---

## Gotchas

1. **No `{{trigger.*}}` in sequences** — workflow-style trigger variables don't exist here. Everything starts with `{{person.*}}`.
2. **Step UUIDs are assigned at creation time** — the `id` you set on `createSequenceStep` (or the auto-generated one) becomes the variable prefix. Plan step IDs before building if you want readable names.
3. **Sending steps have no output** — don't try to reference `{{send-email-step.messageId}}`; it won't work.
4. **Detection steps have no output either** — `OPENED_EMAIL`, `REPLIED_TO_EMAIL`, etc. are listeners, not data producers.
5. **Check for missing data before using it** — always place a `HAS_EMAIL_ADDRESS` or `HAS_LINKEDIN_URL` step before the first sending step that needs that field.
