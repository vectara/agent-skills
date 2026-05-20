---
name: vectara-agents
description: Build AI agents with the Vectara platform — create agents, configure tools, manage sessions, and search corpora.
tags: [vectara, agents, rag, ai, api]
compatibility: [claude-code, cursor, copilot, cline, windsurf]
---

# Vectara Agent Skills

You are helping a developer build with the Vectara platform. Vectara is an enterprise agentic platform for building trusted AI agents with retrieval augmented generation (RAG).

## See also

This skill covers basic agent creation, corpora search, and the 3-step chat flow. For anything beyond that, the sibling skills are more specific:

- **`vectara-agent-auth-and-secrets`** — OAuth, API keys, service-account credentials, `web_get` native auth modes, `$ref` to `agent.secrets` / `session.metadata` / `agent.metadata`, dynamic tool management via `PATCH /v2/agents/{key}`. Use whenever a tool calls an external API that needs credentials.
- **`vectara-agent-orchestration`** — multi-step state machines (`first_step_name` + `steps{}` map + `next_steps`), structured-output gating, sub-agents, runtime-constrained generation, Slack/connector-driven workflows, two-agent gates for human-in-the-loop approvals. Use for anything more complex than a single conversational loop.

## API Reference

The full OpenAPI spec is available at `https://api.vectara.io/v2/openapi.json`. Fetch it when you need exact request/response schemas.

## Documentation Access

You can fetch any Vectara documentation page as markdown by appending `.md` to its URL. For example: `https://docs.vectara.com/docs/agents/agents.md`. The full documentation index is at `https://docs.vectara.com/llms.txt`. When you need detailed information about a Vectara feature, fetch the relevant markdown page directly.

## Critical Rules

1. **API base URL** is `https://api.vectara.io/v2/`.
2. **Authentication**: Use either `x-api-key` header with an API key, or `Authorization: Bearer <token>` with an OAuth token. Both are valid — use whichever the developer has. Never use both simultaneously.
3. **Agents are a first-class resource.** Create them via `POST /v2/agents`, not by stitching together query calls manually.
4. **Every agent needs four things:**
   - `tool_configurations` — at least one tool (corpora_search, web_search, or custom)
   - `steps` — a *map* of named step configs, each with `instructions` and `output_parser`
   - `first_step_name` — the entry-point key into `steps` (typically `"main"`)
   - `model` — the LLM for reasoning (e.g., `claude-sonnet-4-6`, `gpt-4o`, `gemini-2-5-flash-vertex`)

   The older top-level `first_step` field is deprecated — use `first_step_name` + `steps{}` even for single-step agents.
5. **Sessions track conversations.** Create a session before sending messages: `POST /v2/agents/{key}/sessions`
6. **Messages are events.** Send user input via `POST /v2/agents/{key}/sessions/{session_key}/events` with `type: "input_message"`
7. **Never hardcode corpus IDs.** Use `corpus_key` (string) not `corpus_id` (number).

## Agent Creation Pattern

```json
{
  "name": "my-agent",
  "description": "Agent description",
  "model": {
    "name": "claude-sonnet-4-6",
    "parameters": {"temperature": 0.3, "max_tokens": 1000}
  },
  "tool_configurations": {
    "search": {
      "type": "corpora_search",
      "query_configuration": {
        "search": {
          "corpora": [{"corpus_key": "my-corpus"}],
          "limit": 10
        }
      }
    }
  },
  "first_step_name": "main",
  "steps": {
    "main": {
      "instructions": [{
        "type": "inline",
        "name": "My Instructions",
        "template": "You are a helpful assistant. Use the search tool to find relevant information. Cite your sources."
      }],
      "output_parser": {"type": "default"}
    }
  }
}
```

For multi-step workflows, add more entries to `steps` and use `next_steps` on each step to declare transitions — see `vectara-agent-orchestration`.

## Available Tool Types

| Tool Type | `type` value | Purpose |
|-----------|-------------|---------|
| Corpus Search | `corpora_search` | Search indexed Vectara corpora |
| Web Search | `web_search` | Search the internet |
| Web Get | `web_get` | Fetch a URL with native OAuth/bearer/header auth |
| Sub-agent | `sub_agent` | Delegate to another agent |
| Lambda | `lambda` | Run custom Python functions |
| Document Conversion | built-in | Convert PDF/DOCX/PPTX to markdown |
| Structured Indexing | built-in | Index a converted artifact into a corpus |
| MCP | `mcp` | External tool servers via Model Context Protocol |
| Dynamic Vectara | `dynamic_vectara` | Vectara-shipped built-in (e.g. `tol_vectara_slack_post_message`) |

## Discovering Available Tools

The table above lists the *kinds* of tools you can configure. To enumerate the actual tool instances registered to your account — discover `tol_*` IDs, inspect input schemas before wiring a tool, see which MCP servers are connected — use the discovery endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /v2/tools` | List all tools available to the authenticated user — built-ins, `dynamic_vectara`, MCP, and Lambda |
| `GET /v2/tools/{tool_id}` | Full definition of one tool, including its JSON Schema `input_schema` |
| `GET /v2/tool_servers` | List registered MCP tool servers |
| `GET /v2/tool_servers/{tool_server_id}` | Server details |
| `POST /v2/tool_servers/{tool_server_id}/sync` | Re-discover tools exposed by a server |

`GET /v2/tools` query params: `filter` (regex on name+description), `type`, `enabled` (bool), `category` (array — `experimental` is excluded by default, pass it explicitly to include), `tool_server_id`, plus pagination.

Each tool returns: `id` (`tol_...` — use as `tool_id` in `tool_configurations`), `name`, `description`, `input_schema`, `type`, `enabled`, `category`.

```bash
# Browse every tool registered to the account
curl -s -H "x-api-key: $API_KEY" \
  "https://api.vectara.io/v2/tools" \
  | jq '.tools[] | {id, name, type, description}'

# Inspect a tool's input schema before wiring it into an agent
curl -s -H "x-api-key: $API_KEY" \
  "https://api.vectara.io/v2/tools/tol_abc123" \
  | jq '{name, input_schema}'

# After registering or updating an MCP server, sync to refresh the tool list
curl -s -X POST -H "x-api-key: $API_KEY" \
  "https://api.vectara.io/v2/tool_servers/$SERVER_ID/sync"

curl -s -H "x-api-key: $API_KEY" \
  "https://api.vectara.io/v2/tools?tool_server_id=$SERVER_ID" \
  | jq '.tools[] | .name'
```

## Corpus Search Tool Configuration

```json
{
  "type": "corpora_search",
  "query_configuration": {
    "search": {
      "corpora": [
        {
          "corpus_key": "my-corpus",
          "lexical_interpolation": 0.01
        }
      ],
      "limit": 10,
      "context_configuration": {
        "sentences_before": 2,
        "sentences_after": 2
      },
      "reranker": {
        "type": "customer_reranker",
        "reranker_id": "rnk_272725719"
      }
    }
  }
}
```

## Web Search Tool Configuration

```json
{
  "type": "web_search",
  "argument_override": {
    "limit": 10
  }
}
```

## Creating Lambda Tools

Lambda tools are custom Python functions that agents can call. They run in a sandboxed Python 3.12 runtime — no network, no filesystem writes, no `pip install`, no system commands. Allowed stdlib modules: `json`, `math`, `datetime`, `collections`, `itertools`, `functools`, `re`, `time`, `typing`.

**Two rules that make the rest work:**

1. The entry point function MUST be named `process`. Anything else 422s.
2. Input and output schemas are *auto-discovered from Python type annotations*. Parameters with default values become optional. The return must be a JSON-serializable dict.

```python
def process(order_count: int, total_revenue: float, days_active: int = 1) -> dict:
    score = (order_count * 10 + total_revenue * 0.1) / days_active
    tier = 'gold' if score > 20 else 'silver' if score > 10 else 'bronze'
    return {'score': round(score, 2), 'tier': tier}
```

### Create the tool

```bash
curl -X POST https://api.vectara.io/v2/tools \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "lambda",
    "language": "python",
    "name": "customer_score_calculator",
    "title": "Customer Score Calculator",
    "description": "Calculates a customer score from order count, revenue, and days active. Returns score and tier (bronze/silver/gold).",
    "code": "def process(order_count: int, total_revenue: float, days_active: int = 1) -> dict:\n    score = (order_count * 10 + total_revenue * 0.1) / days_active\n    tier = '"'"'gold'"'"' if score > 20 else '"'"'silver'"'"' if score > 10 else '"'"'bronze'"'"'\n    return {'"'"'score'"'"': round(score, 2), '"'"'tier'"'"': tier}",
    "execution_configuration": {
      "max_execution_time_seconds": 30,
      "max_memory_mb": 100
    }
  }'
```

Response includes the assigned `id` (`tol_*`), plus the auto-discovered `input_schema` and `output_schema`. `execution_configuration` limits: `max_execution_time_seconds` 1–300 (default 30), `max_memory_mb` 10–1024 (default 100).

### Test before creating

`POST /v2/tools/test` validates the code, discovers the schema, and runs it against sample input — without persisting the tool. Use this to iterate before committing to a `tol_*` ID.

```bash
curl -X POST https://api.vectara.io/v2/tools/test \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "language": "python",
    "code": "def process(order_count: int, total_revenue: float) -> dict:\n    return {\"score\": order_count * 10 + total_revenue * 0.1}",
    "test_input": {"order_count": 10, "total_revenue": 500}
  }'
```

Required body fields: `code`, `test_input`. Response includes `validation.status` (`valid`/`invalid` + errors), the discovered `input_schema`/`output_schema`, and `execution.output` / `latency_millis` / `memory_used_mb`.

### Test a deployed tool

```bash
curl -X POST https://api.vectara.io/v2/tools/tol_abc123/test \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"input": {"order_count": 50, "total_revenue": 5000, "days_active": 30}}'
```

### Update, disable, delete

`PATCH /v2/tools/{tool_id}` to update code or metadata (schemas re-discovered on update). Disable by patching `{"type": "lambda", "enabled": false}`. `DELETE /v2/tools/{tool_id}` removes it.

### Input-schema gotchas

- **Object parameters must use `TypedDict`** — bare `dict` and `Dict[K, V]` annotations are rejected, because the platform can't infer a JSON Schema from them.
- **Bare `list` parameters break the agent loop with a schema error.** Take a `str` and `json.loads` it inside the function, or use a typed `List[T]`. (See `vectara-agent-orchestration` for context.)
- Default values make a parameter optional; without a default it's required.

### Wire the lambda into an agent

```json
{
  "type": "lambda",
  "tool_id": "tol_abc123",
  "argument_override": {
    "customer_tier": "enterprise",
    "query": { "$ref": "session.metadata.search_query" }
  }
}
```

`argument_override` locks fields the LLM shouldn't control and can pull dynamic values via `$ref` (see `vectara-agent-auth-and-secrets`).

For reusable, governed wiring across agents, create a `LambdaToolConfiguration` once via `POST /v2/tools/{tool_id}/configurations` (returns a `tcf_*` id) and reference it from agents with `{"type": "lambda", "configuration_id": "tcf_..."}`.

## Artifacts

Artifacts are session-scoped files in an agent's workspace — PDFs, DOCX, PPTX, and images (PNG, JPEG, GIF, WebP). They are referenced by `artifact_id`, not embedded in events. Files persist for the life of the session without being indexed into a corpus. Users can upload artifacts and agents can generate new ones. Use artifacts when a user needs an agent to read or produce a file on the fly; use a corpus when the content should be permanently indexed and searchable.

### Artifact tool taxonomy

The built-in artifact tools are configured by `type` alone — no `tool_id`, no `argument_override` required.

| `type` value | Purpose |
|--------------|---------|
| `artifact_read` | Read text content from artifacts (PDF, DOCX, TXT) |
| `image_read` | Analyze visual artifacts (PNG, JPEG) |
| `document_conversion` | Convert between formats (e.g. PDF to markdown) |
| `artifact_grep` | Search for patterns within artifact content |
| `artifact_create` | Generate a new artifact (report, summary, code) and attach it to the session |

Wire them into `tool_configurations` the same way as any other tool:

```json
"tool_configurations": {
  "artifact_read":        {"type": "artifact_read"},
  "image_read":           {"type": "image_read"},
  "document_conversion":  {"type": "document_conversion"},
  "artifact_grep":        {"type": "artifact_grep"}
}
```

### Upload flow

Uploads go through the **events** endpoint as `multipart/form-data` — not embedded in a JSON body. The platform creates an `artifact_upload` event in the session.

```bash
curl -X POST "https://api.vectara.io/v2/agents/$AGENT_KEY/sessions/$SESSION_KEY/events" \
  -H "x-api-key: $API_KEY" \
  -F "files=@/tmp/q3_sales_report.pdf;type=application/pdf" \
  -F "stream_response=false"
```

Do **not** set `Content-Type: application/json` on upload requests — the HTTP client must set the multipart boundary automatically. The response contains an `events[]` array; the upload event has `type: "artifact_upload"` and an `artifacts[]` list, each with an `artifact_id` you can reference later.

### Listing and fetching artifacts

```
GET  /v2/agents/{agent_key}/sessions/{session_key}/artifacts
GET  /v2/agents/{agent_key}/sessions/{session_key}/artifacts/{artifact_id}
```

Call list again after a chat turn to see artifacts the agent created via `artifact_create`.

### End-to-end: analyze an uploaded PDF

```bash
# 1. Create agent with artifact tools wired in
AGENT_KEY=$(curl -s -X POST https://api.vectara.io/v2/agents \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Document Analyst",
    "model": {"name": "claude-sonnet-4-6"},
    "first_step_name": "main",
    "steps": {"main": {
      "instructions": [{"type": "inline", "name": "doc",
        "template": "You analyze uploaded documents. Cite specific sections."}],
      "output_parser": {"type": "default"}
    }},
    "tool_configurations": {
      "document_conversion": {"type": "document_conversion"},
      "artifact_read":       {"type": "artifact_read"},
      "artifact_grep":       {"type": "artifact_grep"}
    }
  }' | jq -r '.key')

# 2. Create a session
SESSION_KEY=$(curl -s -X POST https://api.vectara.io/v2/agents/$AGENT_KEY/sessions \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"name": "pdf-review"}' | jq -r '.key')

# 3. Upload the PDF (multipart, no Content-Type header)
curl -s -X POST "https://api.vectara.io/v2/agents/$AGENT_KEY/sessions/$SESSION_KEY/events" \
  -H "x-api-key: $API_KEY" \
  -F "files=@./q3_sales_report.pdf;type=application/pdf" \
  -F "stream_response=false"

# 4. Ask the agent — it picks document_conversion / artifact_grep itself
curl -s -X POST "https://api.vectara.io/v2/agents/$AGENT_KEY/sessions/$SESSION_KEY/events" \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"type": "input_message",
       "messages": [{"type": "text",
                     "content": "Grep the uploaded report for revenue figures and summarize Q4 outlook."}]}'
```

### Artifacts — common mistakes

- Trying to upload by base64-encoding the file into the JSON `input_message` body. Uploads must be `multipart/form-data` on the events endpoint with a `files=` part.
- Setting `Content-Type: application/json` (or any explicit `Content-Type`) on the upload request — that breaks the multipart boundary. Pass only `x-api-key` and let the HTTP client set the type.
- Wiring `artifact_read` but expecting it to handle PDFs directly. PDFs flow through `document_conversion` first; without it, the agent can't extract text from a PDF artifact.
- Treating artifacts like corpus documents. They live only in the session — once the session is gone, so are they. Index into a corpus if you need persistence or cross-session search.
- Looking for the new `artifact_id` in the message response after asking the agent to create one. Re-call `GET .../sessions/{session_key}/artifacts` to see agent-generated artifacts.

## Agent Schedules

Schedules run an agent automatically on a recurring cadence — cron-style or fixed interval — without a user in the loop. Each execution spins up a fresh session, replays the schedule's initial `message`, and records the result in an execution history you can poll. Use them for periodic digests, monitoring sweeps, scheduled data processing, and alerting workflows. Schedules are scoped to an agent: `POST /v2/agents/{agent_key}/schedules`.

### Schedule shape

Two timing types. Cron uses standard 5-field expressions (UTC); interval uses ISO-8601 duration (minimum `PT1H`).

```json
{
  "name": "Daily Research Digest",
  "description": "Generates a research digest every weekday morning",
  "message": [
    {"type": "text", "content": "Generate today's research digest..."}
  ],
  "schedule": {
    "type": "cron",
    "cron_expression": "0 9 * * 1-5"
  },
  "enabled": false,
  "session_metadata": {
    "report_type": "daily_digest",
    "automated": "true"
  },
  "max_executions_to_keep": 20
}
```

Interval variant — swap the `schedule` block:

```json
"schedule": {
  "type": "interval",
  "interval": "PT6H"
}
```

| Field | Purpose |
|-------|---------|
| `name`, `description` | Human label and notes |
| `message` | Initial input replayed into each auto-created session (same shape as `input_message.messages`) |
| `schedule.type` | `cron` or `interval` |
| `schedule.cron_expression` | 5 fields, UTC: minute, hour, day-of-month, month, day-of-week (Sun=0) |
| `schedule.interval` | ISO-8601 duration, e.g. `PT1H`, `PT6H`, `PT24H` |
| `enabled` | Boolean. Create with `false`, flip to `true` via `PATCH` once verified |
| `session_metadata` | Free-form string map forwarded onto each scheduled session — readable by steps via `$ref` to `session.metadata.*` |
| `max_executions_to_keep` | Retention cap on execution history rows |

### CRUD reference

| Endpoint | Purpose |
|----------|---------|
| `POST /v2/agents/{agent_key}/schedules` | Create a schedule (201 on success, body contains `key`) |
| `GET  /v2/agents/{agent_key}/schedules` | List schedules — response `{"schedules": [...]}` |
| `GET  /v2/agents/{agent_key}/schedules/{schedule_key}` | Get one |
| `PATCH /v2/agents/{agent_key}/schedules/{schedule_key}` | Partial update — send only changed fields |
| `DELETE /v2/agents/{agent_key}/schedules/{schedule_key}` | Delete (204) |
| `GET  /v2/agents/{agent_key}/schedules/{schedule_key}/executions` | Execution history — response `{"executions": [...]}` |

### Dev workflow: create disabled, then enable

Always `POST` with `"enabled": false`, verify the config in the list/get response, then `PATCH {"enabled": true}` once you're ready. This prevents a misconfigured schedule from firing immediately (interval schedules can fire on the first tick) and creates a clean two-step audit trail.

```bash
# Create disabled
SCHEDULE_KEY=$(curl -s -X POST \
  https://api.vectara.io/v2/agents/$AGENT_KEY/schedules \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"...","message":[...],"schedule":{"type":"cron","cron_expression":"0 9 * * 1-5"},"enabled":false}' \
  | jq -r '.key')

# Enable when ready — PATCH carries only what's changing
curl -X PATCH https://api.vectara.io/v2/agents/$AGENT_KEY/schedules/$SCHEDULE_KEY \
  -H "x-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"enabled": true}'
```

### Polling executions

Each execution entry includes: `workflow_run_id`, `attempt`, `status`, `session_key`, `executed_at`, and `error_message` (populated when the run failed). Pull the `session_key` and `GET .../sessions/{session_key}/events` to inspect the actual `input_message`, `tool_input`/`tool_output`, and `agent_output` the schedule produced. An execution can exist with no `session_key` if the run failed before session creation — check `error_message` in that case.

### `session_metadata` forwarding

Whatever you set in the schedule's `session_metadata` object lands on every session the schedule creates. Steps and tool `argument_override` blocks can then pull values via `$ref` to `session.metadata.<key>` — the same mechanism normal sessions use. Values are strings.

### Schedules — common mistakes

- **Creating with `enabled: true` while iterating.** An interval schedule can fire before you've confirmed the config is right. Always create disabled, then `PATCH` to enable.
- **Using interval shorter than `PT1H`.** That's the platform minimum for `schedule.interval`. Use cron for sub-hourly cadences only if your plan supports it.
- **Forgetting cron is UTC.** `0 9 * * 1-5` is 9 AM UTC weekdays, not 9 AM local time. Convert before writing the expression.
- **Sending a full object on `PATCH`.** Only include fields you're changing.
- **Looking for the agent's reply on the execution row.** The execution record carries run metadata only; the agent's actual output lives on the auto-created session — fetch it via `GET .../sessions/{session_key}/events`.

## Chat Flow (3 Steps)

```bash
# 1. Create agent
AGENT_KEY=$(curl -s -X POST https://api.vectara.io/v2/agents \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ ... }' | jq -r '.key')

# 2. Create session
SESSION_KEY=$(curl -s -X POST https://api.vectara.io/v2/agents/$AGENT_KEY/sessions \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-session"}' | jq -r '.key')

# 3. Send message
curl -s -X POST https://api.vectara.io/v2/agents/$AGENT_KEY/sessions/$SESSION_KEY/events \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "input_message",
    "messages": [{"type": "text", "content": "Your question here"}]
  }'
```

## Response Events

When you POST to `/events`, the response (or SSE stream, with `stream_response: true`) is a list of events. Walk them to surface what the agent did — only `agent_output` / `structured_output` are terminal; everything else is mid-turn context you'd typically log or render in a UI trace.

### Event type catalog

| Event `type` | When | Key fields |
|---|---|---|
| `input_message` | Echo of the user's input | `content` |
| `thinking` | Mid-turn reasoning chunk | `content` |
| `tool_input` | The agent decided to call a tool | `tool_configuration_name`, `content` (the input args the LLM constructed) |
| `tool_output` | Tool returned | `tool_configuration_name`, `content`. Corpus search results live at `tool_output.search_results` (not `.results`). For `web_get`, `content.status_code` and `content.content` carry the response body |
| `agent_output` | Terminal free-form reply (steps with `output_parser: {"type": "default"}`) | `content` |
| `structured_output` | Terminal structured reply (steps with `output_parser: {"type": "structured"}`) | `step_name`, `schema_name`, `content` (the parsed dict) |
| `step_transition` | Step-machine transition fired | `from_step`, `to_step` |
| `step_transition_limit_exceeded` | Safety limit tripped — a turn chained more than 500 transitions | `count` |
| `skill_load` | A skill's `content` was injected into context (either model-driven via `invoke_skill` or client-driven via `{"type":"skill"}` input) | `skill_name` |
| `artifact_upload` | Multipart file was attached to the session via `POST /events` | `artifacts[]` (each has `artifact_id`) |
| `streaming_agent_output` | Per-chunk free-form text while streaming. Only with `stream_response: true` | `content` |
| `streaming_agent_output_end` | End-of-stream marker | — |

For multi-step agents that mix structured and default output parsers, walk every event and take the **last** terminal one (`agent_output` or `structured_output`) — intermediate `structured_output` events from upstream steps are not the final reply.

### Streaming

Set `stream_response: true` in the request body and consume the response as Server-Sent Events. Use a client like `sseclient` in Python:

```python
import sseclient, requests
resp = requests.post(url, headers=headers, json={"stream_response": True, "messages": [...]}, stream=True)
for event in sseclient.SSEClient(resp).events():
    data = json.loads(event.data)
    if data.get("type") == "streaming_agent_output":
        print(data["content"], end="", flush=True)
    elif data.get("type") == "streaming_agent_output_end":
        break
```

## Corpus Search Generation

By default, the corpora_search tool has `generation.enabled: false`. To get RAG summaries in tool output, enable it explicitly:

```json
{
  "type": "corpora_search",
  "query_configuration": {
    "search": {
      "corpora": [{"corpus_key": "my-corpus"}],
      "limit": 10
    },
    "generation": {
      "enabled": true,
      "max_used_search_results": 5
    }
  }
}
```

## Idempotent (re)create pattern for agents and tools

DELETE on `/v2/agents/{key}` and `/v2/tools/{id}` is asynchronous — a `POST /v2/agents` right after the DELETE can race with the in-flight cleanup and return `409 Conflict` with "already exists" or "being deleted". The notebook examples wrap (re)creation in a small helper that absorbs the race:

```python
import time, requests

_TRANSIENT = (502, 503, 504)

def _list_agents_by_name(base_url, headers, name):
    page_key = None
    while True:
        params = {"limit": 100, **({"page_key": page_key} if page_key else {})}
        r = requests.get(f"{base_url}/agents", headers=headers, params=params)
        r.raise_for_status()
        data = r.json()
        for a in data.get("agents", []):
            if a.get("name") == name:
                yield a
        page_key = data.get("metadata", {}).get("page_key")
        if not page_key:
            return

def delete_and_create_agent(base_url, headers, config, name, max_attempts=4):
    for existing in _list_agents_by_name(base_url, headers, name):
        requests.delete(f"{base_url}/agents/{existing['key']}", headers=headers)
    for attempt in range(max_attempts):
        r = requests.post(f"{base_url}/agents", headers=headers, json=config)
        if r.status_code == 201:
            return r.json()["key"]
        if r.status_code == 409 and attempt < max_attempts - 1:
            for e in _list_agents_by_name(base_url, headers, name):
                requests.delete(f"{base_url}/agents/{e['key']}", headers=headers)
            time.sleep(2 ** attempt)
            continue
        if r.status_code in _TRANSIENT and attempt < max_attempts - 1:
            time.sleep(2 ** attempt)
            continue
        r.raise_for_status()
    raise RuntimeError(f"Failed after {max_attempts} attempts")
```

Two more gotchas this same pattern needs to cover when deleting **tools**:
- The list endpoint is paginated — page through all results when matching by name; don't assume one page.
- A tool DELETE fails with "tool is being used in different configurations" if any agent still references it. **Delete the referring agents first**, then poll until they're gone (re-list by name returns empty), then delete the tool.

```python
def wait_for_agent_gone(base_url, headers, name, timeout=30, poll=2):
    deadline = time.time() + timeout
    while time.time() < deadline:
        if not any(_list_agents_by_name(base_url, headers, name)):
            return True
        time.sleep(poll)
    return False
```

The full reference implementation (with the same retry posture for bulk corpus-document delete) is at [`vectara_utils.py`](https://github.com/vectara/example-notebooks/blob/main/notebooks/api-examples/vectara_utils.py).

## Common Mistakes to Avoid

- Using `corpus_id` (number) instead of `corpus_key` (string)
- Using the deprecated top-level `first_step` field instead of `first_step_name` + `steps{}` map
- Forgetting `output_parser: {"type": "default"}` on the entry step
- Pinning a single header in `argument_override.headers` and expecting other headers the LLM sets to be merged in — the dict is **replace, not merge**. Pin every header you care about together, or leave the field unset
- Sending messages without creating a session first
- Putting `limit` inside `corpora[]` instead of in `search{}`
- Looking for `tool_output.results` instead of `tool_output.search_results` for corpus search
- Not enabling `generation.enabled: true` in query_configuration when you want RAG summaries

## References

For detailed documentation, fetch these pages:

### Agents
- [Agent Overview](https://docs.vectara.com/docs/agents/agents.md)
- [Agent Quickstart](https://docs.vectara.com/docs/agents/agents-quickstart.md)
- [Create Agent Examples](https://docs.vectara.com/docs/agents/create-agent-examples.md)
- [Tools](https://docs.vectara.com/docs/agents/tools.md)
- [Built-in Tools](https://docs.vectara.com/docs/agents/agent-tools.md)
- [Custom Tools](https://docs.vectara.com/docs/agents/custom-tools.md)
- [Lambda Tools](https://docs.vectara.com/docs/agents/lambda-tools.md)
- [Instructions](https://docs.vectara.com/docs/agents/instructions.md)
- [Sessions](https://docs.vectara.com/docs/agents/sessions.md)
- [Sub-agents](https://docs.vectara.com/docs/agents/subagents.md)
- [MCP](https://docs.vectara.com/docs/agents/model-context-protocol.md)
- [Tool Servers](https://docs.vectara.com/docs/agents/tool-servers.md)
- [Create Agent API](https://docs.vectara.com/docs/api-reference/agent-apis/create-agent.md)

### Tool Discovery
- [List tools](https://docs.vectara.com/docs/rest-api/list-tools.md)
- [Get tool](https://docs.vectara.com/docs/rest-api/get-tool.md)
- [List tool servers](https://docs.vectara.com/docs/rest-api/list-tool-servers.md)
- [Get tool server](https://docs.vectara.com/docs/rest-api/get-tool-server.md)
- [Sync tool server](https://docs.vectara.com/docs/rest-api/sync-tool-server.md)

### Lambda Tool Lifecycle
- [Create and test lambda tools (guide)](https://docs.vectara.com/docs/agents/create-a-lambda-tool.md)
- [Lambda tools (concept)](https://docs.vectara.com/docs/agents/lambda-tools.md)
- [Custom tools (overview)](https://docs.vectara.com/docs/agents/custom-tools.md)
- [Create tool](https://docs.vectara.com/docs/rest-api/create-tool.md)
- [Test lambda without creation](https://docs.vectara.com/docs/rest-api/test-lambda-tool-without-creation.md)
- [Test lambda](https://docs.vectara.com/docs/rest-api/test-tool.md)
- [Update tool](https://docs.vectara.com/docs/rest-api/update-tool.md)
- [Delete tool](https://docs.vectara.com/docs/rest-api/delete-tool.md)

### Search and Retrieval
- [Search Quick Start](https://docs.vectara.com/docs/search-and-retrieval/search-quick-start.md)
- [Hybrid Search](https://docs.vectara.com/docs/search-and-retrieval/hybrid-search.md)
- [Query a Corpus](https://docs.vectara.com/docs/rest-api/query-corpus.md)
- [Search a Corpus](https://docs.vectara.com/docs/rest-api/search-corpus.md)
- [Filters](https://docs.vectara.com/docs/search-and-retrieval/filters.md)
- [Reranking](https://docs.vectara.com/docs/search-and-retrieval/rerankers/vectara-multi-lingual-reranker.md)

### API Reference
- [OpenAPI Spec](https://api.vectara.io/v2/openapi.json)
- [Full Documentation Index](https://docs.vectara.com/llms.txt)
