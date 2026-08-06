---
name: workflow
description: Manage workflow automations in Dalil AI CRM — create workflows and versions, list/get workflows and their runs, activate or deactivate versions, trigger manual runs, and read run status and output. Use workflow-triggers to configure trigger types, workflow-actions to add steps, and workflow-variables to use step outputs in expressions.
---

# Dalil AI: Workflow API Skills

## Quick Reference

- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Content-Type:** `application/json` *(POST/PATCH requests only)*
- **Accept:** `application/json` *(GET requests)*

### Endpoints & Hosts (across the whole workflow skill family)

The workflow skill family spans **four distinct surfaces on two different hosts** — sending a call to the wrong one is a common mistake.

| Surface | Host + path | Used for |
|---|---|---|
| REST (workflows) | `https://app.usedalil.ai/rest/workflows`, `/workflowVersions`, `/workflowRuns` | Reading workflows, versions, runs |
| Main GraphQL | `POST https://app.usedalil.ai/graphql` | All step/version mutations (create/update/delete step, edges, activate, deactivate, run, serverless functions) |
| Metadata GraphQL | `POST https://app.usedalil.ai/metadata` | Discovering `objectName`s, `fieldMetadataId`s via `fieldsList` (see `workflow-metadata` skill) |
| Field/pipeline metadata REST | `https://api.usedalil.ai/rest/metadata` — **different host** (`api.` not `app.`) | Creating/updating/deleting custom fields and pipelines (see `field` skill) |

**Critical rules:**
- A **Workflow** is a container — all logic lives inside a **WorkflowVersion**
- Versions are **immutable once published** (status = ACTIVE) — to edit, create a new DRAFT via `createDraftFromWorkflowVersion`
- Only one version can be ACTIVE at a time per workflow; activating a new one deactivates the previous
- GraphQL mutations handle all lifecycle operations (activate, deactivate, run, add/remove steps)
- REST is used for reading workflows, versions, and runs

---

## Entities

### Workflow (`/rest/workflows`)

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique identifier |
| `name` | string | Display name |
| `statuses` | string[] | Aggregate statuses of all versions: `DRAFT`, `ACTIVE`, `DEACTIVATED` |
| `lastPublishedVersionId` | UUID \| null | ID of the most recently activated version |
| `createdAt` | ISO string | Creation timestamp |
| `updatedAt` | ISO string | Last modified timestamp |

### WorkflowVersion (`/rest/workflowVersions`)

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique identifier |
| `name` | string | Version label (e.g. "v1") |
| `workflowId` | UUID | Parent workflow |
| `status` | string | `DRAFT`, `ACTIVE`, `DEACTIVATED`, or `ARCHIVED` |
| `trigger` | JSON \| null | Trigger configuration object |
| `steps` | JSON[] \| null | Array of step/action configuration objects |
| `createdAt` | ISO string | Creation timestamp |
| `updatedAt` | ISO string | Last modified timestamp |

### WorkflowRun (`/rest/workflowRuns`)

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique identifier |
| `name` | string | Auto-generated run label |
| `workflowId` | UUID | Parent workflow |
| `workflowVersionId` | UUID | Version that was executed |
| `status` | string | `NOT_STARTED`, `ENQUEUED`, `RUNNING`, `COMPLETED`, or `FAILED` |
| `enqueuedAt` | ISO string \| null | When the run was queued |
| `startedAt` | ISO string \| null | When execution began |
| `endedAt` | ISO string \| null | When execution finished |
| `state` | JSON | Full execution state (see below) |
| `createdBy` | object | Actor who triggered the run |

**`state` object structure:**
```json
{
  "flow": {
    "trigger": { },
    "steps": [ ]
  },
  "stepInfos": {
    "{stepId}": {
      "status": "SUCCESS",
      "output": { }
    }
  },
  "workflowRunError": null,
  "triggerUserId": "uuid",
  "triggerWorkspaceMemberId": "uuid",
  "triggerUserName": "Alice Smith"
}
```

---

## Endpoints

| Operation | Method | Path | Notes |
|---|---|---|---|
| Create workflow | POST | `/rest/workflows` | Creates the container only |
| List workflows | GET | `/rest/workflows` | Supports `filter`, `order_by`, `limit` |
| Get workflow | GET | `/rest/workflows/{id}` | Use `depth=1` to include versions inline |
| Update workflow name | PATCH | `/rest/workflows/{id}` | Only `name` is patchable via REST |
| Delete workflow | DELETE | `/rest/workflows/{id}` | Cascades to versions and runs |
| List versions | GET | `/rest/workflowVersions` | Filter by `workflowId[eq]:{id}` |
| Get version | GET | `/rest/workflowVersions/{id}` | Returns trigger + steps JSON |
| List runs | GET | `/rest/workflowRuns` | Filter by `workflowId`, `status` |
| Get run | GET | `/rest/workflowRuns/{id}` | Includes full `state` with step outputs |

---

## Create a Workflow

Step 1 — Create the workflow container:

```bash
curl -s -X POST "https://app.usedalil.ai/rest/workflows" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "My Automation" }'
```

Response:
```json
{
  "data": {
    "createWorkflow": {
      "id": "workflow-uuid",
      "name": "My Automation"
    }
  }
}
```

**Creating a workflow auto-provisions a DRAFT `v1` version — do not POST to `/rest/workflowVersions`.** `POST /rest/workflowVersions` is not a valid way to create a version and returns `400 "Method not allowed."` Fetch the auto-created version instead:

```bash
curl -s -G "https://app.usedalil.ai/rest/workflows/{workflowId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

Extract the version ID from `.data.workflows[0].versions[0].id` (or `.data.workflow.versions[0].id` depending on singular/plural response shape). The auto-created version will have `status: "DRAFT"` and empty `trigger` and `steps`.

*Example prompts: "Create a new workflow called Lead Nurture", "Set up a blank automation for onboarding"*

---

## List Workflows

```bash
curl -s -G "https://app.usedalil.ai/rest/workflows" \
  --data-urlencode "depth=1" \
  --data-urlencode "limit=50" \
  --data-urlencode "order_by=createdAt[DescNullsLast]" \
  -H "Authorization: Bearer {apiKey}"
```

Filter to active only:
```
filter=statuses[like]:ACTIVE
```

*Example prompts: "List all my active workflows", "Show me all automations"*

---

## Get a Workflow with Its Versions

```bash
curl -s -G "https://app.usedalil.ai/rest/workflows/{workflowId}" \
  --data-urlencode "depth=2" \
  -H "Authorization: Bearer {apiKey}"
```

`depth=2` includes versions, and each version includes its trigger and steps.

---

## Get a Specific Version (with trigger + steps)

```bash
curl -s -G "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

---

## Activate a Version (publish)

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation ActivateWorkflowVersion($workflowVersionId: UUID!) { activateWorkflowVersion(workflowVersionId: $workflowVersionId) }",
    "variables": { "workflowVersionId": "version-uuid" }
  }'
```

Returns `true` on success. Automatically deactivates any currently active version for the same workflow.

*Example prompts: "Publish the workflow", "Activate version v2 of Lead Nurture"*

---

## Deactivate a Version

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation DeactivateWorkflowVersion($workflowVersionId: UUID!) { deactivateWorkflowVersion(workflowVersionId: $workflowVersionId) }",
    "variables": { "workflowVersionId": "version-uuid" }
  }'
```

---

## Create a Draft from an Existing Version

Use this to edit an already-published workflow. Creates a new DRAFT copying the trigger and steps from the source version.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateDraftFromWorkflowVersion($input: CreateDraftFromWorkflowVersionInput!) { createDraftFromWorkflowVersion(input: $input) { id name status trigger steps createdAt updatedAt } }",
    "variables": {
      "input": {
        "workflowId": "workflow-uuid",
        "workflowVersionIdToCopy": "source-version-uuid"
      }
    }
  }'
```

*Example prompts: "I want to edit the active workflow", "Create a new draft to modify the existing automation"*

---

## Trigger a Manual Run

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation RunWorkflowVersion($input: RunWorkflowVersionInput!) { runWorkflowVersion(input: $input) { workflowRunId } }",
    "variables": {
      "input": {
        "workflowVersionId": "version-uuid",
        "payload": { "customField": "value" }
      }
    }
  }'
```

**`RunWorkflowVersionInput` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `workflowVersionId` | UUID | Yes | The version to run |
| `workflowRunId` | UUID | No | Supply to reuse an existing run ID |
| `payload` | JSON | No | Data passed to the trigger as initial context |
| `objectNameSingular` | string | No | Object type for record-based triggers (e.g. `"person"`) |
| `recordId` | UUID | No | Specific record to use as trigger payload |

Returns `{ workflowRunId: "uuid" }`.

*Example prompts: "Run the Lead Nurture workflow for person abc-123", "Trigger the onboarding automation manually"*

---

## Read Run Status and Output

```bash
curl -s -G "https://app.usedalil.ai/rest/workflowRuns/{runId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

Check status via `jq`:
```bash
curl -s -G "https://app.usedalil.ai/rest/workflowRuns/{runId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}" | \
  jq '{ status: .data.workflowRun.status, error: .data.workflowRun.state.workflowRunError, steps: (.data.workflowRun.state.stepInfos | to_entries | map({ step: .key, status: .value.status })) }'
```

List recent runs for a workflow, newest first:
```bash
curl -s -G "https://app.usedalil.ai/rest/workflowRuns" \
  --data-urlencode "filter=workflowId[eq]:{workflowId}" \
  --data-urlencode "order_by=createdAt[DescNullsLast]" \
  --data-urlencode "limit=10" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

List failed runs:
```bash
--data-urlencode "filter=status[eq]:FAILED"
```

*Example prompts: "Did the last run succeed?", "Show me which steps failed in run xyz"*

---

## Apply a Pre-built Template

Templates overwrite the steps on an existing workflow version.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateWorkflowFromTemplate($input: CreateWorkflowFromTemplateInput!) { createWorkflowFromTemplate(input: $input) { success workflowId workflowVersionId stepsCreated } }",
    "variables": {
      "input": {
        "workflowId": "workflow-uuid",
        "workflowVersionId": "version-uuid",
        "templateType": "DEAL_FROM_PERSON"
      }
    }
  }'
```

**Available `templateType` values:**

| Value | Description |
|---|---|
| `DEAL_FROM_PERSON` | Manually create an opportunity from a person record |
| `DEAL_FROM_COMPANY` | Manually create an opportunity from a company record |
| `CUSTOMER_SUCCESS_HANDOFF` | Handoff workflow triggered on opportunity stage change |
| `INBOUND_LEAD_TRIAGE` | Route and qualify inbound leads automatically |

---

## Filter Examples

```
# Workflows with an active version
filter=statuses[like]:ACTIVE

# Runs for a specific workflow
filter=workflowId[eq]:{workflowId}

# Runs by status
filter=status[eq]:FAILED
filter=status[eq]:COMPLETED

# Versions for a specific workflow in DRAFT
filter=workflowId[eq]:{workflowId},status[eq]:DRAFT

# Runs started after a date
filter=startedAt[gt]:2026-01-01T00:00:00.000Z
```

---

## Gotchas

1. **Workflow ≠ WorkflowVersion, but version creation is automatic** — Creating a workflow via `POST /rest/workflows` auto-provisions a DRAFT `v1` version. Fetch it via `GET /rest/workflows/{id}?depth=1` → `.versions[0].id`. Do NOT `POST /rest/workflowVersions` — that endpoint returns `400 "Method not allowed."`
2. **Versions are immutable after activation** — Once `status = ACTIVE`, the `trigger` and `steps` fields are frozen. Use `createDraftFromWorkflowVersion` to get an editable copy.
3. **Only one ACTIVE version per workflow** — Activating a new version automatically deactivates the previous one.
4. **All step editing is via GraphQL, not REST** — You cannot PATCH `trigger` or `steps` directly on `/rest/workflowVersions`. Use `createWorkflowVersionStep`, `updateWorkflowVersionStep`, etc. (covered in `workflow-actions` skill).
5. **`depth=1` required for nested data** — Calling `/rest/workflows/{id}` without `depth=1` returns only top-level fields; versions will not be included.
6. **`state.stepInfos` is keyed by step UUID** — To look up a step's output you need its `id`, not its `name`. Match against the `steps` array in `state.flow.steps` to correlate name → id → output.
7. **REST response wraps results** — The response shape is `{ "data": { "workflow": {...} } }` (singular) or `{ "data": { "workflows": [...] } }` (plural). Use `.data.workflow` / `.data.workflowVersions` / `.data.workflowRuns` in jq.
8. **`runWorkflowVersion` only works on ACTIVE versions** — Running a DRAFT version will return an error. There is no dry-run/validate endpoint for a DRAFT, so the first real test of a large workflow requires activating it. Reduce that risk with a pre-activation checklist:
   - Every step has `"valid": true`
   - No step has `"objectName": "workflow"` in `settings.input` — that's a ghost placeholder step left over from a `createWorkflowVersionStep` call that was never followed by `updateWorkflowVersionStep`
   - Every step whose output is referenced downstream via `{{stepId.*}}` has had `computeStepOutputSchema` called and the result written back via `updateWorkflowVersionStep`
   - Every SELECT/MULTI_SELECT filter `value` is a real option for that field, JSON-encoded as an array string (e.g. `"[\"WON\"]"`)
9. **`payload` in `runWorkflowVersion` replaces trigger data** — For MANUAL triggers, the `payload` JSON is used as the trigger output and becomes the `{{trigger.*}}` variable source for the run.
10. **Deleting a workflow cascades to versions and runs** — There is no soft delete for workflows. All associated data is permanently removed.
