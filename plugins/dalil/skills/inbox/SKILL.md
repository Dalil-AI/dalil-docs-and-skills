---
name: inbox
description: Search and retrieve messages from a Dalil workspace — fetch recent threads by channel, keyword-search across message subject and body, find threads with no reply, and create drafts for matching threads. Use when the user asks about LinkedIn/email/WhatsApp messages received or sent, wants to find a specific message, needs a summary of recent inbox activity, or wants to draft follow-up messages for threads that haven't received a reply.
---

# Dalil AI: Message Search Skills

## Quick Reference

- **Base URL:** `https://app.usedalil.ai`
- **Auth:** `Authorization: Bearer {apiKey}`
- **API Key:** `{PASTE_YOUR_API_KEY_HERE}` — replace with your Dalil API key before making any requests. You can also set this once in `.claude/CLAUDE.md` so it's available across all skills.
- **Accept:** `application/json`
- **Resource paths:** `/rest/messageParticipants`, `/rest/messageChannelMessageAssociations`

**Critical rules:**
- Use `/rest/messageParticipants?depth=1` — NOT `/rest/messages` directly (requires `messageThreadId`, returns 400 without it)
- Nested field filters (`message.subject[like]:...`) do NOT work server-side — always filter client-side with `jq`
- Channel `type` in `messageChannelMessageAssociations` is **lowercase**: `"email"`, `"linkedin"`, `"whatsapp"`, `"sms"`
- `direction` (OUTGOING/INCOMING) is only on `messageChannelMessageAssociations`, not on `messageParticipants`
- `displayName` is often empty — always match on both `displayName` AND `handle` when filtering by name
- ISO 8601 strings compare lexicographically in jq, so `>=` date comparison works correctly

---

## Two-Call Setup (required for thread-level queries)

Many scenarios need both endpoints saved to temp files first:

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=500" \
  -H "Authorization: Bearer {apiKey}" > /tmp/participants.json

curl -s -G "https://app.usedalil.ai/rest/messageChannelMessageAssociations" \
  --data-urlencode "depth=1" --data-urlencode "limit=500" \
  -H "Authorization: Bearer {apiKey}" > /tmp/channels.json
```

The `jq` block below is reused across all thread-level scenarios — only the `select(...)` changes:

```bash
jq -n \
  --slurpfile p /tmp/participants.json \
  --slurpfile c /tmp/channels.json \
  '
  ($c[0].data.messageChannelMessageAssociations
    | map({key: .messageId, value: {direction: .direction, channelType: .messageChannel.type}})
    | from_entries) as $channelMap |
  $p[0].data.messageParticipants
  | group_by(.message.messageThreadId)
  | map(
      sort_by(.message.receivedAt) | last |
      {
        threadId: .message.messageThreadId,
        lastReceivedAt: .message.receivedAt,
        subject: .message.subject,
        snippet: (.message.text // "" | gsub("[\\n\\r\\t]"; " ") | .[:200]),
        from: .displayName,
        handle: .handle,
        messageCount: length,       # NOTE: this is 1 for all — use group length instead
        direction: ($channelMap[.message.id].direction // "unknown"),
        channelType: ($channelMap[.message.id].channelType // "unknown")
      }
    )
  | map(select(REPLACE_THIS))
  | sort_by(.lastReceivedAt) | reverse
'
```

Replace `REPLACE_THIS` with the `select(...)` condition from the scenario below.

---

## Scenarios

### 1. Recent messages — last N days

Single call, no thread grouping needed.

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  -H "Authorization: Bearer {apiKey}" | \
  jq --arg since "2026-05-08T00:00:00.000Z" \
     '[.data.messageParticipants[] | select(.message.receivedAt // "" >= $since) | {
       receivedAt: .message.receivedAt,
       subject: .message.subject,
       snippet: (.message.text // "" | .[:300]),
       from: .displayName,
       handle: .handle,
       role: .role
     }] | sort_by(.receivedAt) | reverse'
```

*Example prompts: "What messages did I get this week?", "Show me everything from the last 3 days"*

---

### 2. Keyword search — find messages containing a phrase

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  -H "Authorization: Bearer {apiKey}" | \
  jq --arg kw "i came across your business" \
     --arg since "2026-05-08T00:00:00.000Z" \
     '[.data.messageParticipants[] | select(
       (.message.receivedAt // "" >= $since) and (
         (.message.text // "" | ascii_downcase | contains($kw)) or
         (.message.subject // "" | ascii_downcase | contains($kw))
       )
     ) | {
       receivedAt: .message.receivedAt,
       subject: .message.subject,
       snippet: (.message.text // "" | .[:300]),
       from: .displayName,
       handle: .handle
     }] | sort_by(.receivedAt) | reverse'
```

Keyword is lowercase — `ascii_downcase` makes it case-insensitive. Remove the `$since` line to search all time.

*Example prompts: "Find messages mentioning pricing", "Any message saying yes I'm on the way", "Messages about Series A"*

---

### 3. Messages from a specific person (by name or email)

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  -H "Authorization: Bearer {apiKey}" | \
  jq --arg since "2026-05-08T00:00:00.000Z" \
     --arg name "brian chesky" \
     '[.data.messageParticipants[] | select(
       .role == "from" and
       (.message.receivedAt // "" >= $since) and
       (
         (.displayName // "" | ascii_downcase | contains($name)) or
         (.handle // "" | ascii_downcase | contains($name))
       )
     ) | {
       receivedAt: .message.receivedAt,
       subject: .message.subject,
       snippet: (.message.text // "" | .[:300]),
       from: .displayName,
       handle: .handle
     }] | sort_by(.receivedAt) | reverse'
```

*Example prompts: "Last 7 days messages from Brian Chesky", "Has Ahmed messaged me this week?"*

---

### 4. Unread messages (incoming, not yet read)

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  -H "Authorization: Bearer {apiKey}" | \
  jq '[.data.messageParticipants[] | select(
    .role == "from" and
    (.message.isRead == false)
  ) | {
    receivedAt: .message.receivedAt,
    subject: .message.subject,
    snippet: (.message.text // "" | .[:300]),
    from: .displayName,
    handle: .handle
  }] | sort_by(.receivedAt) | reverse'
```

*Example prompts: "What messages haven't I read yet?", "Show my unread inbox"*

---

### 5. Threads where they replied to me — I need to respond (two-call)

Last message in thread is `INCOMING` — they wrote back, ball is in my court.

```
select(.direction == "INCOMING")
```

*Example prompts: "Who replied to me that I haven't responded to?", "Show warm leads waiting on me", "LinkedIn threads where they replied"*

Add channel filter: `select(.direction == "INCOMING" and .channelType == "linkedin")`

---

### 6. Threads where I sent the last message — no reply yet (two-call)

Last message is `OUTGOING` — I wrote last, they haven't replied.

```
select(.direction == "OUTGOING")
```

*Example prompts: "LinkedIn threads where I sent the last message", "Who haven't I heard back from?"*

---

### 7. Threads with no reply in last N days (two-call)

I sent the last message AND it was within the time window — they still haven't replied.

```
select(
  .direction == "OUTGOING" and
  .lastReceivedAt >= "2026-05-08T00:00:00.000Z"
)
```

*Example prompts: "LinkedIn threads where I haven't received a reply in the last 7 days", "Who am I still waiting to hear back from this week?"*

---

### 8. Threads where my last message contains a keyword (two-call)

Last message is outgoing AND the text contains a specific phrase.

```
select(
  .direction == "OUTGOING" and
  .channelType == "linkedin" and
  (.snippet | ascii_downcase | contains("thanks"))
)
```

*Example prompts: "LinkedIn threads where my last message included 'thanks'", "Threads where I said I'd follow up"*

---

### 9. Cold threads — had activity but gone silent (two-call)

Thread had more than one message (real conversation) but last activity was more than N days ago.

```bash
jq -n \
  --slurpfile p /tmp/participants.json \
  --slurpfile c /tmp/channels.json \
  --arg cutoff "2026-04-15T00:00:00.000Z" \
  --arg oldest "2026-01-01T00:00:00.000Z" \
  '
  ($c[0].data.messageChannelMessageAssociations
    | map({key: .messageId, value: {direction: .direction, channelType: .messageChannel.type}})
    | from_entries) as $channelMap |
  $p[0].data.messageParticipants
  | group_by(.message.messageThreadId)
  | map(
      (sort_by(.message.receivedAt) | last) as $last |
      {
        threadId: $last.message.messageThreadId,
        lastReceivedAt: $last.message.receivedAt,
        subject: $last.message.subject,
        snippet: ($last.message.text // "" | gsub("[\\n\\r\\t]"; " ") | .[:150]),
        from: $last.displayName,
        handle: $last.handle,
        messageCount: length,
        direction: ($channelMap[$last.message.id].direction // "unknown"),
        channelType: ($channelMap[$last.message.id].channelType // "unknown")
      }
    )
  | map(select(
      .messageCount > 1 and
      .lastReceivedAt >= $oldest and
      .lastReceivedAt < $cutoff
    ))
  | sort_by(.lastReceivedAt) | reverse
'
```

Set `$cutoff` to 30 days ago (threads silent since then). Set `$oldest` to exclude very old threads.

*Example prompts: "Which conversations went cold in the last month?", "Show me threads that had replies but have gone quiet", "Who was I talking to that I lost touch with?"*

---

### 10. Most active threads — highest back-and-forth (single call)

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  -H "Authorization: Bearer {apiKey}" | \
  jq '[.data.messageParticipants
    | group_by(.message.messageThreadId)[]
    | {
        threadId: .[0].message.messageThreadId,
        subject: .[0].message.subject,
        messageCount: length,
        lastReceivedAt: (map(.message.receivedAt) | sort | last),
        participants: (map(.displayName // .handle) | unique | map(select(. != "" and . != null)))
      }
  ]
  | map(select(.messageCount > 1))
  | sort_by(.messageCount) | reverse
  | .[0:10]'
```

*Example prompts: "Which conversations have had the most back-and-forth?", "Show my most active threads this month"*

---

### 11. Messages by CRM person (by personId)

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "filter=personId[eq]:{personId}" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  -H "Authorization: Bearer {apiKey}" | \
  jq '[.data.messageParticipants[] | {
    role,
    handle,
    receivedAt: .message.receivedAt,
    subject: .message.subject,
    snippet: (.message.text // "" | .[:300]),
    messageId: .message.id
  }] | sort_by(.receivedAt)'
```

*Example prompts: "Show me all messages with this contact", "Full conversation history with Ahmed" (use after looking up personId via person skill)*

---

### 12. Find threads with a keyword in my last outgoing message + no reply + draft a follow-up (three-step)

**Scenario:** "Find messages from the last 10 days that include 'get back to you soon' where I haven't received a reply, and draft 'Want to follow up?' for each."

The logic: last message in thread is OUTGOING (no reply received) AND the snippet contains the keyword AND it was within the time window. Then for each matching thread, POST a draft via the REST API.

**Step 1 — Fetch data**

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=500" \
  -H "Authorization: Bearer {apiKey}" > /tmp/participants.json

curl -s -G "https://app.usedalil.ai/rest/messageChannelMessageAssociations" \
  --data-urlencode "depth=1" --data-urlencode "limit=500" \
  -H "Authorization: Bearer {apiKey}" > /tmp/channels.json
```

**Step 2 — Find matching threads, save threadIds**

```bash
jq -n \
  --slurpfile p /tmp/participants.json \
  --slurpfile c /tmp/channels.json \
  --arg since "2026-05-09T00:00:00.000Z" \
  --arg kw "get back to you soon" \
  '
  ($c[0].data.messageChannelMessageAssociations
    | map({key: .messageId, value: {direction: .direction, channelType: .messageChannel.type}})
    | from_entries) as $channelMap |
  $p[0].data.messageParticipants
  | group_by(.message.messageThreadId)
  | map(
      sort_by(.message.receivedAt) | last |
      {
        threadId: .message.messageThreadId,
        lastReceivedAt: .message.receivedAt,
        subject: .message.subject,
        snippet: (.message.text // "" | gsub("[\\n\\r\\t]"; " ") | .[:200]),
        from: .displayName,
        direction: ($channelMap[.message.id].direction // "unknown"),
        channelType: ($channelMap[.message.id].channelType // "unknown")
      }
    )
  | map(select(
      .direction == "OUTGOING" and
      .lastReceivedAt >= $since and
      (.snippet | ascii_downcase | contains($kw))
    ))
  | sort_by(.lastReceivedAt) | reverse
' > /tmp/matches.json

cat /tmp/matches.json
```

Show the user the matched threads first and confirm before creating drafts.

**Step 3 — Create a draft for each matched thread**

```bash
jq -r '.[].threadId' /tmp/matches.json | while read threadId; do
  curl -s -X POST "https://app.usedalil.ai/rest/messageDrafts" \
    -H "Authorization: Bearer {apiKey}" \
    -H "Content-Type: application/json" \
    -d "{\"messageThreadId\": \"$threadId\", \"draftedText\": \"Want to follow up?\"}" | \
    jq '{created: .data.createMessageDraft.id, threadId: .data.createMessageDraft.messageThreadId}'
done
```

**Key notes for this scenario:**
- `since` date = today minus 10 days (compute from `date -u -v-10d +%Y-%m-%dT%H:%M:%SZ` on macOS or `date -u -d '10 days ago' +%Y-%m-%dT%H:%M:%SZ` on Linux)
- The keyword check is on `snippet` (the last outgoing message text) — `OUTGOING` + keyword together confirm it's the user's own message
- Always show matches to the user before Step 3 — drafts are created per-thread and visible in the inbox immediately
- If a draft already exists for a thread, POST creates a second one — check `/rest/messageDrafts?filter=messageThreadId[eq]:{threadId}` first if idempotency matters
- `draftedText` accepts plain text or HTML — the RichTextComposer in the UI renders it as HTML, so plain text is safe here

*Example prompts: "Find messages from last 10 days where I said 'get back to you soon' and haven't heard back, draft a follow-up for each", "I have threads where I said I'd follow up — write a follow-up draft for them"*

---

## Quick select() Reference (for two-call scenarios)

| What you want | select() condition |
|---|---|
| They replied, I need to respond | `.direction == "INCOMING"` |
| I sent last, waiting for reply | `.direction == "OUTGOING"` |
| No reply in last 7 days | `.direction == "OUTGOING" and .lastReceivedAt >= "7-days-ago"` |
| LinkedIn only | `+ .channelType == "linkedin"` |
| Email only | `+ .channelType == "email"` |
| Last msg contains keyword | `+ (.snippet \| ascii_downcase \| contains("keyword"))` |
| Cold threads (gone silent) | `.messageCount > 1 and .lastReceivedAt < "cutoff-date"` |
| Keyword in my last msg + no reply | `.direction == "OUTGOING" and .lastReceivedAt >= $since and (.snippet \| ascii_downcase \| contains("keyword"))` |

---

## Pagination

If `pageInfo.hasNextPage` is `true`, fetch next page:

```bash
curl -s -G "https://app.usedalil.ai/rest/messageParticipants" \
  --data-urlencode "depth=1" --data-urlencode "limit=100" \
  --data-urlencode "starting_after={endCursor from pageInfo}" \
  -H "Authorization: Bearer {apiKey}"
```

---

## Gotchas

1. **Do NOT use `/rest/messages` directly** — requires `messageThreadId`, returns 400 without it
2. **Nested field filters don't work server-side** — always filter client-side with `jq`
3. **`depth=1` is required** — without it `.message` is null on every record
4. **Channel type is lowercase** — `"linkedin"` not `"LINKEDIN"` in `messageChannelMessageAssociations`
5. **`direction` is only on `messageChannelMessageAssociations`** — not on `messageParticipants`
6. **`displayName` is often empty** — always match on both `displayName` AND `handle` for name searches
7. **Filter `role == "from"` for sender searches** — without it you also match `to`/`cc` recipients
8. **`.message.text` can be null** — always use `// ""` fallback
9. **Keyword matching: use `ascii_downcase | contains("lowercase keyword")`** for case-insensitive search
10. **ISO 8601 date strings compare correctly with `>=` in jq** — no conversion needed
11. **Always use `limit=500` for `messageChannelMessageAssociations`** — the association count grows faster than participants (multiple channels per message); using `limit=200` silently truncates and causes `direction` to resolve as `"unknown"`, dropping valid matches
12. **`messageCount` in two-call scenarios is always 1** — it counts participants per last-message, not messages per thread. Use the `group_by` length for real message count (see scenario 9/10)
