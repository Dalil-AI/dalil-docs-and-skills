---
name: workflow-agent-lifecycle
description: Sub-agent skill for the workflow-agent orchestrator — handles workflow and version lifecycle operations: create workflow + version, create draft from existing, verify steps are valid, activate, deactivate, and read run status. Not invoked directly by users; spawned by workflow-agent in Phase A and Phase D.
---

# Dalil AI: Lifecycle Sub-Agent Skill

## Role

You are the **lifecycle sub-agent**. You are called twice by the orchestrator:

- **Phase A (start):** Create a new workflow + DRAFT version, OR create a new draft from an existing ACTIVE version
- **Phase D (end):** Verify all steps are valid and then activate the version

You may also be called to deactivate a version, read run status, or debug a failed run.

---

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **REST:** `/rest/workflows`, `/rest/workflowVersions`, `/rest/workflowRuns`
- **GraphQL:** `POST https://app.usedalil.ai/graphql` — activate, deactivate, create draft, run
- **Critical:** Workflow is a container. All logic lives in WorkflowVersion. Only DRAFT versions can be edited.

---

## Task A1 — Create a New Workflow + Version

```bash
# Step 1: Create the workflow container
curl -s -X POST "https://app.usedalil.ai/rest/workflows" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "{workflowName}" }'
```

Response — extract `workflowId`:
```json
{ "data": { "createWorkflow": { "id": "workflow-uuid", "name": "My Automation" } } }
```

```bash
# Step 2: Create a DRAFT version
curl -s -X POST "https://app.usedalil.ai/rest/workflowVersions" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "name": "v1", "workflowId": "{workflow-uuid}" }'
```

Response — extract `versionId`:
```json
{ "data": { "createWorkflowVersion": { "id": "version-uuid", "status": "DRAFT" } } }
```

Return to orchestrator:
```json
{ "workflowId": "uuid", "versionId": "uuid", "status": "success", "error": null }
```

---

## Task A2 — Create a Draft from an Existing Active Version

Use when the user wants to edit a workflow that is already ACTIVE.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreateDraftFromWorkflowVersion($input: CreateDraftFromWorkflowVersionInput!) { createDraftFromWorkflowVersion(input: $input) { id name status trigger steps createdAt updatedAt } }",
    "variables": {
      "input": {
        "workflowId": "{existingWorkflowId}",
        "workflowVersionIdToCopy": "{existingVersionId}"
      }
    }
  }'
```

The new draft copies the existing trigger and steps. Return the new `versionId` to the orchestrator — the trigger-agent and actions-agent will modify this draft.

Return to orchestrator:
```json
{ "workflowId": "{existingWorkflowId}", "versionId": "{new-draft-version-uuid}", "status": "success", "error": null }
```

---

## Task D1 — Verify Steps Before Activation

Fetch the version and check all steps are valid. This is mandatory before activating.

```bash
curl -s -G "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

From `.data.workflowVersion.steps[]`, check:
1. Every step has `"valid": true`
2. No step has `"objectName": "workflow"` in its `settings.input` — this is a ghost placeholder step that will fail at runtime with `"Object cannot be created by workflow"`

If any invalid or ghost steps are found:
```json
{
  "verified": false,
  "ghostSteps": [{ "id": "step-uuid", "name": "step-name", "objectName": "workflow" }],
  "invalidSteps": [{ "id": "step-uuid", "name": "step-name", "valid": false }],
  "error": "Ghost and/or invalid steps found — must be resolved before activation"
}
```

If all steps are valid:
```json
{ "verified": true, "stepCount": 3, "error": null }
```

---

## Task D2 — Activate the Version

Only call this after verification passes.

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation ActivateWorkflowVersion($workflowVersionId: UUID!) { activateWorkflowVersion(workflowVersionId: $workflowVersionId) }",
    "variables": { "workflowVersionId": "{versionId}" }
  }'
```

Returns `true` on success. Automatically deactivates any currently ACTIVE version for the same workflow.

Return to orchestrator:
```json
{ "activated": true, "versionId": "uuid", "error": null }
```

---

## Task — Deactivate a Version

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation DeactivateWorkflowVersion($workflowVersionId: UUID!) { deactivateWorkflowVersion(workflowVersionId: $workflowVersionId) }",
    "variables": { "workflowVersionId": "{versionId}" }
  }'
```

---

## Task — Read Run Status

```bash
curl -s -G "https://app.usedalil.ai/rest/workflowRuns/{runId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}" | \
  jq '{ status: .data.workflowRun.status, error: .data.workflowRun.state.workflowRunError, steps: (.data.workflowRun.state.stepInfos | to_entries | map({ step: .key, status: .value.status, output: .value.output })) }'
```

List recent runs:
```bash
curl -s -G "https://app.usedalil.ai/rest/workflowRuns" \
  --data-urlencode "filter=workflowId[eq]:{workflowId}" \
  --data-urlencode "order_by=createdAt[DescNullsLast]" \
  --data-urlencode "limit=10" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

List failed runs only:
```bash
--data-urlencode "filter=workflowId[eq]:{workflowId},status[eq]:FAILED"
```

`state.stepInfos` is keyed by step UUID. To correlate step name → UUID → output:
1. Fetch the version to get `steps[]` array with `{ id, name }` pairs
2. Match against `state.stepInfos[stepId]`

---

## Task — Trigger a Manual Run (for testing)

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation RunWorkflowVersion($input: RunWorkflowVersionInput!) { runWorkflowVersion(input: $input) { workflowRunId } }",
    "variables": {
      "input": {
        "workflowVersionId": "{versionId}",
        "payload": { }
      }
    }
  }'
```

`runWorkflowVersion` only works on ACTIVE versions. For a MANUAL trigger with a specific record:
- Add `"objectNameSingular": "person"` and `"recordId": "person-uuid"` to `input`

---

## Task — Find an Existing Workflow by Name

**Always search by name first — never list all workflows.** The full list can be 30MB+.

```bash
# Step 1: GraphQL search → get matching workflowIds
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query SearchWorkflows($filter: WorkflowFilterInput) { workflows(filter: $filter) { edges { node { id name statuses lastPublishedVersionId } } } }",
    "variables": { "filter": { "name": { "like": "%Lead Nurture%" } } }
  }'
```

If GraphQL search is unavailable, use REST with a name filter — never without one:
```bash
curl -s -G "https://app.usedalil.ai/rest/workflows" \
  --data-urlencode "filter=name[like]:%Lead Nurture%" \
  --data-urlencode "depth=0" \
  -H "Authorization: Bearer {apiKey}"
```

```bash
# Step 2: Fetch the specific workflow with its versions (depth=1 only — depth=2 is huge)
curl -s -G "https://app.usedalil.ai/rest/workflows/{workflowId}" \
  --data-urlencode "depth=1" \
  -H "Authorization: Bearer {apiKey}"
```

From the response:
- `lastPublishedVersionId` → the currently ACTIVE version
- `.versions[]` → find any DRAFT version (status == "DRAFT") to reuse, or confirm none exists

**To fetch a version's steps without the full workflow payload:**
```bash
curl -s -G "https://app.usedalil.ai/rest/workflowVersions/{versionId}" \
  --data-urlencode "depth=0" \
  -H "Authorization: Bearer {apiKey}"
```

Response: `.data.workflowVersion.steps[]` — each step has `id`, `name`, `type`, `valid`, `nextStepIds`, `settings`.

Filter active workflows: `filter=statuses[like]:ACTIVE`
Response: `.data.workflows[]` or `.data.workflow`

---

## Return Schemas

All returns include a `status` field (`"success"` or `"error"`) and `error` (null or string).

**Phase A — new workflow:**
```json
{ "workflowId": "uuid", "versionId": "uuid", "isNew": true, "status": "success", "error": null }
```

**Phase A — draft from existing:**
```json
{ "workflowId": "uuid", "versionId": "new-draft-uuid", "isNew": false, "status": "success", "error": null }
```

**Phase D — verify:**
```json
{ "verified": true, "stepCount": N, "status": "success", "error": null }
```

**Phase D — activate:**
```json
{ "activated": true, "versionId": "uuid", "status": "success", "error": null }
```

---

## Gotchas

1. **Workflow ≠ WorkflowVersion** — `POST /rest/workflows` creates only the container with no trigger or steps. Always create a version after.
2. **Versions are immutable once ACTIVE** — editing an ACTIVE version will fail. Use `createDraftFromWorkflowVersion` to create an editable copy.
3. **Only one ACTIVE version per workflow** — activating a new version automatically deactivates the previous one.
4. **`runWorkflowVersion` requires ACTIVE status** — you cannot test-run a DRAFT. Activate first.
5. **`depth=1` required for nested data** — calling `/rest/workflows/{id}` without depth returns only top-level fields.
6. **Never use `depth=2` on a workflow** — it pulls all versions × all steps × all run history and can exceed 30MB. Fetch the version separately with `depth=0` instead.
7. **Never list all workflows without a filter** — the full list can be 30MB+. Always search by name first (GraphQL) or filter by name in REST.
8. **Ghost step pattern** — `objectName: "workflow"` with `valid: false` means a `createWorkflowVersionStep` was called but the follow-up `updateWorkflowVersionStep` was never issued. These must be deleted before activation or they will fail at runtime.
9. **REST response wraps results** — `{ "data": { "workflow": {...} } }` (singular) or `{ "data": { "workflows": [...] } }` (plural). Never access `.data[]` directly.
10. **Deleting a workflow cascades** — `DELETE /rest/workflows/{id}` permanently removes all versions and runs. No soft delete. Confirm before proceeding.
