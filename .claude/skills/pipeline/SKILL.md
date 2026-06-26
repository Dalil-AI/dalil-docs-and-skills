---
name: pipeline
description: Manage CRM pipelines in Dalil AI — discover pipelines, add/remove/reorder records within a pipeline, and read pipeline stages. Pipelines are separate from opportunities; they track any object (company, person, etc.) through custom stages via GraphQL.
---

# Dalil AI: Pipeline API Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests.
- **All pipeline operations use GraphQL** — `POST /graphql`. There are NO REST endpoints for pipelines.

## Key Concepts

- A **Pipeline** is a metadata object linked to any CRM object (company, person, etc.) via `objectMetadataId`. It has its own `nameSingular`, `namePlural`, `labelSingular`, `labelPlural`, and custom fields.
- A **Pipeline Record** tracks one CRM record's position in a pipeline. It has `recordId` (the CRM object's id) and `position` (a number representing its stage/column).
- Pipelines are discovered by their associated object's `objectMetadataId` — you need to know the object first.

## Step 1 — Discover Pipelines for an Object

```graphql
query {
  findManyPipelinesByObjectMetadataId(objectMetadataId: "uuid-of-object-metadata") {
    id
    nameSingular
    namePlural
    labelSingular
    labelPlural
    description
    icon
    isActive
  }
}
```

To get `objectMetadataId` for a known object (e.g. company), query the metadata API:
```
GET /rest/metadata/objects?filter=nameSingular[eq]:company
```
The `id` field in the response is the `objectMetadataId`.

## Step 2 — List Records in a Pipeline

```graphql
query {
  getPipelineRecords(pipelineId: "uuid-of-pipeline", limit: 60) {
    id
    recordId
    position
    createdAt
    updatedAt
  }
}
```

To get full record details, take the `recordId` values and fetch from the corresponding REST endpoint:
```
GET /rest/companies?filter=id[in]:[recordId1,recordId2]&depth=1
```

## Step 3 — Get Records with All Pipeline Fields

```graphql
query {
  getPipelineRecordsWithAllFields(
    pipelineId: "uuid-of-pipeline"
    limit: 60
  )
}
```

Returns full pipeline record rows including all custom fields defined on the pipeline.

## Get Pipelines for a Specific CRM Record

To see which pipelines a specific record (e.g. a company) is enrolled in:

```graphql
query {
  getPipelinesByRecordId(recordId: "uuid-of-crm-record") {
    pipelines {
      id
      labelSingular
    }
    opportunities {
      id
      recordId
      position
    }
  }
}
```

## Add a Record to a Pipeline

```graphql
mutation {
  createPipelineRecord(
    pipelineId: "uuid-of-pipeline"
    recordId: "uuid-of-crm-record"
  ) {
    id
    recordId
    position
  }
}
```

## Add Multiple Records to a Pipeline

```graphql
mutation {
  batchCreatePipelineRecords(
    pipelineId: "uuid-of-pipeline"
    recordIds: ["uuid1", "uuid2", "uuid3"]
  ) {
    id
    recordId
    position
  }
}
```

## Update a Record's Position in a Pipeline

```graphql
mutation {
  updatePipelineRecord(
    pipelineId: "uuid-of-pipeline"
    recordId: "uuid-of-crm-record"
    position: 2
  ) {
    id
    recordId
    position
  }
}
```

## Update a Custom Field on a Pipeline Record

```graphql
mutation {
  updatePipelineRecordField(
    pipelineId: "uuid-of-pipeline"
    recordId: "uuid-of-crm-record"
    fieldName: "stage"
    fieldValue: { value: "QUALIFIED" }
  ) {
    id
    recordId
    position
  }
}
```

## Remove a Record from a Pipeline

```graphql
mutation {
  deletePipelineRecord(
    pipelineId: "uuid-of-pipeline"
    recordId: "uuid-of-crm-record"
  )
}
```

## Remove Multiple Records from a Pipeline

```graphql
mutation {
  batchDeletePipelineRecords(
    pipelineId: "uuid-of-pipeline"
    recordIds: ["uuid1", "uuid2"]
  )
}
```

## Full Workflow Example

```
# 1. Get objectMetadataId for companies
GET /rest/metadata/objects?filter=nameSingular[eq]:company

# 2. Find pipelines for companies
POST /graphql
{ findManyPipelinesByObjectMetadataId(objectMetadataId: "...") { id labelSingular } }

# 3. List records in a pipeline
POST /graphql
{ getPipelineRecords(pipelineId: "...") { recordId position } }

# 4. Fetch the actual company records
GET /rest/companies?filter=id[in]:[recordId1,recordId2]&depth=1

# 5. Move a company to the next stage
POST /graphql
mutation { updatePipelineRecord(pipelineId: "..." recordId: "..." position: 3) { position } }
```

## Gotchas

1. **GraphQL only** — There are no REST endpoints for pipeline operations. All mutations and queries go to `POST /graphql`.
2. **Pipeline ≠ opportunity stages** — Pipelines are independent objects, not the `stage` field on an opportunity. They can track any CRM object (company, person, etc.).
3. **Must discover pipelines first** — You need the `pipelineId` UUID before any record operations. Get it via `findManyPipelinesByObjectMetadataId`.
4. **`recordId` is the CRM object's id** — Not the pipeline record's own `id`. A pipeline record has both an `id` (its own UUID in the pipeline table) and a `recordId` (the linked company/person/etc UUID).
5. **`position` is a number** — Represents the column/stage index. 0-based or 1-based depends on the pipeline configuration. Use `getPipelineRecords` to see existing positions before updating.
6. **No search on pipeline records** — There is no `searchPipeline` GraphQL query. To find a record in a pipeline, list all pipeline records and match by `recordId`.
7. **Custom fields vary per pipeline** — Use `getPipelineRecordsWithAllFields` to see all fields for a given pipeline's records.
