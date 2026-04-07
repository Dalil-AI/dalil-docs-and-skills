# Authentication

All Dalil AI API requests must be authenticated with a Bearer token. This token identifies your workspace and authorizes access to its data.

## Getting Your API Key

1. Log in to your Dalil AI workspace at [app.usedalil.ai](https://app.usedalil.ai)
2. Navigate to **Settings → API Keys**
3. Generate a new API key
4. Copy and store it securely — **it will not be shown again**

## Sending the Auth Header

Include these two headers in every request:

```
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

### Example

```bash
curl -X GET "https://app.usedalil.ai/rest/people?limit=10" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json"
```

> **Tip:** For GET requests with complex filter parameters, use `curl -G` with `--data-urlencode` to avoid URL encoding issues:
> ```bash
> curl -G "https://app.usedalil.ai/rest/people" \
>   -H "Authorization: Bearer YOUR_API_KEY" \
>   --data-urlencode "filter=emails.primaryEmail[eq]:john@example.com"
> ```

## Base URL

```
https://app.usedalil.ai
```

- REST endpoints: `/rest/{resource}`
- GraphQL endpoint: `/graphql`

## Error Responses

| Status Code | Meaning | Common Cause |
|-------------|---------|--------------|
| `401` | Unauthorized | Missing or invalid API key |
| `403` | Forbidden | Key does not have permission for this resource |
| `404` | Not Found | Record ID does not exist |
| `429` | Too Many Requests | Rate limit exceeded — back off and retry |
| `500` | Internal Server Error | Unexpected server-side failure |

When you receive a `401`, check that:
- The `Authorization` header is present and spelled correctly
- The value starts with `Bearer ` (with a space)
- The API key has not been revoked in workspace settings
