---
name: workflow-agent
description: Orchestrator agent for building Dalil AI workflows end-to-end. Decomposes the user's request into parallel sub-agent tasks — trigger setup, metadata discovery, variable resolution, action building, and lifecycle management — each running in an isolated context with only the skill it needs. Use this instead of the individual workflow-* skills when building or modifying a complete workflow.
---

# Dalil AI: Workflow Orchestrator Agent

## Purpose

This skill turns a natural-language workflow request into a fully built, activated Dalil AI workflow by coordinating focused sub-agents. Each sub-agent loads only its own skill file, keeping context small and attention sharp.

**Use this skill when the user wants to:**
- Build a new workflow from scratch
- Add or modify steps in an existing workflow
- Debug a failing workflow run
- Activate, deactivate, or re-version a workflow

**Do NOT use this skill for:**
- Simple CRUD on people/companies/tasks → use the entity skills directly
- Reading workflow run logs only → use `workflow` skill directly

---

## Architecture

```
User request
     │
     ▼
┌─────────────────────────┐
│   workflow-agent        │  ← YOU ARE HERE (orchestrator)
│   (this skill)          │
└──┬───────┬──────────────┘
   │       │  (parallel where independent)
   ▼       ▼
[lifecycle] [trigger-agent]
               │
          ┌────┴──────────────┐
          ▼                   ▼
   [metadata-agent]    [variables-agent]
          │
          ▼
   [actions-agent]
```

### Sub-agent skills (never invoke these directly — orchestrator spawns them)

| Sub-agent skill | Responsibility |
|---|---|
| `workflow-agent-lifecycle` | Create workflow + version, activate/deactivate, read run status |
| `workflow-agent-trigger` | Configure trigger type, compute trigger output schema |
| `workflow-agent-metadata` | Discover objectNames, fieldMetadataIds, connectedAccountIds, serverless function IDs |
| `workflow-agent-variables` | Resolve valid `{{...}}` expressions for each step given trigger type and prior steps |
| `workflow-agent-actions` | Build and execute all step mutations (create, update, connect) |

---

## Orchestration Protocol

### Step 1 — Parse the request

Extract from the user's description:
- **Workflow name** (invent one if not given)
- **Trigger type** — what fires it? (record event, schedule, button, webhook, sequence)
- **Trigger object** — which entity? (person, company, opportunity, etc.)
- **Steps** — ordered list of what the workflow should do
- **Conditions / branches** — any if/else logic?
- **Loops** — any "for each" patterns?
- **Communication** — any email/WhatsApp/LinkedIn sends? (needs connectedAccountId)
- **Code execution** — any serverless functions? (needs serverlessFunctionId)
- **Existing workflow?** — are we editing or creating fresh?

### Step 1b — Resolve the existing workflow (skip for new workflows)

If editing an existing workflow, **resolve it before spawning any agents**:

1. **Search by name via GraphQL** (never list all — the full list can be 30MB+):
```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query SearchWorkflows($filter: WorkflowFilterInput) { workflows(filter: $filter) { edges { node { id name statuses lastPublishedVersionId } } } }",
    "variables": { "filter": { "name": { "like": "%{workflowName}%" } } }
  }'
```

2. **Fetch the workflow at depth=1** to get its versions list (status, id):
```bash
curl -s -G "https://app.usedalil.ai/rest/workflows/{workflowId}" \
  --data-urlencode "depth=1" -H "Authorization: Bearer {apiKey}"
```

3. **Decide which version to work from:**
   - Is there a DRAFT version? → use it directly (fetch its steps at `depth=0`)
   - No DRAFT? → lifecycle-agent will `createDraftFromWorkflowVersion` from `lastPublishedVersionId`

4. **If using a DRAFT: fetch its steps and check for ghosts:**
```bash
curl -s -G "https://app.usedalil.ai/rest/workflowVersions/{draftVersionId}" \
  --data-urlencode "depth=0" -H "Authorization: Bearer {apiKey}"
```
Delete any steps where `valid: false` before proceeding. Log which steps already exist — the actions-agent must NOT re-configure them.

This pre-flight saves actions-agent from discovering a broken DRAFT mid-build.

---

### Step 2 — Plan the build

Produce a **build plan** in this structure before spawning sub-agents:

```json
{
  "workflowName": "string",
  "isNewWorkflow": true,
  "existingWorkflowId": null,
  "existingVersionId": null,
  "trigger": {
    "type": "DATABASE_EVENT | MANUAL | CRON | WEBHOOK | SEQUENCE",
    "objectName": "person | company | ...",
    "notes": "e.g. fires on person.created"
  },
  "metadataNeeded": {
    "objectNames": ["person", "task"],
    "fieldMetadataIds": ["need stage field on opportunity"],
    "connectedAccounts": true,
    "serverlessFunctions": false
  },
  "steps": [
    {
      "order": 1,
      "type": "FIND_RECORDS | CREATE_RECORD | SEND_EMAIL | CONDITION | ...",
      "description": "what this step does",
      "needsMetadata": true,
      "variablesFrom": ["trigger", "step-1-id"]
    }
  ]
}
```

### Step 3 — Spawn sub-agents

Run sub-agents in this order, parallelizing where safe:

#### Phase A — Parallel (no dependencies)
Spawn simultaneously:
1. **lifecycle-agent** — creates workflow + DRAFT version (if new), or creates a new draft from existing (if editing). Returns: `workflowId`, `versionId`.
2. **metadata-agent** — discovers all objectNames, fieldMetadataIds, connectedAccountIds, serverlessFunctionIds needed for the build plan. Returns: a metadata context object.

#### Phase B — After Phase A completes
Spawn simultaneously:
3. **trigger-agent** — sets the trigger on the version using `versionId` from lifecycle-agent + trigger spec from the plan. Returns: `triggerOutputSchema`.
4. **variables-agent** — given the trigger type + planned step types, returns the valid `{{...}}` variable paths available from the trigger and each step output. Returns: a variables map.

#### Phase C — After Phases A + B complete
Spawn sequentially (steps must be built in order — each step's UUID is needed for the next):
5. **actions-agent** — builds every step in order. Receives: `versionId`, full metadata context, variables map, trigger output schema, ordered step plan. Returns: array of built step objects with their UUIDs.

#### Phase D — Final
6. **lifecycle-agent** (second call) — verify no ghost steps (`objectName: "workflow"` with `valid: false`), then activate the version.

### Step 4 — Handoff schema between sub-agents

The orchestrator passes structured context to each sub-agent as part of its prompt — never the raw skill file content of other sub-agents. Use this schema:

```json
{
  "apiKey": "the Dalil API key from CLAUDE.md",
  "workflowId": "uuid or null",
  "versionId": "uuid or null",
  "triggerType": "DATABASE_EVENT",
  "triggerObject": "person",
  "triggerOutputSchema": { },
  "metadataContext": {
    "objects": { "person": { "nameSingular": "person", "namePlural": "people" } },
    "fieldMetadataIds": { "opportunity.stage": "field-uuid" },
    "connectedAccounts": [{ "id": "uuid", "handle": "alice@co.com", "provider": "google" }],
    "serverlessFunctions": []
  },
  "variablesMap": {
    "trigger": ["{{trigger.properties.after.id}}", "{{trigger.properties.after.name.firstName}}"],
    "step-uuid-1": ["{{step-uuid-1.id}}", "{{step-uuid-1.first.id}}"]
  },
  "stepsBuilt": [
    { "order": 1, "id": "step-uuid", "type": "CREATE_RECORD", "name": "Create task" }
  ]
}
```

---

## Sub-agent Prompt Templates

When spawning each sub-agent, use the following prompt structures. Fill in `{...}` with actual values.

### lifecycle-agent prompt
```
You are a Dalil AI workflow lifecycle sub-agent. Read your skill file at:
.claude/skills/workflow/workflow-agents/workflow-agent-lifecycle/SKILL.md

Task: {CREATE_NEW | CREATE_DRAFT_FROM_EXISTING | ACTIVATE | VERIFY_AND_ACTIVATE}

Context:
- API key: {apiKey}
- Workflow name: {workflowName}
- Existing workflow ID: {existingWorkflowId or null}
- Existing version ID: {existingVersionId or null}

If CREATE_NEW: create a workflow named "{workflowName}", then create a DRAFT version named "v1". Return workflowId and versionId.
If CREATE_DRAFT_FROM_EXISTING: call createDraftFromWorkflowVersion using workflowId={existingWorkflowId} and workflowVersionIdToCopy={existingVersionId}. Return new versionId.
If VERIFY_AND_ACTIVATE: fetch the version at versionId={versionId}, confirm all steps have valid=true and no step has objectName="workflow". Then activate. Return success or list of invalid steps.

Return JSON: { "workflowId": "...", "versionId": "...", "status": "success|error", "error": null }
```

### metadata-agent prompt
```
You are a Dalil AI workflow metadata sub-agent. Read your skill file at:
.claude/skills/workflow/workflow-agents/workflow-agent-metadata/SKILL.md

Task: Discover all metadata needed for this workflow build.

API key: {apiKey}

Needed:
- Object field metadata for: {list of objectNames}
- fieldMetadataIds for these specific fields: {list like "opportunity.stage", "person.jobTitle"}
- Connected accounts: {true|false} — needed for {email|whatsapp|linkedin} steps
- Serverless functions: {true|false} — needed for CODE steps

Return JSON:
{
  "objects": { "{nameSingular}": { "nameSingular": "...", "namePlural": "...", "objectMetadataId": "..." } },
  "fieldMetadataIds": { "{objectName}.{fieldName}": "field-uuid" },
  "connectedAccounts": [...],
  "serverlessFunctions": [...]
}
```

### trigger-agent prompt
```
You are a Dalil AI workflow trigger sub-agent. Read your skill file at:
.claude/skills/workflow/workflow-agents/workflow-agent-trigger/SKILL.md

Task: Configure the trigger on this workflow version and compute its output schema.

API key: {apiKey}
Version ID: {versionId}

Trigger spec:
- Type: {DATABASE_EVENT | MANUAL | CRON | WEBHOOK | SEQUENCE}
- Object: {person | company | opportunity | ...}
- Event: {created | updated | deleted} (DATABASE_EVENT only)
- Availability: {GLOBAL | SINGLE_RECORD | BULK_RECORDS} (MANUAL only)
- Schedule: {schedule object} (CRON only)
- HTTP method: {GET | POST} + expectedBody (WEBHOOK only)

Steps:
1. PATCH the trigger onto the version
2. Call computeStepOutputSchema for the trigger
3. Re-PATCH the version with the computed outputSchema

Return JSON: { "triggerSet": true, "outputSchema": { ... } }
```

### variables-agent prompt
```
You are a Dalil AI workflow variables sub-agent. Read your skill file at:
.claude/skills/workflow/workflow-agents/workflow-agent-variables/SKILL.md

Task: Produce the valid {{...}} variable paths for each step in this workflow.

Trigger type: {triggerType}
Trigger object: {triggerObject}
Trigger output schema: {triggerOutputSchema}

Steps planned (in order):
{array of { order, id, type, description }}

Steps already built (with UUIDs):
{stepsBuilt array}

For each step, list the {{...}} expressions that will be valid when that step runs — based on the trigger type and all prior step output schemas.

Return JSON:
{
  "trigger": ["{{trigger.properties.after.id}}", "{{trigger.properties.after.name.firstName}}", ...],
  "{stepUuid}": ["{{stepUuid.id}}", "{{stepUuid.first.id}}", ...]
}
```

### actions-agent prompt
```
You are a Dalil AI workflow actions sub-agent. Read your skill file at:
.claude/skills/workflow/workflow-agents/workflow-agent-actions/SKILL.md

Task: Build all steps for this workflow version in order.

API key: {apiKey}
Version ID: {versionId}

Metadata context:
{metadataContext JSON}

Variables map:
{variablesMap JSON}

Trigger output schema:
{triggerOutputSchema}

Steps to build (in order):
{array of { order, type, description, config }}

For each step:
1. Call createWorkflowVersionStep to generate its UUID
2. Call updateWorkflowVersionStep with the full configured step object
3. Call computeStepOutputSchema to populate outputSchema
4. Connect to next step via createWorkflowVersionEdge if needed

Return JSON:
{
  "stepsBuilt": [
    { "order": 1, "id": "uuid", "type": "CREATE_RECORD", "name": "..." }
  ],
  "errors": []
}
```

---

## Parallelization Rules

| What can run in parallel | What must be sequential |
|---|---|
| lifecycle-agent + metadata-agent (Phase A) | trigger-agent needs versionId from lifecycle |
| trigger-agent + variables-agent (Phase B) | variables-agent needs triggerType (from plan, not API) |
| — | actions-agent needs all of Phase A + B |
| — | Steps inside actions-agent are sequential (step N's UUID feeds step N+1) |
| — | Final activation needs all steps built |

---

## Error Handling

| Error scenario | Recovery |
|---|---|
| Ghost step found (objectName="workflow") | Identify which createWorkflowVersionStep call produced it, delete it, re-create correctly |
| Step update hit wrong UUID | Fetch the version, find the correct step UUID, re-issue updateWorkflowVersionStep |
| Metadata field not found | Ask the metadata-agent to list all fields for that object and fuzzy-match |
| Version is ACTIVE (not DRAFT) | Tell lifecycle-agent to createDraftFromWorkflowVersion first |
| Variable path empty at runtime | Check trigger type — CRON/WEBHOOK-GET have no trigger output; use FIND_RECORDS first |

---

## Extensibility

To add a new feature domain (e.g. sequences, AI enrichment):

1. Create `.claude/skills/workflow-agent-{feature}/SKILL.md` with the focused skill content
2. Add it to the **Sub-agent skills** table above
3. Add a condition in **Step 1 — Parse the request** to detect when it's needed
4. Add a sub-agent prompt template in **Sub-agent Prompt Templates**
5. Wire it into the **Phase** it belongs to (usually Phase C alongside actions)

No changes to existing sub-agent skills needed.

---

## Gotchas

1. **Always read the build plan back to the user before spawning sub-agents** — confirm the trigger type, step list, and object names before making API calls.
2. **Resolve the existing workflow yourself (Step 1b) — do not delegate this to lifecycle-agent** — doing it inline with targeted calls (search → depth=1 → version depth=0) costs 3 small API calls. Delegating it causes lifecycle-agent to list all workflows at depth=1 (30MB+).
3. **Pass `fieldMetadataIds` from metadata-agent, not from computeStepOutputSchema** — route the orchestrator's metadata-agent to fetch field IDs from `/metadata` with `fieldsList`. Never use `computeStepOutputSchema` as a metadata discovery shortcut.
4. **Ghost step prevention** — every `createWorkflowVersionStep` returns a step with `objectName: "workflow"` by default. Always immediately follow with `updateWorkflowVersionStep`. Never skip this.
5. **ITERATOR auto-creates an EMPTY placeholder child** — inform actions-agent to override `initialLoopStepIds` in the `updateWorkflowVersionStep` call to point to the real first loop step. The EMPTY child can then be deleted or left in place.
6. **DATABASE_EVENT variables are nested** — all trigger fields live at `{{trigger.properties.after.*}}`, not `{{trigger.*}}`. The variables-agent knows this, but double-check any expressions in SEND_EMAIL bodies or CONDITION filters.
7. **CRON triggers have no output** — if the trigger is CRON, the variables-agent will produce an empty trigger variable list. The first action step must be FIND_RECORDS to fetch data.
8. **Version must be DRAFT for all mutations** — if the user is editing an existing ACTIVE workflow, lifecycle-agent must create a draft first. Never attempt to mutate an ACTIVE version.
9. **Verify before activating** — always run the VERIFY_AND_ACTIVATE lifecycle task, not plain activate. A silent ghost step will break the workflow at runtime.
