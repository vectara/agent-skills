---
name: vectara-python-sdk
description: Build Python applications, scripts, and notebooks against the Vectara platform using the official `vectara` SDK (pip install vectara). Covers sync + async clients, API-key and OAuth init, and the agents / corpora / documents / upload / chats / queries / generation_presets / rerankers / api_keys / users modules, plus the SDK's three-stage error model (Pydantic ValidationError, httpx transport errors, vectara.core.ApiError).
tags: [vectara, python, sdk, rag, agents, retrieval]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Python SDK

You are helping a developer build a Python application, script, or notebook against the Vectara platform using the official `vectara` SDK. The SDK is a typed parallel surface to the REST API — every REST endpoint has a method, every request body has a Pydantic model. Use it instead of hand-rolling `httpx`/`requests` calls.

## When to use this skill

- The developer is writing Python (app, FastAPI/Flask service, batch script, Jupyter notebook, Lambda) that talks to Vectara.
- They want typed request/response objects, not loose dicts and JSON.
- They're already importing `from vectara import ...` or have `pip install vectara` in `requirements.txt` / `pyproject.toml`.
- They need async, streaming, retries, or proper error classes — all things hand-rolled HTTP gives up.

If they're shelling out to `curl` or writing a Node/Go service, the `vectara-agents` (REST) skill is the right one. If they need OAuth flow plumbing into agent tools, see `vectara-agent-auth-and-secrets`. For multi-step agent state machines, see `vectara-agent-orchestration`.

## Critical Rules

1. **Install**: `pip install vectara`. Python 3.7+ required. Pin in `requirements.txt` for reproducible builds.
2. **Top-level import is `from vectara import Vectara`** (sync) **or `from vectara import AsyncVectara`** (async). Both are first-class. Do not invent `VectaraClient`, `vectara.Client`, or `vectara.client.Vectara` — none exist.
3. **API-key client init**: `Vectara(api_key="zwt_...")`. Pass the key positionally-by-name; no `customer_id` is required for v2.
4. **OAuth client init**: `Vectara(client_id="...", client_secret="...")`. The SDK handles the token dance and refresh internally — do not call a token endpoint yourself.
5. **The SDK mirrors the REST surface 1:1.** Method paths follow endpoint paths: `POST /v2/corpora` → `client.corpora.create(...)`; `GET /v2/agents/{key}/sessions` → `client.agent_sessions.list(agent_key)`; `POST /v2/upload_file` → `client.upload.file(...)`. If a REST endpoint exists, the SDK has a method for it.
6. **Errors come in three layers**: `pydantic.ValidationError` (client-side, before the wire), `httpx.TimeoutException` / `httpx.ConnectError` (transport), and `vectara.core.ApiError` (server returned a non-2xx). Catch them separately — they mean different things.
7. **Use typed request models, not dicts, where the SDK exposes one.** `StructuredDocument`, `CoreDocument`, `SearchCorporaParameters`, `GenerationParameters`, `ChatParameters`, `AgentToolConfiguration_CorporaSearch`, `FirstAgentStep`, etc. The wire format accepts dicts, but typed models give you IDE completion and catch typos at construction time, not from a 400.
8. **Corpus identifier is `key` (string).** Use `key="my-docs"` on create, `corpus_key="my-docs"` everywhere else. Never `corpus_id` (integer) — that's a v1 concept and not used by the SDK.
9. **The SDK auto-retries 5xx and certain 429s** with exponential backoff. Override per-call via `request_options={"max_retries": N}` if you need different behavior.
10. **Sync `Vectara` inside `async def`** will block the event loop. Use `AsyncVectara` and `await` every call (`await client.corpora.list()`, `async for chunk in client.query_stream(...)`).

## Client initialization

### API key (most common)

```python
from vectara import Vectara

client = Vectara(api_key="zwt_abc123...")
```

In practice, pull the key from the environment:

```python
import os
from vectara import Vectara

client = Vectara(api_key=os.environ["VECTARA_API_KEY"])
```

API keys carry roles: `serving` (query-only, safest for frontends), `indexing` (write), `admin` (everything). Create per-purpose keys via `client.api_keys.create(...)` rather than sharing one admin key.

### OAuth 2.0

```python
from vectara import Vectara

client = Vectara(
    client_id="your-oauth-client-id",
    client_secret="your-oauth-client-secret",
)
```

The SDK fetches and refreshes tokens automatically. Use OAuth for backend services, multi-user apps, and anything where granular auditing matters. User-management endpoints (`client.users.*`) **require** OAuth — API keys cannot call them.

### Async client

```python
import asyncio
from vectara import AsyncVectara

async def main():
    client = AsyncVectara(api_key=os.environ["VECTARA_API_KEY"])
    response = await client.api_keys.create(
        name="AsyncSearchKey",
        api_key_role="serving",
        corpus_keys=["my-docs"],
    )
    print(response.id)

asyncio.run(main())
```

Every sync method on `Vectara` has an async twin on `AsyncVectara` with the same signature. Streaming endpoints become `async for`.

### Timeouts and retries

```python
client = Vectara(api_key=os.environ["VECTARA_API_KEY"], timeout=30.0)

# Per-call override:
client.corpora.list(request_options={"timeout_in_seconds": 60, "max_retries": 3})
```

## Module taxonomy

The SDK is organized into modules that mirror REST resources. The most common entry points:

| Module | What it does | Typical entry methods |
|---|---|---|
| `client.corpora` | CRUD on corpora (collections of indexed docs) | `.create()`, `.list()`, `.get()`, `.update()`, `.delete()`, `.search()`, `.query()` |
| `client.documents` | Index, list, fetch, update metadata, delete, summarize | `.create()`, `.list()`, `.get()`, `.update()`, `.delete()`, `.summarize()` |
| `client.upload` | Upload binary files (PDF, DOCX, TXT, HTML, MD) for parsing + chunking | `.file()` |
| `client.query` / `client.query_stream` | One-shot RAG query against one or more corpora | `client.query(...)`, `client.query_stream(...)` |
| `client.chats` + `client.create_chat_session(...)` | Stateful multi-turn conversation with history persistence | `create_chat_session()`, `session.chat()`, `session.chat_stream()`, `client.chats.list()`, `client.chats.get()`, `client.chats.turns.list()` |
| `client.agents` | Create / list / get / update / delete agents; manage identity | `.create()`, `.list()`, `.get()`, `.update()`, `.delete()`, `.get_identity()`, `.update_identity()` |
| `client.agent_sessions` | Per-agent chat sessions (one per conversation) | `.create()`, `.list()`, `.get()`, `.update()`, `.delete()` |
| `client.agent_events` | Send messages to an agent session; read event stream | `.create()`, `.list()` |
| `client.generation_presets` | List Vectara's prebuilt LLM presets; reference via `generation_preset_name` | `.list()` (use names like `mockingbird-2.0`, `vectara-summary-ext-24-05-med-omni`) |
| `client.rerankers` | Discover available rerankers; reference via `reranker_name` in search | `.list()` |
| `client.api_keys` | Provision and rotate API keys programmatically | `.create()`, `.list()`, `.get()`, `.update()`, `.delete()` |
| `client.users` | OAuth-only — create / list / update / delete users, reset passwords | `.create()`, `.list()`, `.get()`, `.update()`, `.delete()`, `.reset_password()` |
| `client.health` | Connectivity / auth verification | `.check()` |

Typed models live alongside the methods. Common ones imported directly from `vectara`:

```python
from vectara import (
    Vectara,
    AsyncVectara,
    StructuredDocument,
    StructuredDocumentSection,
    CoreDocument,
    CoreDocumentPart,
    SearchCorporaParameters,
    GenerationParameters,
    ChatParameters,
)
```

Agent-related models live deeper:

```python
from vectara.agent_events.types import CreateAgentEventsRequestBody_InputMessage
# Agent construction models (AgentModel, FirstAgentStep, AgentStepInstruction_Inline,
# AgentOutputParser_Default, AgentToolConfiguration_CorporaSearch,
# AgentCorporaSearchQueryConfiguration, AgentSearchCorporaParameters,
# AgentKeyedSearchCorpus, AgentToolConfiguration_WebSearch) are also under vectara.* —
# rely on IDE auto-import rather than memorizing the deep paths.
```

## Worked example: end-to-end (create corpus → index → query → generate)

```python
import os
import time
from vectara import (
    Vectara,
    StructuredDocument,
    StructuredDocumentSection,
    SearchCorporaParameters,
    GenerationParameters,
)
from vectara.core.api_error import ApiError

client = Vectara(api_key=os.environ["VECTARA_API_KEY"])

# 1. Create a corpus with filter attributes. Filter attributes are immutable
#    after creation — define every field you will ever filter on now.
try:
    corpus = client.corpora.create(
        key="my-docs",
        name="My Documentation",
        description="Product docs and FAQs",
        filter_attributes=[
            {"name": "department", "level": "document", "type": "text",    "indexed": True},
            {"name": "year",       "level": "document", "type": "integer", "indexed": True},
            {"name": "doc_type",   "level": "document", "type": "text",    "indexed": True},
        ],
    )
    print(f"Created corpus: {corpus.key}")
    time.sleep(2)  # let the corpus propagate before writing
except ApiError as e:
    if e.status_code == 409:
        print("Corpus exists, continuing")
    else:
        raise

# 2. Index a structured document with metadata.
doc = StructuredDocument(
    id="hr-handbook-2025",
    type="structured",
    metadata={"department": "hr", "year": 2025, "doc_type": "policy"},
    sections=[
        StructuredDocumentSection(
            title="PTO Policy",
            text="Full-time employees receive 20 days of paid time off per year after 90 days of employment.",
        ),
        StructuredDocumentSection(
            title="Remote Work",
            text="Up to 3 days per week of remote work is permitted with manager approval.",
        ),
    ],
)
client.documents.create(corpus_key="my-docs", request=doc)

# 3. Run a RAG query with metadata filtering and a generation preset.
search = SearchCorporaParameters(
    corpora=[{
        "corpus_key": "my-docs",
        "metadata_filter": "doc.department = 'hr' AND doc.year = 2025",
        "lexical_interpolation": 0.025,
    }],
    context_configuration={"sentences_before": 2, "sentences_after": 2},
    reranker={
        "type": "customer_reranker",
        "reranker_name": "Rerank_Multilingual_v1",
    },
)
generation = GenerationParameters(
    generation_preset_name="vectara-summary-ext-24-05-med-omni",
    max_used_search_results=20,
    response_language="eng",
    enable_factual_consistency_score=True,
)

response = client.query(
    query="How much PTO do full-time employees get?",
    search=search,
    generation=generation,
)

print("Summary:", response.summary)
print("FCS:", response.factual_consistency_score)
for r in response.search_results[:3]:
    print(f"  [{r.score:.3f}] {r.text[:120]}")
```

For binary files (PDF, DOCX) the indexing step is `client.upload.file(...)` instead — Vectara parses, chunks, and indexes automatically:

```python
with open("user_guide.pdf", "rb") as f:
    client.upload.file(
        corpus_key="my-docs",
        file=f,                       # pass the file object directly (streamed)
        filename="user_guide.pdf",
        metadata={"document_type": "manual", "version": "2.1"},
        table_extraction_config={"extract_tables": True},
        chunking_strategy={"type": "sentence_chunking_strategy"},
    )
```

Each file is capped at 10 MB. The filename becomes the document ID — re-uploading the same name with different content returns 409.

## Streaming queries

```python
response = client.query_stream(
    query="How do I troubleshoot login issues?",
    search=search,
    generation=generation,
)
for chunk in response:
    if hasattr(chunk, "generation_chunk") and chunk.generation_chunk:
        print(chunk.generation_chunk, end="", flush=True)
    elif hasattr(chunk, "factual_consistency_score"):
        fcs = chunk.factual_consistency_score
```

For async, switch to `AsyncVectara` and iterate with `async for`.

## Error handling

The SDK surfaces failures at three distinct stages:

| Stage | Exception | When it fires | What to do |
|---|---|---|---|
| Client-side validation | `pydantic.ValidationError` | Wrong type or missing required field on a typed model (e.g. `limit="ten"` instead of `10`) | Fix the call site; catch `e.errors()` for per-field detail |
| Network / transport | `httpx.TimeoutException`, `httpx.ConnectError`, other `httpx` exceptions | Timeout, DNS failure, TLS handshake failure, connection reset | Retry with backoff; surface "service unavailable" |
| Vectara API | `vectara.core.ApiError` (alias `vectara.core.api_error.ApiError`) | Server returned non-2xx; exposes `e.status_code` and `e.body` | Inspect `status_code`; map to user-facing error |

```python
import httpx
from pydantic import ValidationError
from vectara import Vectara, SearchCorporaParameters
from vectara.core.api_error import ApiError

client = Vectara(api_key=os.environ["VECTARA_API_KEY"], timeout=30.0)

try:
    search = SearchCorporaParameters(corpora=[{"corpus_key": "my-docs"}])
    response = client.query(query="What is Vectara?", search=search)

except ValidationError as e:
    # Client-side schema error — happened before any HTTP call.
    for err in e.errors():
        print(f"  {'.'.join(map(str, err['loc']))}: {err['msg']}")

except httpx.TimeoutException:
    # Transport — retry or fail gracefully.
    raise

except httpx.ConnectError:
    # Network down / DNS — alert ops.
    raise

except ApiError as e:
    if e.status_code == 401:
        raise RuntimeError("Invalid API key — check VECTARA_API_KEY")
    elif e.status_code == 403:
        raise RuntimeError("Key lacks permission for this corpus / operation")
    elif e.status_code == 404:
        raise RuntimeError(f"Resource not found: {e.body}")
    elif e.status_code == 409:
        # Idempotency: resource already exists. Often expected.
        pass
    elif e.status_code == 429:
        # Rate limited — backoff. SDK retries transparently by default.
        raise
    else:
        raise RuntimeError(f"Vectara API error {e.status_code}: {e.body}")
```

Status-code cheatsheet:

| Status | Meaning | Most common cause |
|---|---|---|
| 400 | Bad request | Invalid filter expression, missing required field, undefined filter attribute |
| 401 | Unauthorized | Bad / expired API key or OAuth token |
| 403 | Forbidden | Key role lacks permission (e.g. `serving` key trying to index) |
| 404 | Not found | Wrong `corpus_key`, `agent_key`, `document_id` |
| 408 | Request timeout | Slow upstream (LLM); retry with longer `request_timeout` |
| 409 | Conflict | Corpus / document / API key already exists |
| 413 | Payload too large | File > 10 MB |
| 429 | Rate limited | Slow down; SDK auto-retries transient cases |
| 5xx | Server error | Transient; SDK auto-retries with exponential backoff |

## Agents in Python

Agents are the SDK's first-class way to combine RAG, tool use, and LLM reasoning into a stateful conversational worker. The flow is: create the agent once → create a session per conversation → post `input_message` events and read the event stream back.

### Create an agent

```python
from vectara import Vectara
# These typed models all live under the top-level `vectara` namespace:
from vectara import (
    AgentToolConfiguration_CorporaSearch,
    AgentCorporaSearchQueryConfiguration,
    AgentSearchCorporaParameters,
    AgentKeyedSearchCorpus,
    AgentModel,
    FirstAgentStep,
    AgentStepInstruction_Inline,
    AgentOutputParser_Default,
    GenerationParameters,
)

client = Vectara(api_key=os.environ["VECTARA_API_KEY"])

agent = client.agents.create(
    name="Support Agent",
    description="Customer support agent powered by RAG",
    tool_configurations={
        "corpora_search": AgentToolConfiguration_CorporaSearch(
            query_configuration=AgentCorporaSearchQueryConfiguration(
                search=AgentSearchCorporaParameters(
                    corpora=[AgentKeyedSearchCorpus(corpus_key="my-docs")],
                ),
                generation=GenerationParameters(),
            ),
        ),
    },
    model=AgentModel(name="gpt-5.4"),
    first_step=FirstAgentStep(
        name="main",
        instructions=[
            AgentStepInstruction_Inline(
                name="system",
                template="You are a helpful support assistant. Cite sources.",
            ),
        ],
        output_parser=AgentOutputParser_Default(),
    ),
)
print(f"Agent: {agent.key}")
```

The `tool_configurations` map keys can be anything — `corpora_search`, `knowledge_base`, `web` — they're tool *handles* the LLM sees. The value's `type`-shaped model determines what kind of tool it is.

> **SDK lag vs. REST API.** The current Python SDK only supports a **single step** via the `first_step=FirstAgentStep(...)` parameter — the SDK docs explicitly note multi-step workflows are not yet supported. The REST API (covered in `vectara-agents`) has moved on to `first_step_name` + a `steps{}` map for true multi-step state machines (`vectara-agent-orchestration`). If you need state machines, sub-agents, `reentry_step`, or `allowed_skills`, call the REST API directly (or via `client._client_wrapper` / a thin `httpx` layer) until the SDK catches up. The `name="main"` on `FirstAgentStep` is the SDK-side analog of `first_step_name`.

### Create a session and post a message

```python
from vectara.agent_events.types import CreateAgentEventsRequestBody_InputMessage

session = client.agent_sessions.create(
    agent.key,
    metadata={"user_id": "user_123", "channel": "web"},
)

response = client.agent_events.create(
    agent.key,
    session.key,
    request=CreateAgentEventsRequestBody_InputMessage(
        messages=[{"type": "text", "content": "How do I reset my password?"}],
        stream_response=False,
    ),
)

for event in response.events:
    if getattr(event, "type", None) == "agent_output":
        print("Agent:", event.content)
```

### Read events back later

```python
events = client.agent_events.list(agent.key, session.key)
for event in events:
    print(f"[{event.type}] {getattr(event, 'content', '')[:80]}")
```

Event types you'll encounter: `input_message`, `agent_output`, `tool_input`, `tool_output`, `thinking`, `structured_output`, `step_transition`, `compaction`, `context_limit_exceeded`, `session_interrupted`, `artifact_upload`. Always use `getattr(event, "type", None)` — the polymorphic event union doesn't always have `.type` on the same path.

### Multi-turn

A session keeps history automatically. Just call `client.agent_events.create(...)` again on the same `session.key` — the agent sees previous turns.

### Listing, updating, deleting

```python
for a in client.agents.list(limit=10, filter="support"):
    print(a.key, a.name)

client.agents.update(agent.key, description="Updated support agent", enabled=True)
client.agents.delete(agent.key)
```

For the orchestration / `steps[]` / `allowed_tools` / `next_steps` / sub-agent patterns, see the `vectara-agent-orchestration` skill. For OAuth and `$ref` secret resolution on agent tools, see `vectara-agent-auth-and-secrets`.

## Chats (simpler conversational RAG)

If you don't need tools or multi-step reasoning, `client.create_chat_session(...)` is a lighter alternative to agents — RAG plus history, nothing else:

```python
from vectara import SearchCorporaParameters, GenerationParameters, ChatParameters

session = client.create_chat_session(
    search=SearchCorporaParameters(corpora=[{"corpus_key": "my-docs"}]),
    generation=GenerationParameters(
        generation_preset_name="vectara-summary-ext-24-05-med-omni",
        max_used_search_results=20,
        enable_factual_consistency_score=True,
    ),
    chat_config=ChatParameters(store=True),   # store=True for multi-turn history
)

r1 = session.chat(query="What is machine learning?")
r2 = session.chat(query="What are the main types?")          # context carries
r3 = session.chat(query="Give examples of supervised learning?")
print(r3.answer, r3.factual_consistency_score)
```

Streaming: `session.chat_stream(query=...)` returns chunks with `.generation_chunk`. List past chats with `client.chats.list()`, fetch turn history with `client.chats.turns.list(chat_id=...)`.

## Generation presets and rerankers

Presets are referenced by name inside `GenerationParameters`. Common ones:

- `mockingbird-2.0` — high accuracy, Vectara's tuned RAG model.
- `vectara-summary-ext-24-05-med-omni` — GPT-4o-backed, strong for conversational answers.

Rerankers are referenced by name inside `SearchCorporaParameters.reranker`:

```python
search = SearchCorporaParameters(
    corpora=[{"corpus_key": "my-docs"}],
    reranker={
        "type": "customer_reranker",
        "reranker_name": "Rerank_Multilingual_v1",
        "limit": 100,
        "cutoff": 0.6,
    },
)
```

Other reranker `type` values: `mmr` (diversity, takes `diversity_bias`), `userfn` (custom JSONPath/JS scoring expression in `user_function`), `chain` (runs a `rerankers: [...]` list sequentially). Discover what's available with `client.rerankers.list(filter=".*multilingual.*")`.

## Metadata and filtering

- Define filter fields in `filter_attributes` on `client.corpora.create(...)` — **they can never be added later**.
- Each attribute has `name`, `level` (`"document"` or `"part"`), `type` (`"text"`, `"integer"`, `"real"`, `"boolean"`), and `indexed: True`.
- Attach values via `metadata={...}` on the document (and per-section/per-part for `"level": "part"` attributes).
- Filter at query time with `metadata_filter` strings: `"doc.department = 'hr' AND doc.year = 2025"`. Use `doc.` for document-level, `part.` for part-level. Single quotes for strings. `=, !=, >, >=, <, <=, IN, AND, OR` supported.
- The 400 you'll see when this is wrong: `INVALID_ARGUMENT: ... Unrecognized references: doc.department, doc.year`. Means the corpus has no filter attribute for that name. There is no migration path — recreate the corpus.

## API keys and users

```python
# Provision a query-only key for a frontend.
new_key = client.api_keys.create(
    name="SearchKey",
    api_key_role="serving",
    corpus_keys=["my-docs"],
)
print(new_key.id, new_key.api_key_role)

# Disable rather than delete to preserve audit trail.
client.api_keys.update(api_key_id=new_key.id, enabled=False)
```

Roles: `serving` (query-only — give to external clients), `indexing` (write), `admin` (everything — backend only).

Users require OAuth init, not API keys. Usernames must be percent-encoded when passed as path params:

```python
import urllib.parse
from vectara import Vectara

client = Vectara(client_id="...", client_secret="...")
client.users.create(
    email="alice@example.com",
    username="alice",
    api_roles=["corpus_admin", "query_writer"],
)

username = urllib.parse.quote("alice@example.com")
client.users.update(username=username, enabled=True, api_roles=["corpus_admin"])
client.users.reset_password(username=username)
```

## Common Mistakes

- **Mixing v1 and v2 vocabulary.** v2 (and the SDK) use `corpus_key` (string), not `corpus_id` (integer). There is no `customer_id` argument to the SDK client.
- **Passing raw dicts where a typed model is expected.** `client.documents.create(corpus_key=..., request={...})` may work at runtime but you lose typing. Build `StructuredDocument(...)` / `CoreDocument(...)` instead.
- **Forgetting `request=` on `documents.create`.** The signature is `documents.create(corpus_key=..., request=<doc>)`. Positional second arg won't work.
- **Calling `Vectara(...)` inside `async def` and `await`-ing methods.** `Vectara` is synchronous — `await` will fail. Use `AsyncVectara` in async code.
- **Catching `Exception` and losing the three-stage distinction.** Validation errors mean a bug in your code; transport errors mean retry; `ApiError` means look at `status_code`. Don't merge them.
- **Adding `filter_attributes` after corpus creation.** It silently won't happen — `corpora.update` doesn't accept `filter_attributes`. You must delete and recreate the corpus.
- **Re-uploading a file with the same `filename` and different content.** Returns 409. The filename is the document ID. `documents.delete(...)` first, then re-upload.
- **Putting `metadata_filter` inside `SearchCorporaParameters` itself instead of inside the per-corpus dict.** It belongs inside `corpora=[{"corpus_key": ..., "metadata_filter": "..."}]`.
- **Forgetting `store=True` on `ChatParameters`.** Without it, the chat session won't keep history and multi-turn references won't work.
- **Calling user-management endpoints with an API key.** They require OAuth. You'll get 403.
- **Trying to add filter attributes after the fact with an `update` call.** Doesn't exist. Re-create the corpus.
- **Using a sync `Vectara` client from inside an `asyncio` task.** Blocks the event loop. Use `AsyncVectara` and `await` every call.
- **Iterating `documents.list(...)` once, then again.** The result is an iterator — re-call the method, don't try to rewind.

## References

- [Vectara Python SDK overview](https://docs.vectara.com/docs/sdk/python/vectara-python-sdk.md)
- [Python quickstart](https://docs.vectara.com/docs/sdk/python/python-quickstart.md)
- [Error handling](https://docs.vectara.com/docs/sdk/python/python-error-handling.md)
- [Agents](https://docs.vectara.com/docs/sdk/python/agents.md)
- [Query](https://docs.vectara.com/docs/sdk/python/query.md)
- [Corpus](https://docs.vectara.com/docs/sdk/python/corpus.md)
- [Documents](https://docs.vectara.com/docs/sdk/python/documents.md)
- [Chats](https://docs.vectara.com/docs/sdk/python/chats.md)
- [Upload](https://docs.vectara.com/docs/sdk/python/upload.md)
- [API Keys](https://docs.vectara.com/docs/sdk/python/api_keys.md)
- [Generation Presets](https://docs.vectara.com/docs/sdk/python/generation_presets.md)
- [Rerankers](https://docs.vectara.com/docs/sdk/python/rerankers.md)
- [Metadata](https://docs.vectara.com/docs/sdk/python/metadata.md)
- [Users](https://docs.vectara.com/docs/sdk/python/users.md)
- [Customer Support KB tutorial](https://docs.vectara.com/docs/sdk/python/tutorials/customer-support-kb.md)
- [OpenAPI spec](https://api.vectara.io/v2/openapi.json) — useful when an SDK signature is unclear
- [Full docs index](https://docs.vectara.com/llms.txt)
