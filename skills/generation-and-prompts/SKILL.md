---
name: vectara-generation-and-prompts
description: Configure Vectara generation — generation presets (CRUD), Apache Velocity prompt templates, custom prompts with document/part metadata, response languages, LLM selection (Mockingbird, GPT, Gemini, Claude), bring-your-own LLM via POST /v2/llms, and table summarization at ingest time. Covers how generation is wired into POST /v2/query and into the corpora_search agent tool.
tags: [vectara, generation, rag, prompts, velocity, mockingbird, byo-llm, summarization]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara — Generation and Prompt Engineering

You are helping a developer customize what Vectara's RAG layer produces: the summarizer LLM, the prompt template that wraps retrieved results, the metadata surfaced into the prompt, the response language, and how all of this gets reused across queries via named generation presets.

## When to use this skill

- The developer wants to swap the default summarizer for `mockingbird-2.0`, a GPT-4o variant, Gemini, or Claude.
- A query needs a custom prompt template — domain-specific instructions, RFI/RFP framing, structured output, metadata-aware citations.
- The same prompt + LLM + parameters need to be reused across many queries — bundle them in a generation preset.
- Summaries need to come back in a specific language (or follow the query language).
- The customer wants to register a third-party LLM (Claude, Gemini, OpenRouter, on-prem) and route generation through it.
- Tables in PDFs need a custom summarization prompt at ingest time.

If you need hallucination scoring or factual-consistency evaluation, use `vectara-hallucination-and-eval`. If you're wiring generation into an agent's `corpora_search` tool, also read `vectara-agents`.

## Critical Rules

1. **Reference presets by name; don't construct them inline at query time.** The query body's `generation.generation_preset_name` is a *string lookup* against either a platform preset (e.g. `mockingbird-2.0`) or a customer preset created via `POST /v2/generation_presets`. Build the preset once, reference it everywhere.
2. **Apache Velocity is the template language.** All `prompt_template` strings are evaluated by [Velocity 1.7](https://velocity.apache.org/engine/1.7/user-guide.html). Every variable starts with `$`. Use `${var}` (curly form) when adjacent characters would break parsing.
3. **The canonical loop variable is `$qResult`.** The platform binds `$vectaraQueryResults` as the result array; the foreach pattern documented across every prompt-engine page is exactly `#foreach ($qResult in $vectaraQueryResults) ... #end`. Use that name — examples, defaults, and Mockingbird's built-in prompt all do. (`$foreach.index`, `$foreach.first`, `$foreach.last` are the standard Velocity loop helpers.)
4. **`max_used_search_results` is what the LLM sees.** It caps how many retrieved results are passed to the prompt — independent of `search.limit`, which controls retrieval. Default is 25. Bigger = more complete summary, slower, and may exceed the model's context. If a prompt fails to produce output, *lower* this number first.
5. **Hallucination scoring is a separate skill.** `enable_factual_consistency_score` and HHEM/HCM/Mockingbird-2-Echo live in `vectara-hallucination-and-eval`. This skill covers what to *generate*; that skill covers how to *evaluate* what was generated.
6. **BYO LLM requires an OpenAI-compatible chat-completions surface** (or one of the two specialized types: `openai-responses` for o1/o3 reasoning models, `vertex-ai` for Gemini). Anthropic Claude, Azure OpenAI, OpenRouter, and on-prem deployments all work — they speak the OpenAI shape.
7. **A custom LLM cannot be referenced directly from a query.** You register it via `POST /v2/llms`, then either (a) wrap it in a `POST /v2/generation_presets` preset and reference *that* by name, or (b) point an existing preset at it via `model_parameters.llm_name` as a per-call override.
8. **Excluding `generation` disables summarization.** A query body with `search` but no `generation` block returns search results only — no summary, no LLM call.

## Generation presets — CRUD

A preset bundles a prompt template, an LLM, and default model parameters into a reusable named configuration. All values except the LLM name can be overridden at query time.

### Resource shape

| Field | Type | Notes |
|---|---|---|
| `id` | `string` (`gnp_*`) | Server-generated. |
| `name` | `string` (required) | What you reference via `generation_preset_name`. |
| `description` | `string` | Free text. |
| `llm_name` | `string` (required) | Must exist in `GET /v2/llms`. |
| `prompt_template` | `string` (required) | A Velocity template, usually a JSON-array literal of `{role, content}` messages. |
| `max_used_search_results` | `int32` | Default 25. Cap on results fed to the prompt. |
| `max_tokens` | `int32` | Hard cap on response tokens. Overrides `max_response_characters`. |
| `temperature` | `float` | 0.0 = deterministic, 1.0 = maximally creative. |
| `frequency_penalty` | `float` | 0.0–1.0. |
| `presence_penalty` | `float` | 0.0–1.0. |
| `additional_model_params` | `object` | Provider-specific extras. |
| `enabled` | `bool` | |
| `default` | `bool` | Whether it's the default for the bound LLM. |
| `ownership` | `"platform"` \| `"customer"` | Platform presets cannot be replaced or deleted. |

### Endpoints

| Verb | Path | Notes |
|---|---|---|
| `POST` | `/v2/generation_presets` | Create. Requires `name`, `llm_name`, `prompt_template`. |
| `GET` | `/v2/generation_presets?llm_name=&filter=&limit=&page_key=` | List. Regex `filter` matches name + description. |
| `PUT` | `/v2/generation_presets/{generation_preset_id}` | Full replace. Customer-owned only. |
| `DELETE` | `/v2/generation_presets/{generation_preset_id}` | Customer-owned only. 204 on success. |

### Create — full example

```json
POST /v2/generation_presets
{
  "name": "rfi-assistant-mockingbird",
  "description": "RFI/RFP assistant. Cites answerDate from doc metadata. Backed by Mockingbird 2.0.",
  "llm_name": "mockingbird-2.0",
  "prompt_template": "[ { \"role\": \"system\", \"content\": \"You are an RFI answering assistant. Use only information from the provided results.\" }, #foreach ($qResult in $vectaraQueryResults) #if ($foreach.first) { \"role\": \"user\", \"content\": \"Search for '${vectaraQuery}', and give me the first search result.\" }, { \"role\": \"assistant\", \"content\": \"${qResult.getText()}\" }, #else { \"role\": \"user\", \"content\": \"Give me the $vectaraIdxWord[$foreach.index] search result.\" }, { \"role\": \"assistant\", \"content\": \"$qResult.docMetadata().get('answerDate') ${qResult.getText()}\" }, #end #end { \"role\": \"user\", \"content\": \"Generate a comprehensive answer for ${vectaraQuery} in ${vectaraLangName}. Cite using [number] notation. If two answers conflict, prefer the most recent answerDate.\" } ]",
  "max_used_search_results": 8,
  "max_tokens": 1200,
  "temperature": 0.2,
  "frequency_penalty": 0.1,
  "presence_penalty": 0.1,
  "enabled": true
}
```

Use the preset in a query:

```json
POST /v2/query
{
  "query": "What was our RFI answer about SOC 2 Type II scoping?",
  "search": {
    "corpora": [{ "corpus_key": "rfi-answers" }],
    "limit": 20
  },
  "generation": {
    "generation_preset_name": "rfi-assistant-mockingbird",
    "max_used_search_results": 8,
    "response_language": "eng"
  }
}
```

### Per-call overrides

Every preset field except `llm_name` can be overridden in the query's `generation` block (`max_tokens`, `temperature`, `prompt_template`, etc.). To override the LLM without forking the preset, use `model_parameters.llm_name`:

```json
"generation": {
  "generation_preset_name": "rfi-assistant-mockingbird",
  "model_parameters": {
    "llm_name": "Claude 3.7 Sonnet",
    "temperature": 0.0,
    "max_tokens": 800
  }
}
```

## The Vectara Prompt Engine — Velocity syntax

Templates render to a JSON array of OpenAI-style chat messages: `[{ "role": "system" | "user" | "assistant", "content": "..." }, ...]`. The platform parses the rendered string as JSON, so the *output* of the template must be valid JSON.

### Available variables

| Variable | Description |
|---|---|
| `$vectaraQuery` | The end-user's query string. |
| `$vectaraQueryResults` | The array of retrieved results (sorted by relevance). Iterate with `#foreach`. |
| `$vectaraOutChars` | Requested response length in characters (from `max_response_characters`). |
| `$vectaraLangCode` | ISO 639-3 code of the requested response language (e.g. `eng`, `fra`, `deu`). |
| `$vectaraLangName` | Human-readable name of that language (e.g. `English`). |
| `$vectaraIdxWord` | Utility array converting 0-based index to ordinal word: `first`, `second`, ..., `tenth`. |

### Available functions on `$qResult`

| Function | Returns |
|---|---|
| `$qResult.getText()` *or* `$qResult.text()` | The result snippet text. |
| `$qResult.docMetadata()` | Map of document-level metadata. |
| `$qResult.docMetadata().present()` | `true` if any doc metadata exists (use inside `#if`). |
| `$qResult.docMetadata().get("title")` | Specific doc-metadata field. Unknown key returns empty. |
| `$qResult.partMetadata()` | Map of part-level metadata (page, section, offset, etc.). |
| `$qResult.partMetadata().present()` | `true` if any part metadata exists. |
| `$qResult.partMetadata().get("page")` | Specific part-metadata field. |

### Velocity control structures

- `#foreach ($qResult in $vectaraQueryResults) ... #end` — the canonical loop. Inside, `$foreach.index` is 0-based, `$foreach.count` is 1-based, `$foreach.first` and `$foreach.last` are booleans.
- `#if (condition) ... #elseif (...) ... #else ... #end` — branching.
- `${var}` — bracketed form, required when adjacent characters could form a longer identifier (e.g. `${vectaraQuery}s`).
- `$!var` — quiet reference: renders empty instead of literal `$var` when undefined.

### Worked example — system + per-result framing + final question

```text
[
  { "role": "system", "content": "You are a helpful search assistant." },
  #foreach ($qResult in $vectaraQueryResults)
    { "role": "user",      "content": "Give me the $vectaraIdxWord[$foreach.index] search result." },
    { "role": "assistant", "content": "${qResult.getText()}" },
  #end
  { "role": "user", "content": "Generate a summary for the query '${vectaraQuery}' based on the above results. Reply in ${vectaraLangName}." }
]
```

Mockingbird 2.0's *default* template is essentially this shape: an optional empty `system` message, a single `user` message whose content packs the query plus an inline `#foreach` over `$vectaraQueryResults` rendering `[<index+1>) <text>`. Custom Mockingbird prompts must follow Mockingbird's role rules: `system` allowed once at the start; `user` and `assistant` must alternate; the last message must be `user`.

## Custom prompts with metadata

Surface document and part metadata into the prompt so the LLM can cite dates, page numbers, source titles, or any custom attribute you indexed.

### Pattern — guard with `present()` then read fields

```text
[
  { "role": "system", "content": "You are a research assistant. Cite each claim with its source title and page." },
  #foreach ($qResult in $vectaraQueryResults)
    #if ($qResult.docMetadata().present())
      { "role": "user", "content": "Result $foreach.count from \"$qResult.docMetadata().get('title')\" (published $qResult.docMetadata().get('published_at'), page $qResult.partMetadata().get('page')):" },
    #else
      { "role": "user", "content": "Result $foreach.count:" },
    #end
    { "role": "assistant", "content": "${qResult.getText()}" },
  #end
  { "role": "user", "content": "Answer ${vectaraQuery}. After each factual claim, add a citation in the form [<title>, p<page>]." }
]
```

### Pattern — date-aware conflict resolution (RFI bot)

When multiple results answer the same RFI question, prefer the most recent. Embed `answerDate` into each assistant turn so the LLM can reason about recency:

```text
{ "role": "assistant", "content": "$qResult.docMetadata().get('answerDate') ${qResult.getText()}" }
```

Then in the final `user` message: *"If two results conflict, prefer the answer with the most recent answerDate."*

### Where to set a custom prompt

Three options, in increasing order of permanence:

1. **Inline per query** — `generation.prompt_template` in the `POST /v2/query` body. Overrides the preset's template for that one call.
2. **Customer preset** — `POST /v2/generation_presets` bakes the template into a named preset. Reference it via `generation_preset_name`.
3. **Agent tool config** — inside `corpora_search.query_configuration.generation.prompt_template` (or `generation_preset_name`) on an agent's tool. Every time the agent calls the tool, the configured prompt applies.

## Agent tool Velocity templates (separate variable namespace)

Agent *tool description templates* are also Velocity, but they bind a different set of variables — agent and session context, not search results:

| Variable | Description |
|---|---|
| `$agent.name` | Agent name. |
| `$agent.metadata` | Agent metadata map (e.g. `$agent.metadata.department`). |
| `$session.key` | Current session key. |
| `$session.metadata` | Session metadata map (e.g. `$session.metadata.user_email`). |
| `$currentDate` | ISO 8601 timestamp, e.g. `2025-10-24T15:30:45Z`. |

```text
#if($session.metadata.department) Search $session.metadata.department documents #else Search all documents #end for $agent.name on $currentDate
```

Don't try to use `$vectaraQueryResults` in a tool description template — it doesn't exist there. And don't try to use `$session.metadata` in a query prompt template — it doesn't exist there either.

## Response languages

Set `response_language` in the `generation` block of a query:

```json
"generation": {
  "generation_preset_name": "mockingbird-2.0",
  "response_language": "eng"
}
```

- Both **ISO 639-1** (two-letter, e.g. `en`, `fr`, `de`) and **ISO 639-3** (three-letter, e.g. `eng`, `fra`, `deu`) codes are accepted.
- `"auto"` asks the platform to detect the query's language and respond in it. Detection is imperfect — many languages share borrowed words. Pass an explicit code whenever the client knows it (browser locale, user setting).
- The canonical list of supported codes lives in [`common.proto#L10`](https://github.com/vectara/protos/blob/main/common.proto#L10) — fetch when you need the full enum.
- **Mockingbird 2.0** is fully cross-lingual across English, Spanish, French, Arabic, Chinese, Japanese, and Korean — query in one language, retrieve in another, summarize in a third.
- **Factual Consistency Score** is only computed for `eng`, `deu`, `fra`. If you need scoring outside those languages, see `vectara-hallucination-and-eval`.

Inside templates you can read `$vectaraLangCode` (e.g. `"ara"`) and `$vectaraLangName` (e.g. `"Arabic"`) to inject the language into instructions: *"Answer in ${vectaraLangName}."*

## LLM selection — taxonomy

### Platform presets (no setup, reference by name)

| Preset name | Backing LLM | When to use |
|---|---|---|
| `mockingbird-2.0` | Mockingbird 2.0 (Vectara) | **Default for RAG.** Best citation accuracy, multilingual, lowest hallucination rate when paired with HHEM (Mockingbird-2-Echo: 0.9%). |
| `vectara-summary-ext-24-05-med-omni` | `gpt-4o` | High-quality citations; good general-purpose. |
| `vectara-summary-ext-24-05-large` | `gpt-4-turbo` | When `gpt-4o` isn't available; legacy. |
| `vectara-summary-ext-24-05-med` | `gpt-4` | Older GPT-4. |
| `vectara-summary-ext-24-05-sml` | `gpt-3.5-turbo` | Cost-sensitive, high-volume. |
| `vectara-summary-table-query-ext-dec-2024-gpt-4o` | `gpt-4o` | Corpora with table-heavy content. |
| `vectara-summary-ext-v1.2.0`, `vectara-summary-ext-v1.3.0` | legacy | **Deprecating.** Migrate to a `vectara-summary-ext-24-05-*` preset. |

### Use-case matrix

| Need | Recommended preset / model |
|---|---|
| Enterprise RAG with citations | `mockingbird-2.0` |
| Cross-lingual answers (EN→FR, JA→EN, etc.) | `mockingbird-2.0` |
| Lowest hallucination rate | `mockingbird-2.0` + HHEM (see `vectara-hallucination-and-eval`) |
| Long-form / creative synthesis | `vectara-summary-ext-24-05-med-omni` (GPT-4o) |
| High-volume, cost-sensitive | `vectara-summary-ext-24-05-sml` (GPT-3.5) or a BYO Gemini Flash |
| Table-heavy corpora | `vectara-summary-table-query-ext-dec-2024-gpt-4o` |
| Reasoning models (o1, o3) | BYO via `openai-responses` |
| Anthropic Claude | BYO via `openai-compatible` against `api.anthropic.com` |
| Gemini 2.5 Pro / Flash | BYO via `vertex-ai` |

Discover what's available on a tenant:

```bash
curl -s -H "x-api-key: $VECTARA_API_KEY" \
  "https://api.vectara.io/v2/llms?limit=100"

curl -s -H "x-api-key: $VECTARA_API_KEY" \
  "https://api.vectara.io/v2/generation_presets?limit=100"
```

## Bring Your Own LLM

`POST /v2/llms` registers a third-party model. The platform supports three `type`s, each with its own `auth` shape.

| `type` | For | Typical `auth` |
|---|---|---|
| `openai-compatible` | OpenAI, Anthropic Claude, Azure OpenAI, OpenRouter, on-prem vLLM/TGI, any chat-completions API | `header` (e.g. `x-api-key`, `Authorization: Bearer ...`) |
| `openai-responses` | OpenAI Responses API (`o1`, `o3`, future reasoning models) | `bearer` |
| `vertex-ai` | Google Cloud Vertex AI (Gemini family) | `api_key` or `service_account` |

### Common fields

| Field | Description |
|---|---|
| `type` | One of the three above. |
| `name` | Display name — this is what you reference from a preset's `llm_name` or query override. |
| `description` | Optional metadata. |
| `model` | Provider-specific model id (`claude-3-7-sonnet-20250219`, `gpt-4o-mini`, `gemini-2.5-flash`). |
| `uri` | API endpoint URL. |
| `auth` | See per-type shapes above. |
| `headers` | Optional extra HTTP headers. |
| `test_model_parameters` | Sent during validation. Most commonly `{ "max_tokens": 512 }`. |

### Example — register Claude 3.7 Sonnet (OpenAI-compatible)

```bash
curl -L -X POST 'https://api.vectara.io/v2/llms' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: '"$VECTARA_API_KEY" \
  -d '{
    "type": "openai-compatible",
    "name": "Claude 3.7 Sonnet",
    "description": "Anthropic Claude 3.7 Sonnet",
    "model": "claude-3-7-sonnet-20250219",
    "uri": "https://api.anthropic.com/v1/chat/completions",
    "auth": {
      "type": "header",
      "header": "x-api-key",
      "value": "sk-ant-..."
    },
    "test_model_parameters": { "max_tokens": 512 }
  }'
```

Response (201):

```json
{
  "id": "llm_520721844",
  "name": "Claude 3.7 Sonnet",
  "description": "Anthropic Claude 3.7 Sonnet",
  "enabled": true
}
```

### Example — Gemini 2.5 Pro via Vertex AI service account

```json
{
  "type": "vertex-ai",
  "name": "Gemini 2.5 Pro",
  "model": "gemini-2.5-pro",
  "uri": "https://aiplatform.googleapis.com/v1/projects/YOUR-PROJECT-ID/locations/us-central1",
  "auth": {
    "type": "service_account",
    "key_json": "{\"type\":\"service_account\",\"project_id\":\"YOUR-PROJECT\",\"private_key\":\"-----BEGIN PRIVATE KEY-----\\n...\\n-----END PRIVATE KEY-----\\n\"}"
  }
}
```

### Example — OpenAI o1 (Responses API)

```json
{
  "type": "openai-responses",
  "name": "o1-preview",
  "model": "o1-preview",
  "uri": "https://api.openai.com/v1/chat/completions",
  "auth": { "type": "bearer", "token": "sk-..." }
}
```

### Validation behavior

When you create an LLM, the platform issues a test request against the configured endpoint using `test_model_parameters`. If it fails (bad auth, unreachable URI, model id wrong), the LLM is rejected — you don't get an "enabled but broken" record. After creation, `GET /v2/llms` should show your LLM with `"enabled": true`; if not, fix the config and re-create.

### Wiring BYO into a query

Two paths:

1. **Wrap in a customer preset** (recommended for reuse):

   ```json
   POST /v2/generation_presets
   {
     "name": "claude-rag",
     "llm_name": "Claude 3.7 Sonnet",
     "prompt_template": "[ { \"role\": \"system\", \"content\": \"You are a careful analyst.\" }, { \"role\": \"user\", \"content\": \"Answer ${vectaraQuery} using only: #foreach ($qResult in $vectaraQueryResults) [$foreach.count] ${qResult.getText()} #end\" } ]",
     "max_tokens": 1500,
     "temperature": 0.2
   }
   ```

2. **Per-call override** (good for A/B tests):

   ```json
   "generation": {
     "generation_preset_name": "mockingbird-2.0",
     "model_parameters": { "llm_name": "Claude 3.7 Sonnet" }
   }
   ```

`llm_name` on a query overrides the preset's bound LLM. The preset's prompt template still applies — make sure it's compatible with the override LLM's role/turn rules.

## Generation inside `corpora_search` and `POST /v2/query`

The same `generation` object shape works in three places:

| Caller | Path |
|---|---|
| One-off query | `POST /v2/query` → top-level `generation` |
| Per-corpus query | `POST /v2/corpora/{corpus_key}/query` → top-level `generation` |
| Agent tool | inside `tool_configurations.<name>.query_configuration.generation` for a `corpora_search` tool |

Inside an agent tool, generation is **off by default**. Enable it explicitly:

```json
"search_kb": {
  "type": "corpora_search",
  "query_configuration": {
    "search": {
      "corpora": [{ "corpus_key": "support-kb" }],
      "limit": 15
    },
    "generation": {
      "enabled": true,
      "generation_preset_name": "rfi-assistant-mockingbird",
      "max_used_search_results": 8,
      "response_language": "eng"
    }
  }
}
```

The agent's reasoning model (`agent.model`) is separate from the corpus tool's summarizer LLM. The tool's `generation_preset_name` controls the *tool output summary*; the agent's `model` controls how the agent reasons over that output and other tool results.

## Table summarization with custom prompts (ingest-time)

When uploading documents that contain tables, you can attach a `table_extraction_config` that runs a custom prompt over each extracted table during ingest. Useful for domain-specific framing ("summarize this table as a finance analyst would", structured JSON output, etc.).

```json
{
  "extract_tables": true,
  "extractor": { "name": "gmft" },
  "generation": {
    "llm_name": "gpt-4o-mini",
    "prompt_template": "Adopt the perspective of a data analyst and summarize the table below:\n\n$vectaraTable.markdown()",
    "model_parameters": { "temperature": 0.7, "max_tokens": 1024 }
  }
}
```

### Table-specific Velocity variables

| Variable | Returns |
|---|---|
| `$vectaraTable` | The root extracted-table object. |
| `$vectaraTable.title()` | Table title. |
| `$vectaraTable.description()` | Description / caption. |
| `$vectaraTable.json()` | Table contents as JSON. |
| `$vectaraTable.markdown()` | Table contents as Markdown — preferred for LLM consumption. |

Upload with the config inline:

```bash
curl -L -X POST "https://api.vectara.io/v2/corpora/$CORPUS/upload_file" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -F 'file=@"/path/to/report.pdf"' \
  -F 'table_extraction_config={"extract_tables":true,"extractor":{"name":"gmft"}};type=application/json'
```

The `;type=application/json` suffix on the `-F` part is required — without it the multipart parser treats the config as a plain string and the request fails.

## Common Mistakes

- **Forgetting `$` on Velocity variables.** Writing `qResult.getText()` instead of `$qResult.getText()` renders the literal text `qResult.getText()` into the prompt.
- **Bare `${`/`}` adjacent to quotes inside JSON strings.** Backslash-escape any literal `"` you want inside `content`, and prefer `${var}` over `$var` whenever the next character could extend the identifier.
- **Hard-coding the LLM at query time instead of in a preset.** Don't ship `prompt_template` + `model_parameters.llm_name` + tuned penalties inline on every query — bundle them in a preset and reference the preset name.
- **Trying to replace or delete a platform preset.** `vectara-summary-*` and `mockingbird-*` are owned by the platform — `PUT` and `DELETE` return 403/404. Create a customer preset with a new name instead.
- **Using a custom LLM directly in `generation.llm_name`.** Custom LLMs aren't query-callable on their own; they have to be reached via a preset (or via the `model_parameters.llm_name` *override* on a query that still names a preset).
- **Setting `search.limit: 50` and expecting all 50 to reach the LLM.** They don't — `max_used_search_results` (default 25) caps what the prompt sees. Bump both, or accept that retrieval and generation budgets are independent.
- **Looping with the wrong variable name.** It's `$qResult in $vectaraQueryResults`, not `$result in $results` or `$r in $vectaraResults`. The platform binds those exact names.
- **Putting consecutive `user` or `assistant` turns in a Mockingbird custom prompt.** Mockingbird requires strict alternation and a `user` last message. Validate before saving.
- **Omitting `;type=application/json` from a multipart `table_extraction_config`.** The config is parsed as a string and the LLM call is skipped.
- **Mixing the two Velocity namespaces.** Query prompt templates see `$vectaraQuery` / `$vectaraQueryResults`; agent tool description templates see `$agent.*` / `$session.*` / `$currentDate`. Neither namespace is available in the other.

## References

### Concepts
- [Grounded Generation (RAG) overview](https://docs.vectara.com/docs/learn/grounded-generation/grounded-generation-overview.md)
- [Configure query summarization](https://docs.vectara.com/docs/learn/grounded-generation/configure-query-summarization.md)
- [Select a summarizer / generation presets](https://docs.vectara.com/docs/learn/grounded-generation/select-a-summarizer.md)
- [Model selection](https://docs.vectara.com/docs/learn/grounded-generation/model-selection.md)
- [Response languages](https://docs.vectara.com/docs/learn/grounded-generation/grounded-generation-response-languages.md)
- [Mockingbird 2 LLM](https://docs.vectara.com/docs/learn/mockingbird-llm.md)

### Prompts
- [Vectara Prompt Engine](https://docs.vectara.com/docs/prompts/vectara-prompt-engine.md)
- [Custom prompts with metadata](https://docs.vectara.com/docs/prompts/custom-prompts-with-metadata.md)
- [Generative prompts hub](https://docs.vectara.com/docs/generative-prompts.md)
- [Summarize tables with custom prompts](https://docs.vectara.com/docs/prompts/custom-prompt-templates-customization.md)
- [Apache Velocity 1.7 User Guide](https://velocity.apache.org/engine/1.7/user-guide.html)

### APIs
- [Create generation preset](https://docs.vectara.com/docs/api-reference/generation-preset-apis/create-generation-preset.md) — `POST /v2/generation_presets`
- [List generation presets](https://docs.vectara.com/docs/api-reference/generation-preset-apis/list-generation-presets.md) — `GET /v2/generation_presets`
- [Replace generation preset](https://docs.vectara.com/docs/api-reference/generation-preset-apis/replace-generation-preset.md) — `PUT /v2/generation_presets/{id}`
- [Delete generation preset](https://docs.vectara.com/docs/api-reference/generation-preset-apis/delete-generation-preset.md) — `DELETE /v2/generation_presets/{id}`
- [Bring Your Own LLM](https://docs.vectara.com/docs/search-and-retrieval/bring-your-own-llm.md)
- [Create LLM](https://docs.vectara.com/docs/api-reference/llm-apis/create-llm.md) — `POST /v2/llms`
- [List LLMs](https://docs.vectara.com/docs/api-reference/llm-apis/list-ll-ms.md) — `GET /v2/llms`
- [Query a Corpus](https://docs.vectara.com/docs/rest-api/query-corpus.md)
- [Supported language codes (proto)](https://github.com/vectara/protos/blob/main/common.proto#L10)
- [OpenAPI Spec](https://api.vectara.io/v2/openapi.json) — `generation_presets`, `llms`, `queries`
