---
name: vectara-agents
description: Build AI agents with the Vectara platform — create agents, configure tools, manage sessions, and search corpora.
tags: [vectara, agents, rag, ai, api]
compatibility: [claude-code, cursor, copilot, cline, windsurf]
---

# Vectara Agent Skills

You are helping a developer build with the Vectara platform. Vectara is an enterprise agentic platform for building trusted AI agents with retrieval augmented generation (RAG).

## API Reference

The full OpenAPI spec is available at `https://api.vectara.io/v2/openapi.json`. Fetch it when you need exact request/response schemas.

## Documentation Access

You can fetch any Vectara documentation page as markdown by appending `.md` to its URL. For example: `https://docs.vectara.com/docs/agents/agents.md`. The full documentation index is at `https://docs.vectara.com/llms.txt`. When you need detailed information about a Vectara feature, fetch the relevant markdown page directly.

## Critical Rules

1. **API base URL** is `https://api.vectara.io/v2/`.
2. **Authentication**: Use either `x-api-key` header with an API key, or `Authorization: Bearer <token>` with an OAuth token. Both are valid — use whichever the developer has. Never use both simultaneously.
3. **Agents are a first-class resource.** Create them via `POST /v2/agents`, not by stitching together query calls manually.
4. **Every agent needs three things:**
   - `tool_configurations` — at least one tool (corpora_search, web_search, or custom)
   - `first_step` — with `type: "conversational"`, `instructions`, and `output_parser`
   - `model` — the LLM for reasoning (e.g., `gpt-4o`, `gemini-2-5-flash-vertex`)
5. **Sessions track conversations.** Create a session before sending messages: `POST /v2/agents/{key}/sessions`
6. **Messages are events.** Send user input via `POST /v2/agents/{key}/sessions/{session_key}/events` with `type: "input_message"`
7. **Never hardcode corpus IDs.** Use `corpus_key` (string) not `corpus_id` (number).

## Agent Creation Pattern

```json
{
  "name": "my-agent",
  "description": "Agent description",
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
  "first_step": {
    "type": "conversational",
    "instructions": [{
      "type": "inline",
      "name": "My Instructions",
      "template": "You are a helpful assistant. Use the search tool to find relevant information. Cite your sources."
    }],
    "output_parser": {"type": "default"}
  },
  "model": {
    "name": "gpt-4o",
    "parameters": {"temperature": 0.3, "max_tokens": 1000}
  }
}
```

## Available Tool Types

| Tool Type | `type` value | Purpose |
|-----------|-------------|---------|
| Corpus Search | `corpora_search` | Search indexed Vectara corpora |
| Web Search | `web_search` | Search the internet |
| Sub-agent | `subagent` | Delegate to another agent |
| Lambda | `lambda` | Run custom Python functions |
| Document Conversion | built-in | Convert PDF/DOCX/PPTX to markdown |
| MCP | `mcp` | External tool servers via Model Context Protocol |

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
- Forgetting `output_parser: {"type": "default"}` in `first_step`
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
- [Create Agent API](https://docs.vectara.com/docs/api-reference/agent-apis/create-agent.md)

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
