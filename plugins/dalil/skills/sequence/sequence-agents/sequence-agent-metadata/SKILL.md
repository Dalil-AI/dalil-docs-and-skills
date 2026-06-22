---
name: sequence-agent-metadata
description: Sub-agent for discovering Dalil AI sequence metadata — fetches SequenceSenders per platform, ConnectedAccounts, object field IDs, and workspace members. Called by sequence-agent orchestrator; never invoke directly.
---

# Dalil AI: Sequence Metadata Sub-Agent

## Purpose

Discovers all metadata the actions-agent needs to configure sequence steps correctly:
- Which platform senders are available and active (Email / WhatsApp / LinkedIn)
- ConnectedAccount auth status for senders
- `fieldMetadataId` UUIDs for fields used in CREATE_RECORD / UPDATE_RECORD steps
- Workspace members (for task assignment)
- Target sequence IDs (for CREATE_SEQUENCE_PERSON steps)

This runs in Phase A (parallel with lifecycle-agent). Do not invoke directly — spawned by `sequence-agent`.

---

## Inputs (from orchestrator)

```json
{
  "apiKey": "...",
  "platformsNeeded": ["EMAIL", "LINKEDIN"],
  "objectsNeeded": ["person", "task"],
  "fieldsNeeded": ["person.jobTitle", "person.city", "task.status"],
  "needsWorkspaceMembers": true,
  "targetSequenceNames": ["Nurture Sequence"]
}
```

---

## Step-by-step execution

### Step 1: Fetch SequenceSenders

```bash
curl -s -G "https://app.usedalil.ai/rest/sequenceSenders" \
  -H "Authorization: Bearer {apiKey}"
```

From the response, group senders by `platformType`. For each sender, capture:
- `id` (sequenceSenderId)
- `platformType`
- `identifier`
- `messageStatus`, `profileVisitStatus`, `commentPostStatus`, `likePostStatus`, `inviteStatus`
- `workspaceMember.id`
- `connectedAccount.id` (if present)

Filter to only include senders where the relevant status for the needed action is `"ACTIVE"`.

---

### Step 2: Fetch ConnectedAccounts (if EMAIL or LINKEDIN senders found)

```bash
curl -s -G "https://app.usedalil.ai/rest/connectedAccounts" \
  -H "Authorization: Bearer {apiKey}"
```

For each connected account, note:
- `id`
- `handle`
- `provider`
- `authFailedAt` — if non-null, this account's auth is broken

Cross-reference each sender's `connectedAccount.id` against this list to flag broken auth.

---

### Step 3: Fetch object field metadata (if objectsNeeded is non-empty)

For each object in `objectsNeeded`:

```bash
curl -s -X POST "https://app.usedalil.ai/metadata" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetFields($filter: ObjectFilterInput) { objects(filter: $filter) { edges { node { nameSingular fieldsList { id name label type } } } } }",
    "variables": { "filter": { "nameSingular": { "eq": "{objectName}" } } }
  }'
```

Extract `{ id, name, label, type }` for each field. Build a map:
```json
{ "person.jobTitle": "field-metadata-uuid" }
```
— keyed by `"{objectName}.{fieldName}"` matching items in `fieldsNeeded`.

---

### Step 4: Fetch workspace members (if needsWorkspaceMembers is true)

```bash
curl -s -G "https://app.usedalil.ai/rest/workspaceMembers" \
  -H "Authorization: Bearer {apiKey}"
```

Capture: `id`, `name.firstName`, `name.lastName`, `userEmail`.

---

### Step 5: Find target sequences (if targetSequenceNames is non-empty)

For each name in `targetSequenceNames`:

```bash
curl -s -X POST "https://app.usedalil.ai/graphql" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { sequences(filter: { name: { like: \"%{name}%\" } }) { edges { node { id name status } } } }"
  }'
```

---

## Return JSON

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
        "connectedAccountId": "account-uuid",
        "connectedAccountAuthBroken": false
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
        "workspaceMemberId": "member-uuid",
        "connectedAccountAuthBroken": false
      }
    ],
    "whatsapp": []
  },
  "fieldMetadataIds": {
    "person.jobTitle": "field-uuid",
    "task.status": "field-uuid"
  },
  "workspaceMembers": [
    {
      "id": "member-uuid",
      "name": { "firstName": "Alice", "lastName": "Smith" },
      "userEmail": "alice@company.com"
    }
  ],
  "targetSequences": {
    "Nurture Sequence": "sequence-uuid"
  },
  "warnings": [
    "No LINKEDIN senders found — SEND_LINKEDIN steps will fail at runtime"
  ],
  "status": "success",
  "error": null
}
```

---

## Warnings to Surface

Always check and add to `warnings` array:
- No senders for a platform that is needed: `"No EMAIL senders found"`
- Sender's connected account has broken auth: `"Sender alice@... has broken OAuth (authFailedAt set)"`
- Sender's status for needed action is not ACTIVE: `"LinkedIn sender alice has inviteStatus: PAUSED — connection requests will fail"`
- Target sequence not found: `"Sequence 'Nurture Sequence' not found by name"`
- Field not found on object: `"Field 'customField' not found on person"`

---

## Gotchas

1. **Filter by platform before returning** — only include EMAIL senders in `senders.email`, etc. Don't mix.
2. **`connectedAccount` may be null** for some senders (e.g., SMTP senders without OAuth). That's fine — only flag if the account exists but `authFailedAt` is set.
3. **`depth=0` is sufficient** for `/rest/sequenceSenders` — don't use depth=1 or depth=2.
4. **Field `name` vs `id`** — the `fieldMetadataIds` map should contain the field's `id` (UUID), not its `name` string. The actions-agent uses these for UPDATE_RECORD steps that reference specific fields.
5. **One metadata fetch per object** — don't call the metadata API for the same object twice.
