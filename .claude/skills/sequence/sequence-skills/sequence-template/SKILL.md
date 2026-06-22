---
name: sequence-template
description: Reference for Dalil AI sequence templates — available template types, the createSequenceFromTemplate mutation, how templates are defined as step graphs, and how to add new templates in the codebase.
---

# Dalil AI: Sequence Template Reference

## Overview

Sequence templates are pre-built step graphs that create a fully-wired sequence with a single mutation call. They give users a starting point for common outreach patterns without manually adding and connecting steps.

**Key concepts:**
- A template is defined as a `SequenceTemplate` — metadata + an ordered array of `SequenceTemplateStep` objects.
- Steps use symbolic `stepKey` references (e.g. `"email"`, `"whatsapp"`) to express parent/child and next-step relationships. Actual UUIDs are generated at creation time.
- `SequenceTemplateService.createSequenceFromTemplate` walks the steps array in order, calling `createSequenceStep` then `updateSequenceStep` for each step.
- Conditional steps use `trueNextStepKey` / `falseNextStepKey` + `parentConnectionOptions` instead of the linear `nextStepKey`.

---

## GraphQL Mutation

```graphql
mutation CreateSequenceFromTemplate($input: CreateSequenceFromTemplateInput!) {
  createSequenceFromTemplate(input: $input) {
    success
    sequenceId
    stepsCreated
  }
}
```

**Input:**
```json
{
  "name": "My Q1 LinkedIn Outreach",
  "templateType": "LINKEDIN_CONNECTION"
}
```

**`templateType` values** (from `SequenceTemplateType` enum):

| Value | Description |
|---|---|
| `MULTI_CHANNEL_OUTREACH` | Email → WhatsApp → LinkedIn (3 steps) |
| `LINKEDIN_CONNECTION` | LinkedIn connection request → profile visit (2 steps) |
| `PHONE_CONDITIONAL` | HAS_PHONE_NUMBER branch → WhatsApp (true) or CREATE_TASK (false) (3 steps) |

**Response:** `{ success: true, sequenceId: "uuid", stepsCreated: N }`

The created sequence starts in `DRAFT` status. Steps are ready to edit; sending steps have placeholder content that the user should customise before activating.

---

## Available Templates

### MULTI_CHANNEL_OUTREACH

Email → WhatsApp → LinkedIn message. Linear 3-step sequence.

```
Send Email  →  Send WhatsApp Message  →  Send LinkedIn Message
(isFirstStep)   (delay: 2min)             (delay: 2min)
```

Step defaults:
- Email `subject`: `"Introduction - {{currentPerson.name.firstName}}"`
- All messages: greeting + body using `{{currentPerson.name.firstName}}` variable

---

### LINKEDIN_CONNECTION

LinkedIn connection request followed by a profile visit. Linear 2-step sequence.

```
Send LinkedIn Connection Request  →  Visit LinkedIn Profile
(isFirstStep)                        (delay: 1min)
```

Step defaults:
- Connection `message`: personalized connection note using `{{currentPerson.name.firstName}}`

---

### PHONE_CONDITIONAL

Checks if the person has a phone number, then branches:

```
Check If Person Has Phone Number (HAS_PHONE_NUMBER, isFirstStep)
    ├── true  →  Send WhatsApp Message  (delay: 1min)
    └── false →  Create Task - Add Phone Number
```

`CREATE_TASK` input:
```json
{
  "objectRecord": {
    "title": "Add Phone Number",
    "bodyV2": {
      "blocknote": null,
      "markdown": "Please add a phone number for this person: {{currentPerson.name.firstName}} {{currentPerson.name.lastName}}"
    }
  }
}
```

---

## Template Step Schema

Each step in `SequenceTemplate.steps` is a `SequenceTemplateStep`:

```typescript
interface SequenceTemplateStep {
  stepKey: string;                         // symbolic key unique within template
  type: SequenceActionType;
  name: string;
  position: { x: number; y: number };
  isFirstStep?: boolean;                   // exactly one step per template must be true
  parentStepKey?: string;                  // key of the preceding step
  nextStepKey?: string;                    // key of the next step (linear)
  delay?: { unit: 'MINUTE' | 'HOUR' | 'DAY'; value: number };
  input?: Record<string, any>;             // step-type-specific input fields
  // Conditional routing (HAS_* and CUSTOM_CONDITION steps)
  trueNextStepKey?: string;
  falseNextStepKey?: string;
  // For steps that follow a conditional step
  parentConnectionOptions?: { isConnectedToTrue: boolean };
}
```

---

## How Templates Are Processed

`SequenceTemplateService.createSequenceFromTemplate` (server-side, not exposed via API):

1. **Creates a blank DRAFT sequence** with `name` and inherits workspace general settings.
2. **Generates a UUID for every `stepKey`** in the template (`stepIdMap`).
3. **Iterates steps in array order** — calls `createSequenceStep` (creates the node) then `updateSequenceStep` (sets name, settings, routing) for each.
4. **Resolves symbolic keys to UUIDs** using `stepIdMap` before each update.
5. **Conditional steps** — `trueNextStepIds` / `falseNextStepIds` are injected into `settings.input`; `nextStepIds` is set to `[]` on the conditional step itself.
6. **`parentConnectionOptions`** — passed to `createSequenceStep` when a step follows a conditional parent. The service resolves whether the parent is a conditional type and sets `isConnectedToTrue` accordingly.
7. **Step validity** is computed automatically:
   - `SEND_EMAIL`: requires `input.subject` and `input.body`
   - `SEND_WHATSAPP` / `SEND_LINKEDIN`: requires `input.message`
   - `SEND_LINKEDIN_CONNECTION` / `VISIT_LINKEDIN_PROFILE` / `LIKE_LINKEDIN_POST` / `COMMENT_LINKEDIN_POST`: always valid
   - `CREATE_TASK`: requires `input.objectRecord.title`
   - All others: always valid

---

## Adding a New Template (codebase)

1. **Add the enum value** in `template-type.enum.ts`:
   ```typescript
   export enum SequenceTemplateType {
     // ...existing values...
     MY_NEW_TEMPLATE = 'MY_NEW_TEMPLATE',
   }
   ```

2. **Create a template file** at `templates/my-new.template.ts`:
   ```typescript
   import { SequenceActionType } from '...';
   import { SequenceTemplateType } from '../template-type.enum';
   import { type SequenceTemplate } from '../types/sequence-template.type';

   export const MY_NEW_TEMPLATE: SequenceTemplate = {
     type: SequenceTemplateType.MY_NEW_TEMPLATE,
     name: 'Human-Readable Name',
     description: 'Short description shown in the UI',
     steps: [
       {
         stepKey: 'first',
         type: SequenceActionType.SEND_EMAIL,
         name: 'Send Email',
         position: { x: 174, y: 150 },
         isFirstStep: true,
         nextStepKey: 'second',
         input: {
           subject: 'Hello {{currentPerson.name.firstName}}',
           body: '{"type":"doc","content":[...]}',
         },
       },
       // ...more steps
     ],
   };
   ```

3. **Register it** in `templates/index.ts`:
   ```typescript
   import { MY_NEW_TEMPLATE } from './my-new.template';
   
   export const SEQUENCE_TEMPLATES: Record<SequenceTemplateType, SequenceTemplate> = {
     // ...existing entries...
     [SequenceTemplateType.MY_NEW_TEMPLATE]: MY_NEW_TEMPLATE,
   };
   ```

---

## Message Body Format

Template `input.body` (SEND_EMAIL) and `input.message` (SEND_LINKEDIN, SEND_WHATSAPP, SEND_LINKEDIN_CONNECTION) use a **ProseMirror/TipTap JSON doc string**:

```json
"{\"type\":\"doc\",\"content\":[{\"type\":\"paragraph\",\"content\":[{\"type\":\"text\",\"text\":\"Hi \"},{\"type\":\"variableTag\",\"attrs\":{\"variable\":\"{{currentPerson.name.firstName}}\"}},{\"type\":\"text\",\"text\":\",\"},{\"type\":\"hardBreak\"},{\"type\":\"text\",\"text\":\"Your message here\"}]}]}"
```

Key nodes:
- `{ "type": "text", "text": "..." }` — plain text
- `{ "type": "variableTag", "attrs": { "variable": "{{currentPerson.name.firstName}}" } }` — variable substitution
- `{ "type": "hardBreak" }` — line break

The variable namespace is `currentPerson` (not `person`) inside templates.

---

## Gotchas

1. **Steps array order matters** — the service processes steps sequentially. Always list parent steps before their children, and conditional branch steps (true/false) after their conditional parent.
2. **`isFirstStep: true` must appear on exactly one step** — the first step in the array.
3. **Conditional steps omit `nextStepKey`** — they use `trueNextStepKey` / `falseNextStepKey` instead.
4. **`parentConnectionOptions` is required for branches** — any step that follows a conditional (HAS_* or CUSTOM_CONDITION) parent must include `{ isConnectedToTrue: true/false }`.
5. **Template creates a DRAFT sequence** — the user still needs to assign senders and activate. Sending steps with placeholder content will have `valid: true` only if the template provides non-empty subject/body/message values.
6. **`getAvailableTemplates()`** returns the list of registered template keys — if the enum is added but the template is not registered in `SEQUENCE_TEMPLATES`, the mutation will throw `"Template X not found"`.
7. **Variable namespace is `currentPerson`** — unlike step settings edited via the UI where variables may use `person`, template-defined inputs use `currentPerson.name.firstName` etc.
8. **Created sequence starts with no senders** — the template does not inherit or assign any senders; the user must add senders manually before the sequence can be activated.
9. **`steps` field on the sequence is a `RawJSONScalar`** — querying `sequence.steps` via GraphQL returns raw JSON, not a typed subgraph. You cannot select individual subfields (e.g. `steps { id name }`) in GraphQL; parse the scalar client-side instead.
10. **Step `valid: true` even with placeholder content** — sending steps (email, WhatsApp, LinkedIn) are marked valid as long as `subject`/`body`/`message` are non-empty strings. Template placeholder text satisfies this — it does not mean the content is ready to send.
