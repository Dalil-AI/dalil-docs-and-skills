---
name: sequence-agent-lifecycle
description: Sub-agent for Dalil AI sequence lifecycle management — create a new sequence, update status (DRAFT/ACTIVE/DEACTIVATED), verify steps are valid, and activate. Called by sequence-agent orchestrator; never invoke directly.
---

# Dalil AI: Sequence Lifecycle Sub-Agent

## Purpose

This sub-agent handles sequence creation and activation. It is called twice by the orchestrator:
- **Phase A** — create the sequence record (DRAFT status)
- **Phase D** — verify all steps are valid, then activate (set status to ACTIVE)

Do not invoke this skill directly. It is spawned by `sequence-agent`.

---

## Tasks

### Task: CREATE_NEW

Create a new sequence in DRAFT status.

**Steps:**
1. POST to `/rest/sequences` with `{ "name": "{sequenceName}", "status": "DRAFT" }`
2. Return the new `sequenceId`

```bash
curl -s -X POST "https://app.usedalil.ai/rest/sequences" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "{sequenceName}", "status": "DRAFT" }'
```

Response: `.data.sequence.id` → this is the `sequenceId`.

**Return JSON:**
```json
{
  "sequenceId": "uuid",
  "status": "success",
  "error": null
}
```

---

### Task: VERIFY_AND_ACTIVATE

Verify all steps are valid, then activate the sequence.

**Steps:**

1. Fetch the sequence with its steps:
```bash
curl -s -G "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

2. Parse `response.data.sequence.steps` (array of step objects).

3. Check validation:
   - Every step must have `valid: true`
   - There must be exactly one step with `isFirstStep: true`
   - `isFirstStep` step must be `type: "START"`
   - No step should have `type: "EMPTY"` with `isFirstStep: true`
   - Every step's `nextStepIds` must reference real step IDs (no dangling references)

4. If validation passes, activate:
```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "status": "ACTIVE" }'
```

5. Optionally apply sequence-level settings if provided:
```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "settings": { ... } }'
```

**Return JSON:**
```json
{
  "activated": true,
  "sequenceId": "uuid",
  "stepCount": 5,
  "invalidSteps": [],
  "status": "success",
  "error": null
}
```

If validation fails:
```json
{
  "activated": false,
  "sequenceId": "uuid",
  "stepCount": 5,
  "invalidSteps": [
    { "id": "step-uuid", "type": "SEND_EMAIL", "name": "Intro email", "reason": "valid: false — missing subject" }
  ],
  "status": "error",
  "error": "Steps have validation errors"
}
```

---

### Task: DEACTIVATE

Set sequence status to DEACTIVATED.

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "status": "DEACTIVATED" }'
```

**Return JSON:**
```json
{
  "sequenceId": "uuid",
  "status": "success",
  "error": null
}
```

---

### Task: GET_STATUS

Fetch the current sequence status and step count (for diagnostics).

```bash
curl -s -G "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

**Return JSON:**
```json
{
  "sequenceId": "uuid",
  "name": "Q1 Outreach",
  "status": "DRAFT",
  "stepCount": 3,
  "steps": [
    { "id": "uuid", "type": "START", "name": "Start", "valid": true, "isFirstStep": true },
    { "id": "uuid", "type": "SEND_EMAIL", "name": "Intro email", "valid": true, "isFirstStep": false }
  ],
  "error": null
}
```

---

## Error Recovery

| Scenario | Recovery |
|---|---|
| `valid: false` on a step | Report the step name, type, and issue to orchestrator — do NOT activate |
| No START step found | Report: "No step with isFirstStep: true found" |
| More than one `isFirstStep: true` | Report all steps that have it |
| Sequence already ACTIVE | For editing: first PATCH to DEACTIVATED, then re-activate after changes |
| 404 on sequence fetch | Wrong `sequenceId` — report back to orchestrator |

---

## Gotchas

1. **Never activate before all steps are built** — the orchestrator calls VERIFY_AND_ACTIVATE only after actions-agent has completed.
2. **`settings` is a separate PATCH** — apply sequence-level settings (platform limits, timezone, pauseOnReply) in a second PATCH call after activation, or before.
3. **`depth=1` is required** to get steps in the fetch — `depth=0` returns the sequence without steps.
4. **`valid: false` is common when settings are incomplete** — any required field missing from `settings.input` causes this. Don't assume the actions-agent set everything correctly; verify.
