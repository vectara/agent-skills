---
name: vectara
description: Build AI agents with the Vectara platform — create agents, configure tools, manage sessions, and search corpora using the v2 API.
tags: [vectara, agents, rag, ai, api]
compatibility: [claude-code, cursor, copilot, cline, windsurf]
---

# Vectara Agent Skills

You are helping a developer build with the Vectara platform. Vectara is an enterprise agentic platform for building trusted AI agents with retrieval augmented generation (RAG).

## Critical Rules

1. **Always use the v2 API.** The base URL is `https://api.vectara.io/v2/`. Never use `/v1/` endpoints — they are deprecated.
2. **Authentication**: Use `x-api-key` header with an API key, or `Authorization: Bearer <token>` with an OAuth token. Never use both.
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
- `tool_input` / `tool_output` — tool calls and results
- `agent_output` — the final response

## Common Mistakes to Avoid

- Using `/v1/query` instead of the agents API
- Using `corpus_id` (number) instead of `corpus_key` (string)
- Forgetting `output_parser: {"type": "default"}` in `first_step`
- Sending messages without creating a session first
- Putting `limit` inside `corpora[]` instead of in `search{}`

## References

For detailed documentation, fetch these pages:

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
- [Full Documentation Index](https://docs.vectara.com/llms.txt)
