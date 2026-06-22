---
name: sequence-settings
description: Reference for Dalil AI sequence-level settings — SequenceLimitSettings schema, platform daily limits, send window, timezone, pause behavior, threading, and workflow integrations. Applied via PATCH on the sequence record.
---

# Dalil AI: Sequence Settings Reference

## Overview

Sequence settings (`SequenceLimitSettings`) are stored as RAW_JSON in the `sequence.settings` field. They act as global constraints and behaviors applied to every enrolled person's run. Applied via REST PATCH:

```bash
curl -s -X PATCH "https://app.usedalil.ai/rest/sequences/{sequenceId}" \
  -H "Authorization: Bearer {apiKey}" \
  -H "Content-Type: application/json" \
  -d '{ "settings": { ... } }'
```

Settings can be applied at any time — before or after activation.

---

## Full Settings Schema

```json
{
  "email": {
    "dailyEmails": 50
  },
  "linkedIn": {
    "dailyConnectionRequests": 20,
    "dailyMessages": 15,
    "dailyProfileVisits": 40,
    "dailyComments": 10,
    "dailyLikes": 50
  },
  "whatsApp": {
    "dailyMessages": 30
  },
  "businessDaysOnly": true,
  "activeWindow": {
    "days": [1, 2, 3, 4, 5],
    "window": { "start": "09:00", "end": "17:00" }
  },
  "timezone": "Europe/Paris",
  "sequenceCreatorTimeZone": false,
  "pauseOnReply": true,
  "pauseOnLinkClicked": false,
  "pauseForSameCompany": false,
  "createTaskOnReply": false,
  "createTaskOnLinkClicked": false,
  "priorityForOwner": false,
  "isGeneralSettings": false,
  "isEmailSingleThread": true,
  "timeGapInMinutes": 5,
  "createEngagementTaskForSequenceOwner": false,
  "triggerWorkflowIdOnReply": null,
  "triggerWorkflowIdOnNoReply": null,
  "timeTillNoReply": { "unit": "DAY", "value": 7 }
}
```

---

## Field Reference

### Platform Daily Limits

#### `email`
| Field | Type | Description |
|---|---|---|
| `dailyEmails` | number | Max emails sent per day across all enrolled people |

#### `linkedIn`
| Field | Type | Description |
|---|---|---|
| `dailyConnectionRequests` | number | Max connection requests per day |
| `dailyMessages` | number | Max DMs per day |
| `dailyProfileVisits` | number | Max profile visits per day |
| `dailyComments` | number | Max post comments per day |
| `dailyLikes` | number | Max post likes per day |

#### `whatsApp`
| Field | Type | Description |
|---|---|---|
| `dailyMessages` | number | Max WhatsApp messages per day |

---

### Scheduling

| Field | Type | Default | Description |
|---|---|---|---|
| `businessDaysOnly` | boolean | `false` | Skip Saturdays and Sundays for all steps globally |
| `activeWindow.days` | number[] | all days | DayOfWeek array: `1`=Mon … `7`=Sun — default send days for all steps |
| `activeWindow.window` | `{start, end}` | none | Time window in HH:mm format for all steps (e.g., `"09:00"` to `"17:00"`) |
| `timezone` | string | workspace default | IANA timezone string (e.g., `"Europe/Paris"`, `"America/New_York"`) |
| `sequenceCreatorTimeZone` | boolean | `false` | If true, use the sequence creator's timezone instead of the `timezone` field |
| `timeGapInMinutes` | number | `0` | Minimum gap in minutes between two consecutive actions sent to the same person |

**Step-level scheduling overrides sequence-level** — if a step has its own `days` and `window` in `settings.input`, those take priority over `activeWindow`.

---

### Pause & Stop Behavior

| Field | Type | Default | Description |
|---|---|---|---|
| `pauseOnReply` | boolean | `false` | Pause the person's run when they reply to any message |
| `pauseOnLinkClicked` | boolean | `false` | Pause the person's run when they click a tracked link |
| `pauseForSameCompany` | boolean | `false` | Pause all people from the same company when one of them replies |

**Recommendation:** Set `pauseOnReply: true` for almost every sequence. Continuing to send to someone who already replied is a mistake.

---

### Task Creation

| Field | Type | Default | Description |
|---|---|---|---|
| `createTaskOnReply` | boolean | `false` | Auto-create a task when the person replies |
| `createTaskOnLinkClicked` | boolean | `false` | Auto-create a task when the person clicks a link |
| `createEngagementTaskForSequenceOwner` | boolean | `false` | Create a task for the sequence owner on any engagement event |

---

### Email Threading

| Field | Type | Default | Description |
|---|---|---|---|
| `isEmailSingleThread` | boolean | `false` | All email steps are sent as replies to the first email (single thread per person) |

When `true`, each subsequent `SEND_EMAIL` step continues the same email thread. When `false`, each email is a new conversation.

---

### Workflow Integrations

| Field | Type | Description |
|---|---|---|
| `triggerWorkflowIdOnReply` | string (UUID) \| null | Trigger this workflow when any person replies |
| `triggerWorkflowIdOnNoReply` | string (UUID) \| null | Trigger this workflow when a person hasn't replied after `timeTillNoReply` |
| `timeTillNoReply` | `{ unit, value }` | How long to wait before considering "no reply" — `unit`: `"MINUTE"` \| `"HOUR"` \| `"DAY"` |

---

### Owner & Priority

| Field | Type | Default | Description |
|---|---|---|---|
| `priorityForOwner` | boolean | `false` | Assign sends to the person's owner first (if they have a sender) |
| `isGeneralSettings` | boolean | `false` | Internal flag — marks these settings as workspace-wide defaults; leave false for per-sequence settings |

---

## Common Presets

### Standard business email sequence
```json
{
  "email": { "dailyEmails": 50 },
  "businessDaysOnly": true,
  "activeWindow": {
    "days": [1, 2, 3, 4, 5],
    "window": { "start": "09:00", "end": "17:00" }
  },
  "timezone": "America/New_York",
  "pauseOnReply": true,
  "isEmailSingleThread": true,
  "timeGapInMinutes": 30
}
```

### LinkedIn outreach sequence
```json
{
  "linkedIn": {
    "dailyConnectionRequests": 15,
    "dailyMessages": 10,
    "dailyProfileVisits": 30
  },
  "businessDaysOnly": true,
  "activeWindow": {
    "days": [1, 2, 3, 4, 5],
    "window": { "start": "09:00", "end": "18:00" }
  },
  "pauseOnReply": true,
  "timeGapInMinutes": 60
}
```

### Multi-channel sequence (email + LinkedIn)
```json
{
  "email": { "dailyEmails": 30 },
  "linkedIn": {
    "dailyConnectionRequests": 10,
    "dailyMessages": 8,
    "dailyProfileVisits": 20
  },
  "businessDaysOnly": true,
  "activeWindow": {
    "days": [1, 2, 3, 4, 5],
    "window": { "start": "09:00", "end": "17:00" }
  },
  "timezone": "Europe/London",
  "pauseOnReply": true,
  "pauseForSameCompany": false,
  "isEmailSingleThread": true,
  "timeGapInMinutes": 15,
  "createTaskOnReply": true
}
```

### Aggressive sequence (high volume, no time limits)
```json
{
  "email": { "dailyEmails": 200 },
  "businessDaysOnly": false,
  "pauseOnReply": true,
  "isEmailSingleThread": false
}
```

---

## How to Ask the User for Settings

When building a sequence, ask about these settings in priority order:

1. **`pauseOnReply`** — "Should the sequence automatically pause for a person when they reply?" (almost always yes)
2. **`businessDaysOnly` + `activeWindow`** — "Should steps only run on business days, and within a specific time window?"
3. **`timezone`** — "What timezone should sending times be based on?"
4. **`isEmailSingleThread`** — "Should all emails be sent as a single thread (replies to the first email), or separate emails?"
5. **Platform limits** — "Are there any daily sending limits you want to enforce?"
6. **`triggerWorkflowIdOnReply`** — "Should a workflow be triggered when someone replies?"

---

## Gotchas

1. **Settings are not versioned** — unlike workflow versions, changes to `settings` apply immediately to all ongoing runs.
2. **Step-level `days`/`window` overrides sequence-level `activeWindow`** — per-step scheduling takes precedence. Don't set both to the same value unnecessarily.
3. **`timezone` is IANA format** — use `"America/New_York"` not `"EST"`. Invalid timezone strings cause silent fallback to UTC.
4. **`isEmailSingleThread: true` requires the first email step to have a `subject`** — subsequent steps don't need a subject (they inherit the thread). But if the first step has no subject, threading breaks.
5. **`timeTillNoReply` only matters if `triggerWorkflowIdOnNoReply` is set** — it has no effect on its own.
6. **`dailyEmails` is workspace-wide, not per-person** — a limit of 50 means 50 emails total per day across all people in the sequence, not 50 per person.
7. **Partial PATCH is safe** — you only need to include fields you want to change. Omitted fields retain their current values.
