# Field Types Reference

Dalil AI supports a rich set of field types for both standard and custom properties. This guide explains the exact format required for each type when creating or updating records.

---

## Simple Types

### TEXT

Plain string value.

```json
"jobTitle": "Software Engineer"
```

### NUMBER

Numeric value — integer or float.

```json
"employees": 150
```

### BOOLEAN

```json
"idealCustomerProfile": true
"visaSponsorship": false
```

### DATE_TIME

ISO 8601 string. Always include the time component and `Z` for UTC.

```json
"closeDate": "2024-06-15T00:00:00.000Z"
"dueAt": "2024-03-01T09:00:00.000Z"
```

### SELECT

A single value from a predefined list. **Values must be `UPPER_SNAKE_CASE`.**

```json
"stage": "DISCOVERY"
"status": "IN_PROGRESS"
```

Common values by field:

| Field | Valid Values |
|-------|-------------|
| Opportunity `stage` | `DISCOVERY`, `PROPOSAL`, `NEGOTIATION`, `CLOSED_WON`, `CLOSED_LOST` |
| Task `status` | `TODO`, `IN_PROGRESS`, `DONE` |

> **Note:** Your workspace may have custom SELECT fields with different option values. Use the [Metadata API](metadata.md) to discover valid options.

### MULTI_SELECT

An array of `UPPER_SNAKE_CASE` values.

```json
"workPolicy": ["ON_SITE", "HYBRID", "REMOTE_WORK"]
```

Can also be sent as a comma-separated string:

```json
"workPolicy": "ON_SITE,HYBRID"
```

### RATING

A predefined rating from `RATING_1` to `RATING_5`.

```json
"priority": "RATING_3"
```

### UUID

A UUID string used to reference another record.

```json
"companyId": "550e8400-e29b-41d4-a716-446655440000"
```

---

## Composite Types

### FULL_NAME

Person name as a nested object. **Do not use flat `firstName`/`lastName` at the top level.**

```json
"name": {
  "firstName": "John",
  "lastName": "Smith"
}
```

### EMAILS

```json
"emails": {
  "primaryEmail": "john@example.com",
  "additionalEmails": ["john.backup@example.com"]
}
```

### PHONES

```json
"phones": {
  "primaryPhoneNumber": "5551234567",
  "primaryPhoneCountryCode": "US",
  "primaryPhoneCallingCode": "+1",
  "additionalPhones": []
}
```

> **Phone search tip:** When filtering by phone number, the calling code and number are stored separately. A number like `+15551234567` is stored as calling code `+1` and number `5551234567`. Filter both:
> ```
> filter=phones.primaryPhoneNumber[ilike]:5551234567,phones.primaryPhoneCallingCode[eq]:+1
> ```

### LINKS

Used for LinkedIn, X (Twitter), website domain, and intro video fields. **Always provide all three keys.**

```json
"linkedinLink": {
  "primaryLinkUrl": "https://linkedin.com/in/johnsmith",
  "primaryLinkLabel": "LinkedIn Profile",
  "secondaryLinks": []
}
```

Applies to: `linkedinLink`, `xLink`, `domainName`, `introVideo`

> **Warning:** Omitting `secondaryLinks: []` or sending just a URL string will cause the request to fail or store incomplete data.

### CURRENCY

Amount in micros (multiply actual dollar amount by 1,000,000) plus a currency code.

```json
"amount": {
  "amountMicros": 50000000,
  "currencyCode": "USD"
}
```

| Actual Amount | Micros Value |
|---------------|-------------|
| $1.00 | 1,000,000 |
| $50.00 | 50,000,000 |
| $1,000.00 | 1,000,000,000 |
| $50,000.00 | 50,000,000,000 |
| $500,000.00 | 500,000,000,000 |

> **Warning:** The micros convention is easy to get wrong. `$50,000` is `50000000000` (eleven digits), not `50000000` (eight digits).

### ADDRESS

A flat object — all fields are at the same level, prefixed with `address`.

```json
"address": {
  "addressStreet1": "123 Main Street",
  "addressStreet2": "Suite 200",
  "addressCity": "New York",
  "addressState": "New York",
  "addressPostcode": "10001",
  "addressCountry": "United States",
  "addressLat": 40.7128,
  "addressLng": -74.0060
}
```

### BODY_V2 (Rich Text)

Used for Note and Task body content. Requires both a `markdown` string and a `blocknote` JSON string.

```json
"bodyV2": {
  "markdown": "Meeting notes from today.\nAction items discussed.",
  "blocknote": "[{\"id\":\"unique-uuid-1\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Meeting notes from today.\",\"styles\":{}}],\"children\":[]},{\"id\":\"unique-uuid-2\",\"type\":\"paragraph\",\"props\":{\"textColor\":\"default\",\"backgroundColor\":\"default\",\"textAlignment\":\"left\"},\"content\":[{\"type\":\"text\",\"text\":\"Action items discussed.\",\"styles\":{}}],\"children\":[]}]"
}
```

**Blocknote block structure** — each paragraph is one block:

```json
{
  "id": "unique-uuid",
  "type": "paragraph",
  "props": {
    "textColor": "default",
    "backgroundColor": "default",
    "textAlignment": "left"
  },
  "content": [{ "type": "text", "text": "Your paragraph text here.", "styles": {} }],
  "children": []
}
```

Rules:
- Each block needs a unique UUID-format `id`
- Split multi-paragraph text into one block per paragraph
- The entire `blocknote` value is a **JSON-stringified array** of blocks

> **Shortcut:** You can send plain text in a `body` field (instead of `bodyV2`) and the API will auto-convert it to the correct `bodyV2` format with both markdown and blocknote representations.

---

## System Fields (Read-Only)

These fields are automatically managed. **Never include them in create or update requests.**

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Unique record identifier |
| `createdAt` | DATE_TIME | Creation timestamp |
| `updatedAt` | DATE_TIME | Last update timestamp |
| `deletedAt` | DATE_TIME | Soft-delete timestamp |
| `position` | NUMBER | Display order |
| `createdBy` | ACTOR | Who/what created the record |
| `groupId` | UUID | Group identifier |
| `visibilityLevel` | NUMBER | Visibility level |
| `score` | NUMBER | Computed relevance score |
| `lastContactAt` | DATE_TIME | Last contact timestamp |

---

## Custom Fields

Each workspace can define custom fields for any entity. Custom field names are discovered via the [Metadata API](metadata.md) and used as JSON keys in the same way as standard fields:

```json
{
  "name": "Acme Corp",
  "myCustomTextField": "custom value",
  "myCustomSelectField": "OPTION_VALUE",
  "myCustomNumberField": 42
}
```

Custom fields follow the same type rules listed above.
