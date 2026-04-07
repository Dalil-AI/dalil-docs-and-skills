# Metadata API

The Metadata API lets you discover the shape of your workspace: custom fields, pipeline configurations, and workspace members. Before creating records with custom properties, or working with pipelines, you should fetch the relevant metadata.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/rest/metadata/objects/standard-id/{standardId}` | Get field definitions for an entity type |
| GET | `/rest/metadata/pipelines` | List all pipelines in the workspace |
| GET | `/rest/metadata/pipelines/{pipelineId}` | Get field definitions for a specific pipeline |
| GET | `/rest/workspaceMembers` | List all workspace members |

---

## Object Metadata (Custom Fields)

Each entity type has a fixed **standard ID**. Use it to retrieve all field definitions — both standard fields and any custom fields your workspace has added.

```
GET /rest/metadata/objects/standard-id/{standardId}
```

### Standard IDs

| Entity | Standard ID |
|--------|-------------|
| Person | `20202020-e674-48e5-a542-72570eee7213` |
| Company | `20202020-b374-4779-a561-80086cb2e17f` |
| Opportunity | `20202020-9549-49e8-b0e5-1d14de9583c0` |
| Note | `20202020-3e09-4c77-8d0a-1e7b0e5b0c7e` |
| Task | `20202020-a]f3-4c67-a5a0-7d1e5a7c0d22` |
| Note Target | `20202020-3d0e-4c77-8d0a-1e7b0e5b0c7e` |
| Task Target | `20202020-4f3d-4c67-a5a0-7d1e5a7c0d22` |

### Example

```bash
# Get all Person field definitions (including custom fields)
curl -G "https://app.usedalil.ai/rest/metadata/objects/standard-id/20202020-e674-48e5-a542-72570eee7213" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Response Structure

```json
{
  "data": {
    "getObjectMetadataByStandardId": {
      "fields": [
        {
          "name": "jobTitle",
          "label": "Job Title",
          "type": "TEXT",
          "isCustom": false,
          "isActive": true,
          "isNullable": true,
          "defaultValue": null,
          "description": "The job title of the person",
          "options": null
        },
        {
          "name": "myCustomSelectField",
          "label": "Lead Source",
          "type": "SELECT",
          "isCustom": true,
          "isActive": true,
          "isNullable": true,
          "defaultValue": null,
          "description": "",
          "options": [
            {
              "id": "option-uuid",
              "label": "Website",
              "value": "WEBSITE",
              "position": 0,
              "color": "blue"
            },
            {
              "id": "option-uuid-2",
              "label": "Referral",
              "value": "REFERRAL",
              "position": 1,
              "color": "green"
            }
          ]
        }
      ]
    }
  }
}
```

Key fields in the response:

| Field | Description |
|-------|-------------|
| `name` | The JSON key to use when creating/updating records |
| `label` | Human-readable display name |
| `type` | Field type (TEXT, NUMBER, SELECT, CURRENCY, etc.) |
| `isCustom` | `true` for workspace-defined fields |
| `isNullable` | Whether the field can be left empty |
| `options` | For SELECT/MULTI_SELECT: the valid values and their labels |

---

## Pipeline Metadata

### List All Pipelines

```
GET /rest/metadata/pipelines
```

```bash
curl -G "https://app.usedalil.ai/rest/metadata/pipelines" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Returns each pipeline's `id`, `nameSingular`, `namePlural`, `labelSingular`, `labelPlural`, and parent record type. You need `namePlural` to construct pipeline endpoints and `nameSingular` for GraphQL search.

### Get Pipeline Fields

```
GET /rest/metadata/pipelines/{pipelineId}
```

```bash
curl -G "https://app.usedalil.ai/rest/metadata/pipelines/pipeline-uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Returns the field definitions specific to that pipeline, including stage options, amount fields, date fields, and other custom properties. Use this to know what you can send when creating or updating pipeline records.

---

## Workspace Members

List all members of your workspace. Use member UUIDs to populate owner/assignee fields.

```
GET /rest/workspaceMembers
```

```bash
curl -G "https://app.usedalil.ai/rest/workspaceMembers" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

The response includes each member's UUID, name, email, and role. Use these UUIDs for:

| Field | Entity |
|-------|--------|
| `ownerId` | Person, Opportunity |
| `assigneeId` | Task |
| `accountOwnerId` | Company |

---

## System Fields to Ignore

The following fields appear in metadata responses but are **read-only** — never include them in create or update requests:

| Field | Description |
|-------|-------------|
| `id` | Record UUID — auto-generated |
| `createdAt` | Creation timestamp — auto-set |
| `updatedAt` | Last update timestamp — auto-set |
| `deletedAt` | Soft-delete timestamp — auto-set |
| `position` | Display order — auto-managed |
| `createdBy` | Creator info — auto-set |
| `groupId` | Group identifier — auto-set |
| `visibilityLevel` | Visibility setting — auto-managed |
| `score` | Computed relevance score — auto-computed |
| `lastContactAt` | Last contact timestamp — auto-computed |

---

## Use Cases

**Discover custom fields before creating records:**
Fetch the entity's metadata to find any workspace-specific fields and their valid values before sending a create request.

**Build dynamic forms:**
Use field metadata to render appropriate input controls — text inputs for TEXT fields, dropdowns for SELECT fields with their options, etc.

**Validate SELECT values:**
Check the `options` array to find valid `value` strings (these are the UPPER_SNAKE_CASE values you send in API requests).

**Populate owner/assignee dropdowns:**
Fetch workspace members and use their UUIDs for `ownerId`, `assigneeId`, and `accountOwnerId`.

**Work with pipelines:**
Always fetch pipeline metadata first to get `namePlural` (for endpoint construction) and field definitions (for knowing what to send).

---

## See Also

- [Pipeline](pipeline.md) — full pipeline workflow that starts with metadata discovery
- [Field Types](field-types.md) — how to format values for each field type
