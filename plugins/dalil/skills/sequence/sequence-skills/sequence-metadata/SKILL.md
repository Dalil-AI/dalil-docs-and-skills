---
name: sequence-metadata
description: Reference for fetching Dalil AI sequence metadata — connected accounts, sequence senders per platform, object field IDs, and workspace members needed to configure sequence steps.
---

# Dalil AI: Sequence Metadata Reference

## What Metadata is Needed

Before configuring sequence steps, you need to fetch:

1. **SequenceSenders** — platform sender identities (email/WhatsApp/LinkedIn) for the workspace
2. **ConnectedAccounts** — OAuth-backed accounts (email providers linked to senders)
3. **Object field metadata** — `fieldMetadataId` UUIDs for fields referenced in CREATE_RECORD / UPDATE_RECORD steps
4. **Workspace members** — for assigning tasks, notes, or filtering by owner
5. **Existing sequences** — for CREATE_SEQUENCE_PERSON steps (need target sequenceId)

---

## 1. Fetch SequenceSenders

Returns all senders configured for this workspace, grouped by platform.

```bash
curl -s -G "https://app.usedalil.ai/rest/sequenceSenders" \
  -H "Authorization: Bearer {apiKey}"
```

Response:
```json
{
  "data": {
    "sequenceSenders": [
      {
        "id": "uuid",
        "platformType": "EMAIL",
        "identifier": "alice@company.com",
        "messageStatus": "ACTIVE",
        "profileVisitStatus": null,
        "commentPostStatus": null,
        "likePostStatus": null,
        "inviteStatus": null,
        "workspaceMember": {
          "id": "workspace-member-uuid",
          "name": { "firstName": "Alice", "lastName": "Smith" }
        },
        "connectedAccount": {
          "id": "connected-account-uuid",
          "handle": "alice@company.com",
          "provider": "google"
        }
      }
    ]
  }
}
```

**Platform types:** `EMAIL` | `WHATSAPP` | `LINKEDIN`

**Sender status fields by platform:**
- Email: `messageStatus`
- LinkedIn: `messageStatus`, `profileVisitStatus`, `commentPostStatus`, `likePostStatus`, `inviteStatus`
- WhatsApp: `messageStatus`

---

## 2. Fetch ConnectedAccounts

OAuth-backed accounts linked to senders. Needed when validating that email/LinkedIn senders have active auth.

```bash
curl -s -G "https://app.usedalil.ai/rest/connectedAccounts" \
  -H "Authorization: Bearer {apiKey}"
```

Response:
```json
{
  "data": {
    "connectedAccounts": [
      {
        "id": "uuid",
        "handle": "alice@company.com",
        "provider": "google",
        "accessTokenExpiresAt": "2025-12-31T00:00:00.000Z",
        "authFailedAt": null
      }
    ]
  }
}
```

Filter for active (non-expired, no auth failure):
- `authFailedAt` should be `null`
- `accessTokenExpiresAt` should be in the future

---

## 3. Fetch Object Field Metadata

Needed for CREATE_RECORD / UPDATE_RECORD steps that reference fields by `fieldMetadataId`.

### List all objects
```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { objects(filter: { isSystem: { is: false } }) { edges { node { id nameSingular namePlural } } } }"
  }'
```

### Get fields for a specific object
```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetObjectFields($filter: ObjectFilterInput) { objects(filter: $filter) { edges { node { nameSingular fieldsList { id name label type isNullable } } } } }",
    "variables": { "filter": { "nameSingular": { "eq": "person" } } }
  }'
```

Response per field:
```json
{
  "id": "field-metadata-uuid",
  "name": "jobTitle",
  "label": "Job Title",
  "type": "TEXT",
  "isNullable": true
}
```

**Important:** Use `field.id` (the `fieldMetadataId`) when building step `objectRecord` for fields that require UUID references. For plain text/number fields, the field `name` string is sufficient.

### Common field names by object

**Person:**
```
name (composite: firstName, lastName)
emails (composite: primaryEmail)
phones (composite: primaryPhoneNumber)
linkedinLink (composite: url)
jobTitle
city
avatarUrl
companyId (relation UUID)
```

**Task:**
```
title
status (SELECT: TODO, IN_PROGRESS, DONE)
dueAt (DATE_TIME)
assigneeId (relation UUID — WorkspaceMember)
```

**Note:**
```
title
body (RICH_TEXT)
```

---

## 4. Fetch Workspace Members

Needed when assigning tasks/notes to specific users, or filtering by owner.

```bash
curl -s -G "https://app.usedalil.ai/rest/workspaceMembers" \
  -H "Authorization: Bearer {apiKey}"
```

Response:
```json
{
  "data": {
    "workspaceMembers": [
      {
        "id": "workspace-member-uuid",
        "name": { "firstName": "Alice", "lastName": "Smith" },
        "userEmail": "alice@company.com",
        "avatarUrl": "..."
      }
    ]
  }
}
```

---

## 5. Find Existing Sequences (for CREATE_SEQUENCE_PERSON)

When a step needs to enroll the person in another sequence, look up the target sequence by name:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { sequences(filter: { name: { like: \"%{name}%\" } }) { edges { node { id name status } } } }"
  }'
```

---

## 6. Metadata Context Object (used in handoff to sub-agents)

When the metadata sub-agent completes, it returns this structured context:

```json
{
  "senders": {
    "email": [
      {
        "id": "sender-uuid",
        "identifier": "alice@company.com",
        "platformType": "EMAIL",
        "messageStatus": "ACTIVE",
        "workspaceMemberId": "member-uuid",
        "connectedAccountId": "account-uuid"
      }
    ],
    "linkedin": [
      {
        "id": "sender-uuid",
        "identifier": "linkedin.com/in/alice",
        "platformType": "LINKEDIN",
        "messageStatus": "ACTIVE",
        "profileVisitStatus": "ACTIVE",
        "commentPostStatus": "ACTIVE",
        "likePostStatus": "ACTIVE",
        "inviteStatus": "ACTIVE",
        "workspaceMemberId": "member-uuid"
      }
    ],
    "whatsapp": []
  },
  "fieldMetadataIds": {
    "person.jobTitle": "field-uuid",
    "task.status": "field-uuid"
  },
  "workspaceMembers": [
    { "id": "member-uuid", "name": { "firstName": "Alice" }, "userEmail": "alice@..." }
  ],
  "targetSequences": {
    "Nurture Sequence": "sequence-uuid"
  }
}
```

---

## Gotchas

1. **Filter senders by platform before using** — email steps need an EMAIL sender, LinkedIn steps need a LINKEDIN sender. Don't assume one sender covers all platforms.
2. **Check `messageStatus: "ACTIVE"`** — a sender with `messageStatus: "PAUSED"` or `"FAILED"` cannot send. Alert the user if no active sender exists for the required platform.
3. **ConnectedAccount `authFailedAt`** — if non-null, the OAuth token is broken. Email/LinkedIn sending will fail. Warn the user.
4. **Use `field.id` for relation fields** — when setting `companyId` or `assigneeId` in `objectRecord`, use the UUID of the target record, not the field's `fieldMetadataId`.
5. **`fieldMetadataId` vs field `name`** — `fieldMetadataId` is needed for metadata API operations; field `name` (e.g., `"jobTitle"`) is what goes in `objectRecord` keys.
6. **Never fetch `/rest/sequenceSenders` with `depth=2`** — it will pull in all related records and create a huge response. Use `depth=0` (default) or `depth=1`.
