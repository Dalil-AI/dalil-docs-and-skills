# Webhooks

Webhooks allow you to receive real-time notifications when data changes in Dalil AI. When an event occurs — a person is created, an opportunity is updated, a task is deleted — Dalil AI sends an HTTP POST to your specified URL with the event payload.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/rest/webhooks` | Register a webhook |
| GET | `/rest/webhooks` | List registered webhooks |
| DELETE | `/rest/webhooks/{id}` | Remove a webhook |

---

## Supported Entities

| Entity | Value |
|--------|-------|
| Company | `company` |
| People | `people` |
| Opportunity | `opportunity` |
| Task | `task` |

## Supported Actions

| Action | Value | Description |
|--------|-------|-------------|
| Created | `created` | A record was created |
| Updated | `updated` | A record was updated |
| Deleted | `deleted` | A record was deleted |
| All | `*` | Any of the above |

---

## Operation String Format

Webhooks subscribe using an **operation string** that combines entity and action with a dot:

| Pattern | Matches | Example |
|---------|---------|---------|
| `entity.action` | Specific entity + action | `people.created` |
| `entity.*` | All actions for an entity | `company.*` |
| `*.action` | Specific action across all entities | `*.updated` |
| `*` | Everything | `*` |

You can subscribe to multiple operation patterns in a single webhook by providing an array.

---

## Register a Webhook

```
POST /rest/webhooks
```

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `targetUrl` | string | Yes | URL to receive POST requests when events fire |
| `operations` | string[] | Yes | Array of operation patterns to subscribe to |
| `description` | string | No | Human-readable label for this webhook |
| `secret` | string | No | Secret for request validation |

### Example — Subscribe to new contacts

```bash
curl -X POST "https://app.usedalil.ai/rest/webhooks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "targetUrl": "https://your-server.com/webhooks/new-contact",
    "operations": ["people.created"],
    "description": "New contact notifications"
  }'
```

### Example — Subscribe to all company changes

```bash
curl -X POST "https://app.usedalil.ai/rest/webhooks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "targetUrl": "https://your-server.com/webhooks/company",
    "operations": ["company.*"],
    "description": "All company events"
  }'
```

### Example — Subscribe to all events

```bash
curl -X POST "https://app.usedalil.ai/rest/webhooks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "targetUrl": "https://your-server.com/webhooks/all",
    "operations": ["*"],
    "description": "All Dalil AI events"
  }'
```

### Response

```json
{
  "data": {
    "createWebhook": {
      "id": "webhook-uuid-here"
    }
  }
}
```

Save the returned `id` — you'll need it to delete the webhook later.

---

## List Webhooks

```
GET /rest/webhooks
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `filter` | string | — | Filter conditions |
| `limit` | number | 60 | Records per page |

```bash
# Find webhooks pointing to a specific URL
curl -G "https://app.usedalil.ai/rest/webhooks" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "filter=targetUrl[eq]:https://your-server.com/webhooks/all"

# List all webhooks
curl -G "https://app.usedalil.ai/rest/webhooks" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Delete a Webhook

```
DELETE /rest/webhooks/{id}
```

```bash
curl -X DELETE "https://app.usedalil.ai/rest/webhooks/webhook-uuid-here" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

## Webhook Payload

When an event triggers, Dalil AI sends an HTTP POST to your `targetUrl` with the event data in the request body. The payload includes the full record data for the affected entity.

Handle the incoming POST on your server — respond with a `2xx` status code to acknowledge receipt.

---

## Common Patterns

### Audit trail for all deletions

```json
{
  "targetUrl": "https://your-app.com/audit/deletions",
  "operations": ["*.deleted"],
  "description": "Deletion audit trail"
}
```

### Deal stage change notifications

```json
{
  "targetUrl": "https://your-app.com/deals/updates",
  "operations": ["opportunity.updated"],
  "description": "Opportunity stage changes and updates"
}
```

### Sync new contacts to an external system

```json
{
  "targetUrl": "https://your-crm-sync.com/incoming",
  "operations": ["people.created", "company.created"],
  "description": "New record sync to external CRM"
}
```

### Monitor all task activity

```json
{
  "targetUrl": "https://your-app.com/tasks/events",
  "operations": ["task.*"],
  "description": "All task creation, updates, and deletions"
}
```
