---
name: sequence-agent
description: Orchestrator agent for building Dalil AI sequences end-to-end. Decomposes the user's request into parallel sub-agent tasks — lifecycle, metadata, trigger setup, variable resolution, and step building — each running in an isolated context with only the skill it needs. Use this instead of individual sequence-* skills when building or modifying a complete sequence.
---

# Dalil AI: Sequence Orchestrator Agent

## Purpose

Turns a natural-language sequence request into a fully built, activated Dalil AI sequence by coordinating focused sub-agents. Each sub-agent loads only its own skill file, keeping context small and attention sharp.

**Use this skill when the user wants to:**
- Build a new outreach sequence from scratch
- Add or modify steps in an existing sequence
- Configure senders, platform limits, or enrollment settings
- Activate, deactivate, or duplicate a sequence

**Do NOT use this skill for:**
- Simple CRUD on people/companies/tasks → use entity skills directly
- Reading sequence run logs only → use `sequence` skill directly

---

## Architecture

```
User request
     │
     ▼
┌─────────────────────────┐
│   sequence-agent        │  ← YOU ARE HERE (orchestrator)
│   (this skill)          │
└──┬──────────────────────┘
   │
   Phase A (parallel):
   ├── sequence-agent-lifecycle    create sequence (DRAFT)
   └── sequence-agent-metadata    fetch senders, field IDs, workspace members
   │
   Phase B (after A, parallel):
   ├── sequence-agent-trigger     create START step, apply sequence settings
   └── sequence-agent-variables   resolve {{...}} paths for all planned steps
   │
   Phase C (after A+B, sequential):
   └── sequence-agent-actions     build every step in order, wire nextStepIds
   │
   Phase D (final):
   └── sequence-agent-lifecycle   verify all steps valid → activate
```

### Sub-agent skills (never invoke these directly — orchestrator spawns them)

| Sub-agent skill | Responsibility |
|---|---|
| `sequence-agent-lifecycle` | Create sequence record, verify steps, activate/deactivate |
| `sequence-agent-metadata` | Fetch senders, connected accounts, field metadata, workspace members |
| `sequence-agent-trigger` | Create START step, apply sequence-level settings (pauseOnReply, limits, timezone) |
| `sequence-agent-variables` | Resolve valid `{{...}}` paths available at each step |
| `sequence-agent-actions` | Build all steps in order (create → configure → connect via nextStepIds) |

---

## Orchestration Protocol

### Step 1 — Parse the request

Extract from the user's description:
- **Sequence name** (invent one if not given)
- **Goal** — what is this outreach trying to achieve?
- **Platforms** — which channels? (email, LinkedIn, WhatsApp)
- **Steps** — ordered list of what the sequence should do
- **Timing** — how many days between steps?
- **Conditions / branches** — any "if they replied" / "if they opened" logic?
- **Enrollment** — how will people be added? (manual, via workflow, from another sequence)
- **Settings** — pause on reply? business days only? timezone? email threading?
- **Existing sequence?** — are we editing or creating fresh?

### Step 1b — Resolve existing sequence (skip for new sequences)

If editing an existing sequence, resolve it before spawning any agents:

1. **Search by name via GraphQL:**
```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { sequences(filter: { name: { like: \"%{name}%\" } }) { edges { node { id name status } } } }"
  }'
```

2. **Fetch with steps:**
```bash
curl -s -G "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

3. **Check which steps already exist** — log their IDs and types. The actions-agent must NOT re-create already-configured steps.

4. **Identify the START step** — find `isFirstStep: true` in the steps array. Pass its `id` as `startStepId` to the trigger-agent (skip the CREATE step) and to the actions-agent.

### Step 2 — Build the plan

Before spawning sub-agents, produce a **build plan** and show it to the user for confirmation:

```json
{
  "sequenceName": "Q1 LinkedIn Outreach",
  "isNewSequence": true,
  "existingSequenceId": null,
  "platforms": ["LINKEDIN", "EMAIL"],
  "enrollmentType": "MANUAL_BULK",
  "sequenceSettings": {
    "pauseOnReply": true,
    "businessDaysOnly": true,
    "activeWindow": { "days": [1,2,3,4,5], "window": { "start": "09:00", "end": "17:00" } },
    "timezone": "Europe/Paris",
    "isEmailSingleThread": true
    // Full schema: sequence-skills/sequence-settings/SKILL.md
  },
  "metadataNeeded": {
    "platformsNeeded": ["LINKEDIN", "EMAIL"],
    "objectsNeeded": [],
    "fieldsNeeded": [],
    "needsWorkspaceMembers": false,
    "targetSequenceNames": []
  },
  "steps": [
    { "order": 1, "type": "START", "description": "Entry point" },
    { "order": 2, "type": "HAS_LINKEDIN_URL", "description": "Check LinkedIn URL exists" },
    { "order": 3, "type": "VISIT_LINKEDIN_PROFILE", "description": "Visit their profile (true branch)" },
    { "order": 4, "type": "DELAY", "description": "Wait 2 days", "config": { "unit": "DAY", "value": 2 } },
    { "order": 5, "type": "SEND_LINKEDIN_CONNECTION", "description": "Send connection request with note" },
    { "order": 6, "type": "DELAY", "description": "Wait 5 days", "config": { "unit": "DAY", "value": 5 } },
    { "order": 7, "type": "ACCEPTED_LINKEDIN_CONNECTION", "description": "Did they accept?" },
    { "order": 8, "type": "SEND_LINKEDIN", "description": "Send intro DM (accepted branch)" },
    { "order": 9, "type": "SEND_EMAIL", "description": "Send email fallback (not accepted branch)" }
  ]
}
```

**Always confirm this plan with the user before spawning sub-agents** — trigger type, step list, platform mix, and settings.

### Step 3 — Spawn sub-agents

Run sub-agents in this order:

#### Phase A — Parallel (no dependencies)
1. **lifecycle-agent** — creates the sequence record in DRAFT status. Returns: `sequenceId`.
2. **metadata-agent** — fetches senders, field IDs, workspace members. Returns: metadata context.

#### Phase B — After Phase A, parallel
3. **trigger-agent** — creates START step, applies sequence-level settings. Receives: `sequenceId`. Returns: `startStepId`, `variableContext`.
4. **variables-agent** — resolves `{{...}}` paths for each planned step. Receives: `stepsPlanned`. Returns: `variablesMap`.

#### Phase C — After Phases A + B, sequential
5. **actions-agent** — builds all steps in order (starting from step 2). Receives: `sequenceId`, `startStepId`, `metadataContext`, `variablesMap`, `stepsToBuild`. Returns: `stepsBuilt`.

#### Phase D — Final
6. **lifecycle-agent** (second call) — verify all steps have `valid: true`, then activate. Returns: activation status.

### Step 4 — Handoff schema between sub-agents

```json
{
  "apiKey": "from CLAUDE.md",
  "sequenceId": "uuid",
  "sequenceName": "string",
  "startStepId": "uuid (from trigger-agent)",
  "metadataContext": {
    "senders": {
      "email": [{ "id": "uuid", "identifier": "alice@co.com", "messageStatus": "ACTIVE" }],
      "linkedin": [],
      "whatsapp": []
    },
    "fieldMetadataIds": { "person.jobTitle": "field-uuid" },
    "workspaceMembers": [{ "id": "uuid", "userEmail": "alice@co.com" }],
    "targetSequences": {}
  },
  "variablesMap": {
    "step_2": { "availableVariables": ["{{person.name.firstName}}", ...] }
  },
  "stepsBuilt": [
    { "order": 1, "id": "start-uuid", "type": "START", "name": "Start" },
    { "order": 2, "id": "step-2-uuid", "type": "SEND_EMAIL", "name": "Intro email" }
  ]
}
```

---

## Sub-Agent Prompt Templates

### lifecycle-agent prompt
```
You are a Dalil AI sequence lifecycle sub-agent. Read your skill file at:
.claude/skills/sequence/sequence-agents/sequence-agent-lifecycle/SKILL.md

Task: {CREATE_NEW | VERIFY_AND_ACTIVATE | DEACTIVATE | GET_STATUS}

Context:
- API key: {apiKey}
- Sequence name: {sequenceName}
- Existing sequence ID: {existingSequenceId or null}

If CREATE_NEW: create a sequence named "{sequenceName}" with status DRAFT. Return sequenceId.
If VERIFY_AND_ACTIVATE: fetch the sequence at sequenceId={sequenceId} with depth=1, verify all steps have valid=true and exactly one isFirstStep=true START step. Then PATCH status to ACTIVE. Return activation result.

Return JSON: { "sequenceId": "...", "activated": true/false, "invalidSteps": [], "status": "success|error", "error": null }
```

### metadata-agent prompt
```
You are a Dalil AI sequence metadata sub-agent. Read your skill file at:
.claude/skills/sequence/sequence-agents/sequence-agent-metadata/SKILL.md

Task: Discover all metadata needed for this sequence build.

API key: {apiKey}

Needed:
- Platforms: {["EMAIL", "LINKEDIN", "WHATSAPP"]} — fetch senders for these platforms
- Object field metadata for: {list of objectNames}
- Fields needed: {list like "person.jobTitle", "task.status"}
- Workspace members: {true|false}
- Target sequence names: {list of sequence names for CREATE_SEQUENCE_PERSON steps}

Return JSON:
{
  "senders": { "email": [...], "linkedin": [...], "whatsapp": [] },
  "fieldMetadataIds": { "person.jobTitle": "uuid" },
  "workspaceMembers": [...],
  "targetSequences": { "Nurture Sequence": "uuid" },
  "warnings": [],
  "status": "success"
}
```

### trigger-agent prompt
```
You are a Dalil AI sequence trigger sub-agent. Read your skill file at:
.claude/skills/sequence/sequence-agents/sequence-agent-trigger/SKILL.md

Task: Create the START step and apply sequence-level settings.

API key: {apiKey}
Sequence ID: {sequenceId}
Enrollment type: {MANUAL_BULK | AUTO_VIA_WORKFLOW | CROSS_SEQUENCE}

Sequence settings to apply:
{sequenceSettings JSON — pauseOnReply, businessDaysOnly, timezone, activeWindow, isEmailSingleThread, email limits, LinkedIn limits, etc.}

Steps:
1. Call createSequenceStep with stepType: "START", isFirstStep: true
2. Fetch the sequence to get the START step UUID
3. Call updateSequenceStep to set valid: true, empty settings
4. PATCH the sequence with the provided settings

Return JSON: { "startStepId": "uuid", "startStepCreated": true, "sequenceSettingsApplied": true, "status": "success" }
```

### variables-agent prompt
```
You are a Dalil AI sequence variables sub-agent. Read your skill file at:
.claude/skills/sequence/sequence-agents/sequence-agent-variables/SKILL.md

Task: Resolve valid {{...}} variable paths for each planned step.

Steps planned (in order):
{array of { order, type, description }}

For each step, list the {{...}} expressions that will be valid — based on person/sequencePerson context and prior step outputs.
Also include recommendations for guard steps (HAS_EMAIL_ADDRESS etc.) that should be added.

Return JSON:
{
  "variablesMap": {
    "step_N": { "type": "...", "availableVariables": [...], "producesOutput": false }
  },
  "recommendations": [
    { "stepOrder": N, "stepType": "...", "recommendation": "..." }
  ],
  "status": "success"
}
```

### actions-agent prompt
```
You are a Dalil AI sequence actions sub-agent. Read your skill file at:
.claude/skills/sequence/sequence-agents/sequence-agent-actions/SKILL.md

Task: Build all steps for this sequence in order (starting from step 2 — START already created).

API key: {apiKey}
Sequence ID: {sequenceId}
START step ID (already created): {startStepId}

Metadata context:
{metadataContext JSON}

Variables map:
{variablesMap JSON}

Steps to build (order 2 onwards):
{array of { order, type, name, config }}

For each step:
1. createSequenceStep to get UUID
2. Fetch sequence to find new step's UUID
3. updateSequenceStep with full settings + valid: true
4. Update previous step's nextStepIds to point to this step

Return JSON:
{
  "stepsBuilt": [{ "order": N, "id": "uuid", "type": "...", "name": "..." }],
  "errors": [],
  "warnings": [],
  "status": "success"
}
```

---

## Parallelization Rules

| What can run in parallel | What must be sequential |
|---|---|
| lifecycle-agent + metadata-agent (Phase A) | trigger-agent needs `sequenceId` from lifecycle |
| trigger-agent + variables-agent (Phase B) | variables-agent is stateless — only needs step types from the plan |
| — | actions-agent needs all of Phase A + B |
| — | Steps inside actions-agent are sequential (step N's UUID needed for step N+1) |
| — | Final activation needs all steps built |

---

## Error Handling

| Scenario | Recovery |
|---|---|
| No senders for required platform | Surface warning to user — steps will fail at runtime. Ask if they want to proceed or skip those steps |
| `valid: false` on step after actions-agent | Report step name and type. Do NOT activate. Ask user to fix the step config |
| No START step found | trigger-agent failed — re-run trigger-agent |
| Sequence already ACTIVE | For editing: DEACTIVATE first (lifecycle-agent task DEACTIVATE), then edit, then re-activate |
| ConnectedAccount auth broken | Warn user: "{identifier}'s OAuth is broken — fix the connected account before activating" |
| Target sequence not found by name | Ask user for the exact sequence name or ID |

---

## Gotchas

1. **Always show the build plan before spawning sub-agents** — confirm step list, platforms, and settings with the user first.
2. **Resolve existing sequence yourself (Step 1b), not via lifecycle-agent** — fetching at depth=1 is 1 targeted API call. Don't delegate this.
3. **No versioning in sequences** — unlike workflows, there is no WorkflowVersion layer. You edit the sequence directly. If ACTIVE, deactivate first.
4. **START step is built by trigger-agent, not actions-agent** — actions-agent receives `startStepId` and only updates its `nextStepIds`.
5. **`createSequenceStep` returns `stepsDiff`, not the step UUID** — actions-agent must fetch the sequence after each creation to get the UUID.
6. **DELAY uses `settings.delay`, not `settings.input`** — this is a common mistake. The actions-agent skill documents this but double-check.
7. **HAS_* and detection steps need two wired branches** — both the true and false paths must lead somewhere. Plan both branches before starting Phase C.
8. **`pauseOnReply: true` is almost always desired** — default to it unless the user explicitly says otherwise.
9. **Sender status matters** — before activating, confirm at least one sender per needed platform has its relevant action status as `ACTIVE` (e.g., `messageStatus: "ACTIVE"` for email/LinkedIn/WhatsApp sends, `inviteStatus: "ACTIVE"` for connection requests).
