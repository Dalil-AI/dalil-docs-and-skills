# Dalil Workflow Skill — Friction Log & Improvement Suggestions

Compiled from building two workflows ("New Lead → COLD/WARM" and the 39-step "Dalil AI SDR")
by mirroring exported definitions into a fresh workspace. Each item = what went wrong + the
concrete doc/skill fix.

Ordered roughly by impact (failed API calls first, then undocumented-but-essential, then
misleading specs, then ergonomics).

---

## A. Documentation that is WRONG and causes failed calls

### A1. "Create a draft version" via POST is not allowed — versions are auto-created
- **What happened:** Followed `workflow` skill "Step 2 — Create a draft version"
  (`POST /rest/workflowVersions` with `{name, workflowId}`). Returned **400 "Method not
  allowed."** Turns out creating the workflow container **auto-creates a `v1` DRAFT version**.
- **Cost:** One hard failure mid-build; had to GET the workflow at `depth=1` to discover the
  auto-version and recover.
- **Fix:** Both the `workflow` skill AND `workflow-agent-lifecycle` (Task A1) tell you to POST
  a version. Change them to: "Creating a workflow auto-provisions a DRAFT `v1`. Fetch it via
  `GET /rest/workflows/{id}?depth=1` → `.versions[0].id`. Do NOT POST `/rest/workflowVersions`."

### A2. `GET /rest/metadata/fields?filter=objectMetadataId[eq]:{id}` does not scope
- **What happened:** Used the documented field-list filter to get one object's fields; it
  returned a **global union of every field in the workspace** (thousands), not the object's
  fields. The `field`/`workflow-metadata` skills imply this filter scopes.
- **Workaround:** Use `GET /rest/metadata/objects/{id}` and read the embedded `fields[]`, or
  the `/metadata` GraphQL `fieldsList`.
- **Fix:** Note the REST fields-filter does not reliably scope; recommend the object-detail or
  GraphQL `fieldsList` route for getting a single object's `fieldMetadataId`s.

---

## B. Essential mechanics that are UNDOCUMENTED (had to GraphQL-introspect)

### B1. You can pass your own `id` when creating a step — the key to deterministic builds
- **What happened:** `createWorkflowVersionStep` returns `{triggerDiff, stepsDiff}` and the
  skill never explains how to get the **new step's UUID** back. I resorted to diffing the full
  step list before/after every create (an extra GET per step). Only by introspecting
  `CreateWorkflowVersionStepInput` did I find it accepts **`id` (String)** and **`nextStepId`
  (UUID)**. Passing my own UUIDs made the whole 39-step build deterministic.
- **Fix:** Document `id` and `nextStepId` on the create input. Lead with: "Generate a UUID
  client-side, pass it as `id`, and you never have to parse `stepsDiff`." This single fact
  removes the most painful part of programmatic building.

### B2. CONDITION (and ITERATOR) auto-create placeholder child steps
- **What happened:** Creating ONE `CONDITION` step caused **three** steps to appear (the
  condition + two placeholder branch nodes). This broke my "exactly one new step" capture and
  silently litters the version with extra nodes.
- **Cost:** A full failed build + teardown + rebuild.
- **Fix:** The orchestrator gotcha mentions ITERATOR's EMPTY child but **nothing** documents
  CONDITION's two auto-children. Add to the `workflow-actions` CONDITION section: "Creating a
  CONDITION auto-spawns two placeholder branch steps. Either reuse them as your true/false
  targets or delete them (`deleteWorkflowVersionStep`) before wiring your own." Same caveat
  belongs on anything else that fans out by default.

### B3. Delete-step / delete-edge signatures are undocumented
- **What happened:** Needed to clean up the auto-placeholders; the skill says
  `deleteWorkflowVersionStep` exists but never gives its input. Had to introspect:
  `DeleteWorkflowVersionStepInput { workflowVersionId: UUID, stepId: String }` (note: **`stepId`,
  not `id`**), and `deleteWorkflowVersionEdge` reuses `CreateWorkflowVersionEdgeInput`
  (`source`/`target`).
- **Fix:** Document both mutations with full input shapes and the `stepId` vs `id` gotcha.

### B4. `AI_CRM_AGENT` step type is completely undocumented
- **What happened:** Two of the SDR's most important steps are `AI_CRM_AGENT`. This type does
  not appear anywhere in `workflow-actions`. I reverse-engineered its settings from the export:
  `{ main: {id, type:"entity", nameSingular}, input: [entity field-selection maps keyed by
  fieldMetadataId], prompt, recordId, includeReason, businessContextDocumentIds }`.
- **Unclear even after reverse-engineering:** how the agent decides which fields to *write*
  (the export's per-field boolean maps seem to gate read access, not writes), and how
  `businessContextDocumentIds` are created/managed.
- **Fix:** Add a full `AI_CRM_AGENT` section: settings shape, how target-entity writes work,
  how to attach a business-context document, and the `prompt` + doc relationship. This is a
  first-class action in real Dalil workflows and currently invisible to the skill.

### B5. ENRICH has a real provider shape AND structured output — both contradict the docs
- **What happened:** `workflow-actions` documents ENRICH as `{ body: "<AI prompt>" }` with
  **empty output**. The actual provider-enrichment steps use a totally different input:
  `{ operations: ["LINKEDIN_BASIC_DETAILS", ...], enrichOptions, selectedOptions, overwrite,
  objectRecordId, objectNameSingular }`. And the downstream "enrichment success" FILTER reads
  `{{enrichStepId.options.liBasicDetails.success}}` — i.e. ENRICH **does** emit structured
  per-operation output, despite the `workflow-variables` table listing ENRICH output as
  "Empty".
- **Fix:** Split ENRICH into (a) freeform AI-prompt enrich and (b) provider enrich
  (`operations`/`selectedOptions`), document the `objectRecordId`/`objectNameSingular` fields,
  and correct the variables table: ENRICH exposes `{{id.options.<opId>.success}}` flags. List
  the valid operation enums (`LINKEDIN_BASIC_DETAILS`, `LINKEDIN_POSTS_PERSON`,
  `LINKEDIN_REACTIONS`, `EMAIL_FINDER`, …).

---

## C. Filter & pipeline specifics that are missing or misleading

### C1. FIND_RECORDS needs BOTH `recordFilters` and a parallel `gqlOperationFilter`
- **What happened:** The real find steps carry a human-readable `recordFilters` array **and** a
  separate `gqlOperationFilter` (`{and:[{id:{in:[...]}}]}`) that is what actually drives the
  query. The skill shows `recordFilters` but never says the `gqlOperationFilter` is required /
  is the real filter. I only got it right by mirroring the export.
- **Fix:** Document that FIND_RECORDS/FIND_PIPELINE_RECORDS require the `gqlOperationFilter`
  alongside the display `recordFilters`, with a worked "find by id" example.

### C2. Two different filter shapes, not clearly distinguished
- FILTER / CONDITION use `stepFilters` + `stepFilterGroups` + `stepOutputKey` (a `{{...}}`
  expression).
- FIND_RECORDS / BULK use `filter.recordFilters` + `recordFilterGroups` + `fieldMetadataId` +
  `gqlOperationFilter`.
- **Fix:** A side-by-side table. Mixing them up silently produces a filter that never matches.

### C3. SELECT filter `value` must be a JSON-encoded array string
- **What happened:** A SELECT filter value is `"[\"SALES_NAV\"]"` (stringified array), not
  `"SALES_NAV"`. The filter-system examples show plain strings.
- **Fix:** Call this out explicitly for SELECT/MULTI_SELECT filters; it's a silent-failure trap.

### C4. Composite sub-field filtering needs `compositeFieldSubFieldName`
- **What happened:** The "is email empty?" condition on `emails.primaryEmail` required
  `"compositeFieldSubFieldName": "primaryEmail"` on the filter. Undocumented.
- **Fix:** Add to the filter-system section for composite fields (EMAILS/PHONES/LINKS/etc.).

### C5. Operand/type casing is inconsistent in practice
- **What happened:** FILTER steps used UPPER (`IS`, `IS_EMPTY`, `type:"SELECT"`), but the
  enrichment filter used `type:"boolean"` (lower) and FIND recordFilters used `operand:"is"`
  (lower). The skill says "both forms exist" but doesn't say which context expects which.
- **Fix:** State the convention per step type, or confirm both are universally accepted.

### C6. ADD_PIPELINE_RECORD uses `fieldsToCreate`; pipeline record `id` == parent record `id`
- **What happened:** ADD_PIPELINE_RECORD uses `fieldsToCreate` (not `fieldsToUpdate`), and the
  later FIND_PIPELINE_RECORDS filters by `id == person.id` — i.e. the pipeline record shares
  the parent record's UUID. Both conventions were inferred from the export, not the docs.
- **Fix:** Document `fieldsToCreate`, and the "pipeline record id mirrors parent record id"
  convention (it's how you re-find the record you just added).

### C7. No documented way to CREATE a pipeline
- **What happened:** The SDR depends on a `coldOutboundEmails` pipeline. Neither the `pipeline`
  nor `field` skill documents creating a pipeline — only operating on existing ones. The user
  had to create it manually in the UI before I could attach fields/steps.
- **Fix:** Either document pipeline creation, or have the `pipeline` skill state up front
  "pipelines must be created in the UI; this skill only operates on existing ones" so it's a
  known prerequisite, not a surprise mid-build.

### C8. Pipeline fields share the parent object's name namespace
- **What happened:** Creating a `sourceDetails` field on the `coldOutbound` pipeline (scoped to
  `person`) failed with a duplicate-name error because `person` already had `sourceDetails`.
- **Fix:** Note in the `field` skill that pipeline-scoped field names must be unique across the
  **parent object's** fields too, not just the pipeline's.

---

## D. Edges, schemas, and host sprawl

### D1. `nextStepIds` vs `createWorkflowVersionEdge` — two sources of truth, relationship unclear
- **What happened:** It's never stated whether setting `nextStepIds` in
  `updateWorkflowVersionStep` *creates* edges or merely records them. Create-with-`parentStepId`
  auto-makes edges; for fan-in (4 steps → 1) I defensively added explicit
  `createWorkflowVersionEdge` calls because I couldn't tell if `nextStepIds` alone would wire
  them.
- **Fix:** Clarify the model: is the edge graph derived from `nextStepIds`, or is it
  independent? Give the canonical "fan-in / fan-out" recipe (which combination of
  `parentStepId`, `nextStepIds`, and explicit edges to use).

### D2. `computeStepOutputSchema` is load-bearing and easy to under-call
- **What happened:** Downstream `{{stepId.first.field}}` references resolve to empty at runtime
  unless you compute the producing step's output schema. For FIND steps feeding filters this
  matters a lot. The skill mentions it but undersells how often it's mandatory.
- **Fix:** Add a rule: "compute the output schema for every step whose output is referenced by
  a later step (FIND_RECORDS, FIND_PIPELINE_RECORDS, ADD_PIPELINE_RECORD, etc.) before wiring
  the consumer." Note it returns `{}` for HTTP_REQUEST/CODE.

### D3. HTTP_REQUEST `body`: object vs stringified-JSON ambiguity
- **What happened:** `workflow-actions` shows `body` as a JSON object; the real SmartLead step
  used `body` as a **stringified** JSON string. Unclear which the engine wants.
- **Fix:** State the expected type (and whether `{{...}}` interpolation works inside a string
  body vs an object body).

### D4. Endpoint/host sprawl across the skill family
- Workflow REST + GraphQL: `app.usedalil.ai/rest`, `app.usedalil.ai/graphql`
- Metadata GraphQL: `app.usedalil.ai/metadata` (different path, same host)
- Field/pipeline metadata REST: `api.usedalil.ai/rest/metadata` (**different host**)
- **What happened:** Easy to send a call to the wrong host/path. I mixed them up at least once.
- **Fix:** One consolidated "Endpoints & Hosts" table at the top of the skill family covering
  all four surfaces and what lives where.

---

## E. Ergonomics / capability gaps

### E1. No way to validate or test-run a DRAFT
- **What happened:** `runWorkflowVersion` only works on ACTIVE versions. So a 39-step DRAFT
  can't be dry-run before activation — the first real test requires going live. Risky for big
  workflows.
- **Fix:** If a validate/dry-run endpoint exists, document it. Otherwise add a "pre-activation
  checklist" (all steps `valid:true`, no `objectName:"workflow"` ghosts, every referenced
  variable's producer has a computed output schema, every SELECT value is a real option).

### E2. `updateWorkflowVersionStep` is full-replace — carry everything every time
- Documented (gotcha #2) but worth elevating: every update must include `nextStepIds`,
  `settings.input` in full, etc., or you silently wipe wiring/fields. Bit me until I made every
  update carry the complete step object.

---

## What worked well (keep it)
- The `field` skill's type reference and SELECT option rules were accurate and complete.
- `computeStepOutputSchema` returning a paste-ready schema is a nice pattern.
- The ghost-step detection guidance (`objectName:"workflow"` / `valid:false`) was correct and
  caught real issues.
- Mirroring exported JSON is, in practice, the most reliable way to learn the true step shapes —
  which itself argues for documenting those shapes directly in the skill.
