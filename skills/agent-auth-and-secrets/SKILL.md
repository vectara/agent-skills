---
name: vectara-agent-auth-and-secrets
description: Configure OAuth, API-key, and service-account credentials on Vectara agent tools using agent.secrets, session.metadata, and $ref. Includes web_get native OAuth, dynamic tool addition, and the eager-ref resolution model.
tags: [vectara, agents, auth, oauth, secrets, web_get]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Agents — Auth, Secrets, and Dynamic Tools

You are helping a developer wire credentials into Vectara agent tools. Vectara's resolver substitutes `{"$ref": "<path>"}` values inside `argument_override` at session-create time, so a tool's URL, headers, and OAuth fields can be filled from three namespaces depending on the credential's scope.

## When to use this skill

- The agent calls an external API requiring OAuth, an API key, or a bearer token.
- Per-user credentials need to flow into specific chat sessions (e.g. each user's own Google Drive refresh token).
- Service-account credentials need to be shared across every session of an agent (e.g. a Wolken or ServiceNow integration).
- Tools need to be added or removed without recreating the agent.

If you only need a corpus search or a non-authenticated tool, use the `vectara-agents` skill instead.

## The three credential namespaces

| Namespace | Where it lives | Visibility | Typical use |
|---|---|---|---|
| `agent.secrets.*` | per-agent, encrypted at rest with the agent's KMS key | masked as `****` in events and on read | service-account credentials shared across all sessions |
| `session.metadata.*` | per-chat-session | in-the-clear (string) | per-user OAuth tokens, calling-user identity |
| `agent.metadata.*` | per-agent | in-the-clear (string) | non-secret per-agent config (URLs, IDs, scopes) |

`RefNode` classifies `agent.secrets.*` as SECRET (gets masked in `tool_input` events); the other two are STANDARD (visible in events). Don't put secrets in `session.metadata` or `agent.metadata` if event observability is in scope.

## The `$ref` pattern

Anywhere a tool's `argument_override` accepts a string, you can use `{"$ref": "<dot.path>"}` to substitute a value from one of the three namespaces. The resolver walks the JSON tree at session-create time and replaces every reference eagerly. References inside nested objects (like `auth.client_id`, `headers.Authorization`) resolve too — the resolver doesn't only look at top-level keys.

```json
"argument_override": {
  "headers": {
    "Authorization": { "$ref": "agent.secrets.MYAPI_BEARER" },
    "X-Tenant-Id":   { "$ref": "agent.metadata.tenant_id" }
  }
}
```

## `web_get` tool with native OAuth

The `web_get` tool supports five auth modes via its `auth` field. All credential subfields accept `$ref`:

| `auth.type` | What it does | Token caching |
|---|---|---|
| `none` | No auth header | n/a |
| `bearer` | Adds `Authorization: Bearer <token>` | static |
| `header` | Adds a custom header (e.g. `X-API-Key`) | static |
| `oauth_client_credentials` | RFC 6749 §4.4 — fetches access token via client_id/secret | platform-cached until expiry |
| `oauth_refresh_token` | RFC 6749 §6 — exchanges refresh_token for access_token | platform-cached, refreshed ~1 min before expiry |

The OAuth modes use RFC 6749's **HTTP Basic** scheme to authenticate the client to the token endpoint. If you're building a token endpoint to receive these requests, parse credentials from the `Authorization: Basic` header, not the body.

## Recipe 1 — Per-user OAuth refresh-token (Google Drive style)

User-specific refresh tokens flow through `session.metadata`. The wrapper app runs the OAuth dance, stores tokens per user, then injects them into `session.metadata` when creating each chat session. Vectara mints fresh access tokens automatically.

```json
"gdrive_read_doc": {
  "type": "web_get",
  "description_template": "Fetch a Google Drive file. URL is https://www.googleapis.com/drive/v3/files/<FILE_ID>/export?mimeType=text/plain for Google Docs, or /files/<FILE_ID>?alt=media for binaries. Authorization is filled by the platform; do not pass it yourself.",
  "argument_override": {
    "method": "GET",
    "auth": {
      "type": "oauth_refresh_token",
      "client_id":     { "$ref": "session.metadata.GOOGLE_CLIENT_ID" },
      "client_secret": { "$ref": "session.metadata.GOOGLE_CLIENT_SECRET" },
      "refresh_token": { "$ref": "session.metadata.GOOGLE_REFRESH_TOKEN" },
      "token_endpoint": "https://oauth2.googleapis.com/token"
    }
  }
}
```

**Session-create payload:**

```bash
POST /v2/agents/{key}/sessions
{
  "metadata": {
    "GOOGLE_CLIENT_ID":     "1234-abc.apps.googleusercontent.com",
    "GOOGLE_CLIENT_SECRET": "GOCSPX-...",
    "GOOGLE_REFRESH_TOKEN": "1//0g..."
  }
}
```

**Note:** `auth.token_endpoint` is typed strictly as `string` (no `$ref` variant) — hardcode it.

## Recipe 2 — Per-user static bearer (per-header `$ref`)

GitHub-style: the wrapper exchanges OAuth code for a non-expiring bearer, stores it as the full `Authorization` header value, and injects via per-header `$ref`. No refresh-token machinery needed.

```json
"github_create_issue": {
  "type": "web_get",
  "argument_override": {
    "method": "POST",
    "url":    { "$ref": "session.metadata.GITHUB_ISSUES_URL" },
    "headers": {
      "Authorization": { "$ref": "session.metadata.GITHUB_BEARER_HEADER" },
      "Accept":        "application/vnd.github+json",
      "Content-Type":  "application/json"
    }
  }
}
```

Pre-build the full URL in `session.metadata` (here `GITHUB_ISSUES_URL = "https://api.github.com/repos/<owner>/<repo>/issues"`) so the LLM never has to construct it.

## Recipe 3 — Service-account OAuth refresh-token (Wolken-style)

One credential set on the agent itself, shared across every session. Admin clicks "provision" once.

```json
"create_ticket": {
  "type": "web_get",
  "argument_override": {
    "method": "POST",
    "url":   { "$ref": "agent.metadata.TICKETS_API_URL" },
    "auth": {
      "type": "oauth_refresh_token",
      "client_id":     { "$ref": "agent.secrets.TICKETS_CLIENT_ID" },
      "client_secret": { "$ref": "agent.secrets.TICKETS_CLIENT_SECRET" },
      "refresh_token": { "$ref": "agent.secrets.TICKETS_REFRESH_TOKEN" },
      "token_endpoint": "https://tickets.example.com/oauth/token"
    },
    "headers": { "Content-Type": "application/json" }
  }
}
```

**Provision agent secrets:**

```bash
PATCH /v2/agents/{key}/secrets
{
  "secrets": {
    "TICKETS_CLIENT_ID":     "...",
    "TICKETS_CLIENT_SECRET": "...",
    "TICKETS_REFRESH_TOKEN": "...",
    "TICKETS_PROVISIONED":   "true"
  }
}
```

`TICKETS_PROVISIONED` is a convenience marker — read it back via `GET /v2/agents/{key}/secrets` to know whether the SA is configured (since all values are masked as `****` on read).

## Eager-ref resolution — and sentinels

Vectara resolves every `$ref` in `argument_override` at session-create time. If a referenced key doesn't exist in its namespace, the call returns **HTTP 422** with `Failed to resolve eager references: tool '<name>': unresolved refs [...]`. The agent loop never starts.

Two ways to handle this gracefully:

1. **Pre-populate placeholder values** when creating the agent. Right after `POST /v2/agents`, `PATCH /v2/agents/{key}/secrets` with sentinel strings so `agent.secrets.*` refs always resolve. The runtime OAuth call fails 401 if values are still placeholders, which the LLM can surface to the user.
2. **Always emit all expected keys in `session.metadata`** at session-create — populate real values when an integration is connected, sentinel strings (e.g. `"NOT_CONNECTED"`) otherwise.

```bash
# Run once after agent creation:
PATCH /v2/agents/{key}/secrets
{
  "secrets": {
    "MYAPI_CLIENT_ID":     "NOT_PROVISIONED_YET",
    "MYAPI_CLIENT_SECRET": "NOT_PROVISIONED_YET",
    "MYAPI_REFRESH_TOKEN": "NOT_PROVISIONED_YET"
  }
}
```

## Dynamic tool management

Add a new tool to an existing agent with a single PATCH — no recreate needed:

```bash
PATCH /v2/agents/{key}
{
  "tool_configurations": {
    "github_get_issue": {
      "type": "web_get",
      "description_template": "Fetch a GitHub issue. Call with url = https://api.github.com/repos/<owner>/<repo>/issues/<number>. Authorization is injected from session.metadata.GITHUB_BEARER_HEADER.",
      "argument_override": {
        "method": "GET",
        "headers": {
          "Authorization": { "$ref": "session.metadata.GITHUB_BEARER_HEADER" },
          "Accept":        "application/vnd.github+json"
        }
      }
    }
  }
}
```

PATCH merges into `tool_configurations`; existing tools are untouched. **New chat sessions** see the new tool immediately — existing sessions resolved their refs at session-create, so re-create the session to pick up the new tool. Remove a tool by sending `null` for its key.

## Agent secrets API surface

| Endpoint | Body | Behavior |
|---|---|---|
| `GET /v2/agents/{key}/secrets` | — | returns `{secrets: {name: "****", ...}}` — values always masked |
| `PUT /v2/agents/{key}/secrets` | `{secrets: {...}}` | replaces the entire secrets map |
| `PATCH /v2/agents/{key}/secrets` | `{secrets: {...}}` | merge: present-with-value upserts, present-with-null deletes, absent leaves untouched |

Stored encrypted at rest with the agent's `agent_ekey_id`. Values redacted in every `tool_input` event when resolved from an `agent.secrets.*` `$ref`. Don't store the same value elsewhere on the agent (e.g. in `metadata`) — that bypasses the redaction.

## Why credentials flow through `session.metadata` vs `agent.secrets`

| Question | If yes → use `agent.secrets` | If yes → use `session.metadata` |
|---|---|---|
| Same value for every user of the agent? | ✅ | |
| Different per end-user? | | ✅ |
| Sensitive enough to need event masking? | ✅ | (use `session.secrets` when shipped) |
| Set by an admin once? | ✅ | |
| Set by your wrapper app per chat session? | | ✅ |

> **Roadmap note:** `session.secrets.*` — same crypto + redaction as `agent.secrets` but scoped per session — is a planned platform release. Once shipped, replace `$ref: session.metadata.X` with `$ref: session.secrets.X` for per-user credentials and get event masking back.

## Common mistakes to avoid

- Putting a `$ref` on `auth.token_endpoint`. Schema typed strictly as `string`. Hardcode the token URL.
- Referencing a secret/metadata key that hasn't been set yet — session creation 422s on eager-ref failure. Pre-PATCH placeholders.
- Sending OAuth client credentials in the request body of a `web_get` token endpoint. Vectara's `OAuth2Client` sends them via `Authorization: Basic` per RFC 6749 §2.3.1.
- Treating `session.metadata` as encrypted. It isn't — values appear in `tool_input` events. Use `agent.secrets` for actual secrets when possible.
- Adding a tool via PATCH and expecting an existing session to see it. Sessions resolve refs at create time. Re-create the session.
- Pinning a header with the **nested form** — `argument_override: {"headers": {"Authorization": "..."}}` — and expecting the agent to still set other headers like `Accept` or `Content-Type`. **The whole `headers` property is hidden from the agent's tool schema** in that case (top-level override keys are stripped from the schema), so the agent literally cannot emit `headers` at all and only your one pinned header survives. To pin one header while leaving siblings open to the agent, use a **dot-notation key** at the override top level:
  ```json
  "argument_override": {
    "headers.Authorization": { "$ref": "agent.secrets.api_key" }
  }
  ```
  Only `headers.Authorization` is hidden from the schema; the agent can still set `headers.Accept`, `headers.Content-Type`, etc. The same trick works for any nested path (`body.api_key`, `auth.refresh_token`, etc.).
- Expecting `argument_override.url` to support template substitution. The URL string is passed verbatim to the HTTP client — `{var}` syntax produces a request with a literal `{var}` in the path, which usually 404s at the upstream. For variable URLs, pin everything *except* `url` and use `description_template` to tell the agent the URL pattern to construct.
- Putting structured user-supplied fields like `requestor_email` into `web_get.body`. The body is a string the LLM constructs; for structured-arg tools use `mcp` or `dynamic_vectara` tool types.

## Working example

A runnable end-to-end demo of all three patterns lives at:
- **Repo:** [vectara/toolkits-auth-demo](https://github.com/vectara/toolkits-auth-demo) — Drive (per-user OAuth refresh), GitHub (per-user static bearer), MockTickets (service-account)
- **API tour page:** the `/apis` route in that demo walks through `POST /v2/agents` → `PATCH /v2/agents/{key}/secrets` → `POST /v2/agents/{key}/sessions` → `POST /v2/agents/{key}/sessions/{id}/events` → `PATCH /v2/agents/{key}` (dynamic tool add) with live request/response panes against any tenant.

## References

- [Tools](https://docs.vectara.com/docs/agents/tools.md)
- [Sessions](https://docs.vectara.com/docs/agents/sessions.md)
- [OpenAPI Spec](https://api.vectara.io/v2/openapi.json) — `web_get`, `agent_secrets`, `agents` schemas
