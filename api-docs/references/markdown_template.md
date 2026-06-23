# Markdown Template & Conventions

Structure every endpoint entry this way — consistency matters more than creativity here, since this is reference material people scan, not prose they read top to bottom.

```markdown
## POST /challenge/start

Creates a new challenge session with a random sequence of actions.

**Request body**: none

**Response** `200 OK`
\`\`\`json
{
  "session_id":  "uuid",
  "actions":     ["smile", "head_turn_left"],
  "next_action": "smile",
  "expires_at":  "2024-01-01T12:02:00Z"
}
\`\`\`

| Field         | Type     | Description                          |
|---------------|----------|---------------------------------------|
| session_id    | string   | UUID identifying this challenge session |
| actions       | string[] | Full ordered list of actions to complete |
| next_action   | string   | The action the client should perform first |
| expires_at    | string   | ISO 8601 timestamp; session is invalid after this |

**Errors**: none for this endpoint (always succeeds)
```

## For endpoints with errors, always include an errors table

```markdown
**Errors**

| Status | Condition                          | Body                                       |
|--------|-------------------------------------|---------------------------------------------|
| 404    | session_id not found                 | `{"error": "challenge: session not found"}` |
| 410    | session expired or already completed | `{"error": "challenge: session expired"}`   |
| 503    | face analyzer unreachable            | `{"error": "face analyzer unavailable"}`    |
```

## Document structure for the whole file

```markdown
# <API Name> API Reference

<one paragraph: what this API is for, base URL, auth if any>

## Authentication
<if applicable — otherwise omit this section entirely, don't write "N/A">

## Endpoints

### POST /challenge/start
...

### POST /challenge/verify
...

## Error format
<describe the consistent error envelope shape used across all endpoints, once, here — don't repeat the explanation under every endpoint>
```

## Writing conventions

- **Lead with what it does, not how it works internally.** "Creates a new challenge session" not "Calls store.Create() with a randomized action slice."
- **Always show a real example response**, not a schema description in prose. Developers copy-paste from examples; they don't parse prose.
- **Field tables, not bullet lists**, for request/response shapes — tables scan faster when someone's looking up a single field.
- **Note side effects inline** where they matter: "This advances the session to the next action" or "This call is idempotent and safe to retry."
- **Don't document the same error shape repeatedly** — describe the error envelope once in an "Error format" section, then just reference status codes per-endpoint.
- Keep prose minimal. This is reference material, not a tutorial — save explanatory walkthroughs for a separate guide if one is needed.
