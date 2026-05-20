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
- **`vectara-agent-orchestration`** — multi-step state machines (`first_step` + `steps[]` + `next_steps`), structured-output gating, sub-agents, runtime-constrained generation, Slack/connector-driven workflows, two-agent gates for human-in-the-loop approvals. Use for anything more complex than a single conversational loop.

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
   - `model` — the LLM for reasoning (e.g., `gpt-4o`, `gemini-2-5-flash-vertex`)

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
    "name": "gpt-4o",
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

The agent responds with a list of events:
- `input_message` — your original message
- `thinking` — agent's reasoning process
- `tool_input` / `tool_output` — tool calls and results. Note: corpus search results are in `tool_output.search_results` (not `results`)
- `agent_output` — the final response

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
