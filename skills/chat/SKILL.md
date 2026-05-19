---
name: vectara-chat
description: Build conversational RAG with Vectara Chat — persistent chats and turns, the OpenAI-compatible /chat/completions endpoint, and custom LLM registration (OpenAI/Anthropic/Vertex/Azure). For multi-turn chatbot UX over a corpus without tool calls or multi-step workflows.
tags: [vectara, chat, rag, openai-compatible, llm, chatbot]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Chat — Conversational RAG and OpenAI-Compatible Completions

You are helping a developer build chatbot-style RAG against Vectara. This skill covers two distinct surfaces:

1. **Vectara Chat API** (`/v2/chats`, `/v2/chats/{chat_id}/turns`) — persistent, server-stored conversations over one or more corpora. Each chat is a container of turns; each turn is a query + answer pair, optionally with RAG search results and factual-consistency score.
2. **OpenAI-compatible Chat Completions** (`/v2/llms/chat/completions`) — a stateless, drop-in replacement for `client.chat.completions.create` that lets you point the OpenAI SDK at Vectara and use any LLM you have registered. Not RAG by itself — pure LLM passthrough.
3. **LLM registration** (`/v2/llms`) — CRUD for bringing your own model credentials (OpenAI, Anthropic, Azure OpenAI, Vertex AI/Gemini, self-hosted) so chats and completions can route to them.

## When to use this skill

- The developer wants a "chat with my docs" UX with conversation memory and citations.
- They want OpenAI SDK compatibility — `from openai import OpenAI; client = OpenAI(base_url="...")` — with Vectara as the backend.
- They need to register a customer-managed LLM (BYO key, custom model, Azure deployment, Vertex service account).
- They need to inspect or moderate chat history (list/get/delete chats and turns).

## Chat vs Agent — which surface to use

Pick **Chat** for conversational RAG over one or more corpora with no orchestration. Pick **Agent** when you need tools, multi-step flow, write actions, or human-in-the-loop. The two are different resources with different IDs and different APIs — don't mix them.

| If you need… | Use Chat (`vectara-chat`) | Use Agent (`vectara-agents` / `vectara-agent-orchestration`) |
|---|---|---|
| Single-turn or multi-turn RAG over corpora | yes | yes (overkill) |
| Tool calls (web search, custom HTTP, lambda, MCP) | no | yes |
| Multi-step state machine (`first_step` + `steps[]`, `next_steps`) | no | yes |
| OpenAI SDK drop-in (`client.chat.completions.create`) | yes (via `/v2/llms/chat/completions`) | no |
| Server-stored conversation history with `chat_id` / `turn_id` you can list and delete | yes | yes (different shape — `session_key`) |
| External-app credentials per user (OAuth refresh tokens, etc.) on tools | no (no tools) | yes (`vectara-agent-auth-and-secrets`) |
| Bring-your-own LLM via `POST /v2/llms` | yes | yes |
| Factual consistency score (HHEM) on each answer | yes | no (not surfaced in agent events) |
| Disable / hide a specific turn (`enabled: false`) without losing history | yes | no — use session deletion |

**Critical:** A chat `chat_id` (prefix `cht_…`) and an agent `session_key` are completely different identifiers in different resources. You cannot pass an agent session key to `/v2/chats/{chat_id}/turns` or vice versa.

## API Reference

- Base URL: `https://api.vectara.io/v2/`
- OpenAPI spec: `https://api.vectara.io/v2/openapi.json`
- Docs index: `https://docs.vectara.com/llms.txt`. Any doc page is available as markdown by appending `.md` to the URL (e.g. `https://docs.vectara.com/docs/rest-api/create-chat.md`).

## Critical Rules

1. **Chats are persistent containers; turns are request/response pairs.** `POST /v2/chats` creates a chat AND runs the first turn in one call, returning `{chat_id, turn_id, answer, search_results, factual_consistency_score, …}`. Subsequent turns go to `POST /v2/chats/{chat_id}/turns`.
2. **Authentication.** Send either `x-api-key: <key>` OR `Authorization: Bearer <token>` — never both. Same convention as every other Vectara endpoint.
3. **OpenAI-compatible endpoint URL is exact.** From the docs:
   - Direct HTTP path: `https://api.vectara.io/v2/llms/chat/completions`
   - OpenAI SDK `base_url`: `https://api.vectara.io/v2/llms` (the SDK appends `/chat/completions`).
   - Do NOT use `https://api.vectara.io/v1/...` (v1 is gone) and do NOT pass `https://api.vectara.io/v2/llms/chat/completions` as the SDK `base_url` (the SDK will double the path).
4. **The OpenAI-compatible endpoint is LLM passthrough, not RAG.** It does not query a corpus. If you want grounded answers from your data, use `POST /v2/chats` (or the agent APIs). The completions endpoint is for when you want OpenAI-shaped calls routed through Vectara to a model you've registered via `/v2/llms`.
5. **Registered LLMs must speak the OpenAI Chat Completions wire format** (or one of the supported adapter types: `openai-compatible`, `openai-responses`, `anthropic`, `vertex-ai`). Generic REST endpoints will not work — they must accept the OpenAI request body or be wrapped by one of the typed adapters.
6. **Streaming requires `stream_response: true` (Chat API) or `stream: true` (Completions API).** They're different field names because they're different endpoints. The completions endpoint also requires `Accept: text/event-stream` for clean SSE handling.
7. **`save_history` defaults to `true` on `POST /v2/chats/{chat_id}/turns`.** If you don't want stored history, send `save_history: false` (it overrides `chat.store`).
8. **Turn IDs match `trn_…`; chat IDs match `cht_…`.** Disabling a turn (`PATCH …/turns/{turn_id}` with `enabled: false`) disables that turn AND every subsequent turn. Enabling a previously disabled turn is not implemented — once disabled, stay disabled. Deleting a turn deletes all later turns too.
9. **The chat endpoints are marked deprecated in the OpenAPI spec** but remain in production use. Long-term, the agent APIs are the strategic surface; pick `chat` when you specifically want this simpler shape, OpenAI SDK compatibility, or factual consistency scoring.

## Worked example — chat → turn → list → disable

The Chat API runs the first turn as part of `POST /v2/chats`, so step 1 already gives you an `answer`. Use the returned `chat_id` for follow-ups.

```bash
# 1. Start chat (creates the chat and runs the first turn in one shot)
START=$(curl -s -X POST https://api.vectara.io/v2/chats \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What were the carbon reduction efforts by EU banks in 2023?",
    "search": {
      "corpora": [{"corpus_key": "esg_reports", "lexical_interpolation": 0.005}],
      "limit": 10,
      "context_configuration": {"sentences_before": 2, "sentences_after": 2},
      "reranker": {"type": "customer_reranker", "reranker_name": "Rerank_Multilingual_v1"}
    },
    "generation": {
      "generation_preset_name": "mockingbird-2.0",
      "max_used_search_results": 5,
      "response_language": "eng",
      "citations": {"style": "markdown", "url_pattern": "https://docs.example.com/{doc.id}", "text_pattern": "{doc.title}"},
      "enable_factual_consistency_score": true
    },
    "chat": {"store": true},
    "save_history": true
  }')

CHAT_ID=$(echo "$START" | jq -r '.chat_id')   # cht_...
TURN_ID=$(echo "$START" | jq -r '.turn_id')   # trn_...

# 2. Add a follow-up turn (the platform uses prior turns as conversation context)
curl -s -X POST "https://api.vectara.io/v2/chats/$CHAT_ID/turns" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Which of those banks reported the largest year-over-year reduction?",
    "search": {"corpora": [{"corpus_key": "esg_reports"}], "limit": 10},
    "generation": {"generation_preset_name": "mockingbird-2.0"}
  }'

# 3. List all turns
curl -s "https://api.vectara.io/v2/chats/$CHAT_ID/turns" \
  -H "x-api-key: $VECTARA_API_KEY"

# 4. Disable a turn (and all subsequent turns) — useful for moderation / "edit" UX
curl -s -X PATCH "https://api.vectara.io/v2/chats/$CHAT_ID/turns/$TURN_ID" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}'

# 5. Delete a chat (turns deleted with it; returns 204)
curl -s -X DELETE "https://api.vectara.io/v2/chats/$CHAT_ID" \
  -H "x-api-key: $VECTARA_API_KEY"
```

Streaming variant: add `"stream_response": true` to the turn body and parse the SSE stream. Event types include `chat_info` (with `chat_id`/`turn_id`), `search_results`, `generation_chunk`, `generation_end`, `factual_consistency_score`, `end`, and `error`.

## OpenAI SDK against Vectara — Python

Quoting verbatim from the Vectara docs (`use-external-applications-sdk.md`): the OpenAI SDK `base_url` is `https://api.vectara.io/v2/llms`. The SDK appends `/chat/completions` itself.

```python
from openai import OpenAI

client = OpenAI(
    api_key="your-vectara-api-key",
    base_url="https://api.vectara.io/v2/llms",
)

completion = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Vectara?"},
    ],
)

print(completion.choices[0].message.content)
```

If you prefer the Vectara-native `x-api-key` header instead of `Authorization: Bearer …`, pass a dummy `api_key` and inject the real key via `default_headers`:

```python
client = OpenAI(
    api_key="dummy_key_not_used",
    base_url="https://api.vectara.io/v2/llms",
    default_headers={"x-api-key": "your-vectara-api-key"},
)
```

Streaming with the SDK requires `Accept: text/event-stream` in your headers; the Vectara docs' reference client adds it via `default_headers` and parses the SSE stream manually (each `data:` line is OpenAI-compatible delta JSON; a literal `data: [DONE]` line ends the stream).

The `model` you pass is matched against the LLMs registered on your account (`GET /v2/llms`). Built-in models like `gpt-4o`, `gpt-4o-mini` are usually present out of the box; custom-registered models are addressed by the `name` you gave them in `POST /v2/llms`.

## LLM management — CRUD

Use `/v2/llms` to bring your own model. Every registered LLM must conform to one of four typed adapters; `openai-compatible` is the catch-all and works with anything that speaks the OpenAI Chat Completions wire format.

### Create — `POST /v2/llms`

```json
{
  "type": "openai-compatible",
  "name": "my-gpt5",
  "model": "gpt-5",
  "uri": "https://api.openai.com/v1/chat/completions",
  "auth": {"type": "bearer", "token": "sk-..."}
}
```

Other adapter types from the docs:

| Provider | `type` | Notable fields |
|---|---|---|
| OpenAI / generic OpenAI-compatible (Anthropic via OpenAI shim, OpenRouter, self-hosted vLLM/Ollama) | `openai-compatible` | `auth.type: "bearer"` or `"header"` (custom header like `x-api-key`), optional `headers` |
| OpenAI Responses API (o1, o3 reasoning) | `openai-responses` | Same auth surface as `openai-compatible`; for models that don't stream |
| Anthropic direct (or via Bedrock / Vertex Model Garden) | `anthropic` | `auth.type` includes `bedrock_static_iam`, `bedrock_api_key`, `vertex_service_account`, `vertex_access_token` for non-direct paths |
| Google Vertex AI / AI Studio Gemini | `vertex-ai` | `auth.type: "service_account"` (with `key_json`) or `"api_key"`; `uri` accepts Vertex base URI or full Google AI Studio URL |

Azure OpenAI example (custom header auth):

```json
{
  "type": "openai-compatible",
  "name": "my-azure-gpt4",
  "model": "gpt-4",
  "uri": "https://YOUR-RESOURCE.openai.azure.com/openai/deployments/YOUR-DEPLOYMENT/chat/completions?api-version=2024-02-15-preview",
  "auth": {"type": "header", "header": "api-key", "value": "your-azure-key"}
}
```

Vertex AI service-account example:

```json
{
  "type": "vertex-ai",
  "name": "my-gemini",
  "model": "gemini-2.5-flash",
  "uri": "https://us-central1-aiplatform.googleapis.com/v1/projects/YOUR-PROJECT/locations/us-central1",
  "auth": {"type": "service_account", "key_json": "{...service account JSON string...}"}
}
```

On create AND update, the platform performs a live test call against the upstream LLM. Bad credentials or unreachable URIs return HTTP 400 with the upstream error in `messages`.

### Read

```bash
# List (regex filter on name/description supported)
curl -s "https://api.vectara.io/v2/llms?filter=claude&limit=20" \
  -H "x-api-key: $VECTARA_API_KEY"

# Get one
curl -s "https://api.vectara.io/v2/llms/llm_1021844" \
  -H "x-api-key: $VECTARA_API_KEY"
```

Response includes `ownership` (`platform` for built-ins, `customer` for ones you created), `enabled`, `default`, and `capabilities` (`image_support`, `context_limit`, `tool_calling`, `structured_outputs`, `requires_role_alternation`). Capabilities are inferred from the model name if not provided at create time; explicit values override.

### Update — `PATCH /v2/llms/{llm_id}`

All fields are optional; `name` is immutable. Built-in (`platform`-owned) LLMs return 403 on update or delete.

```json
{
  "type": "openai-compatible",
  "model": "gpt-5-2026-03",
  "auth": {"type": "bearer", "token": "sk-new-rotated-token"},
  "capabilities": {"context_limit": 256000, "tool_calling": true}
}
```

Useful auxiliary fields on create/update: `idle_timeout_seconds` (1–3600, SSE read timeout; set to `null` on update to clear), `enabled` (toggle without deleting), `test_model_parameters` (extra fields sent during the connection test, e.g. `{"max_tokens": 512}` for models that require it).

### Delete — `DELETE /v2/llms/{llm_id}`

Returns 204. Built-ins (ownership `platform`) cannot be deleted.

## Common Mistakes

- Passing `https://api.vectara.io/v2/llms/chat/completions` as the OpenAI SDK `base_url`. The SDK appends `/chat/completions` itself — use `https://api.vectara.io/v2/llms`.
- Mixing an agent `session_key` with `chat_id`/`turn_id`. They are different resources; the Chat API does not accept agent IDs and vice versa.
- Expecting RAG out of the `/v2/llms/chat/completions` endpoint. It is pure LLM passthrough — no corpus search runs. For RAG, use `POST /v2/chats` or an agent with a `corpora_search` tool.
- Forgetting `stream_response: true` on chat turns (the field is `stream`, not `stream_response`, on the OpenAI-compatible endpoint — they're different).
- Forgetting `Accept: text/event-stream` on streaming requests — some clients silently buffer the whole SSE body and look hung.
- Putting an arbitrary REST URL in `POST /v2/llms` with no adapter that matches. The endpoint must speak the OpenAI Chat Completions wire format (or one of the typed adapters: `anthropic`, `vertex-ai`, `openai-responses`).
- Sending both `x-api-key` and `Authorization` headers — pick one.
- Trying to re-enable a disabled turn. Per the update-turn schema: "Enabling a turn is not implemented." If you need to restore conversation state, you'll have to recreate the chat.
- Deleting a middle turn and being surprised the later turns vanish too. Delete and disable both cascade forward.
- Hardcoding `corpus_id` (number) in chat search instead of `corpus_key` (string). Same rule as everywhere else in the v2 API.
- Trying to update or delete a built-in LLM. `ownership: "platform"` is read-only — 403.
- Using `prompt_name` / `prompt_text` in the chat `generation` block. Both are deprecated in favor of `generation_preset_name` / `prompt_template`.

## References

### Chat API
- [Chat overview](https://docs.vectara.com/docs/console-ui/vectara-chat-overview.md)
- [Create chat](https://docs.vectara.com/docs/rest-api/create-chat.md)
- [Create chat turn](https://docs.vectara.com/docs/rest-api/create-chat-turn.md)
- [Get chat](https://docs.vectara.com/docs/rest-api/get-chat.md)
- [Get chat turn](https://docs.vectara.com/docs/rest-api/get-chat-turn.md)
- [List chats](https://docs.vectara.com/docs/rest-api/list-chats.md)
- [List chat turns](https://docs.vectara.com/docs/rest-api/list-chat-turns.md)
- [Update chat turn](https://docs.vectara.com/docs/rest-api/update-chat-turn.md)
- [Delete chat](https://docs.vectara.com/docs/rest-api/delete-chat.md)
- [Delete chat turn](https://docs.vectara.com/docs/rest-api/delete-chat-turn.md)

### OpenAI-Compatible Completions
- [Create chat completion](https://docs.vectara.com/docs/rest-api/create-chat-completion.md)
- [Use OpenAI libraries with Vectara](https://docs.vectara.com/docs/tutorials/use-openai-libraries-with-vectara.md)
- [Use external applications SDK](https://docs.vectara.com/docs/tutorials/use-external-applications-sdk.md)

### LLM Management
- [Create LLM](https://docs.vectara.com/docs/rest-api/create-llm.md)
- [List LLMs](https://docs.vectara.com/docs/rest-api/list-ll-ms.md)
- [Get LLM](https://docs.vectara.com/docs/rest-api/get-llm.md)
- [Update LLM](https://docs.vectara.com/docs/rest-api/update-llm.md)
- [Delete LLM](https://docs.vectara.com/docs/rest-api/delete-llm.md)

### Tutorials
- [Build a chatbot](https://docs.vectara.com/docs/tutorials/build-a-chatbot.md) — uses the Agents API rather than `/v2/chats`; useful for the chatbot UX layer but the backend pattern is agent-sessions, not chat.

### API Reference
- [OpenAPI spec](https://api.vectara.io/v2/openapi.json)
- [Docs index](https://docs.vectara.com/llms.txt)
