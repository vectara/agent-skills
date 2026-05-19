---
name: vectara-queries
description: Build direct Vectara queries — single-corpus, multi-corpus, and GET search; hybrid (lexical+semantic) blending; metadata filters; rerankers (chain, knee, MMR, multilingual, UDF); context configuration; the optional `generation` block (presets, Velocity prompts, BYO LLM); citations. Use for retrieval embedded in existing apps, background RAG jobs, or for the matching `query_configuration` block inside an agent's `corpora_search` tool.
tags: [vectara, query, search, retrieval, rag, generation, reranker, filters, velocity, byo-llm]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Queries — Direct Search and Generation

You are helping a developer query Vectara corpora directly. This skill covers the `/v2/query` family of endpoints — both the retrieval surface (filters, hybrid search, rerankers) and the generation surface (presets, prompts, BYO LLM) that share the same request body.

## When to use this skill

- The task is a direct `POST /v2/query`, `POST /v2/corpora/{key}/query`, or `GET /v2/corpora/{key}/search` call — quick lookups, RAG embedded in an existing app, background jobs, internal tools.
- The developer is configuring an agent's `corpora_search` tool — the `query_configuration.search` and `query_configuration.generation` blocks use the **exact same shape** as `/v2/query`. Tune them here.
- Result quality is poor: missing the right document, near-duplicate results, stale results dominating.
- The developer needs filters (`doc.X` / `part.X`), reranking (`customer_reranker`, `mmr`, `chain`, `userfn`, knee), hybrid search (`lexical_interpolation`), citations, or a custom prompt template.
- The developer needs to register a third-party LLM (Claude, Gemini, OpenRouter, on-prem) and route generation through it.

**For chat-style RAG UX with multi-turn conversation, tool calls, or workflow logic, use `vectara-agents` instead.** Agents are the recommended surface for anything beyond a single request/response.

## Critical Rules

1. **API base URL** is `https://api.vectara.io/v2/`. Auth is `x-api-key: <key>` **or** `Authorization: Bearer <jwt>` — never both.
2. **Use `corpus_key` (string), not `corpus_id` (number).** `corpus_id` is deprecated.
3. **Three distinct endpoints — different body shapes.** Don't put `corpora[]` in a single-corpus body; don't omit `corpora[]` in the multi-corpus body. See the table below.
4. **`corpora` is an array on the multi-corpus endpoint.** Each element has its own `corpus_key`, `metadata_filter`, `lexical_interpolation`, `custom_dimensions`, `semantics`. `limit`, `context_configuration`, and `reranker` are top-level under `search`.
5. **`lexical_interpolation`:** `0.0` = pure semantic, `1.0` = pure keyword (BM25-equivalent). Hybrid sweet spot is `0.005–0.05` (default `0.025`). Reach for higher values only when an exact term isn't surviving tokenization — fix that at ingest first.
6. **`metadata_filter` only works on declared+indexed attributes.** Filtering on an undeclared field returns an error, not zero results. See `vectara-corpus-and-ingestion` for declaring `filter_attributes`.
7. **Rerankers use `type` as the discriminator.** Valid types: `customer_reranker`, `mmr`, `userfn`, `chain`, `none`. `limit` and `cutoff` live on each reranker stage, not on `search.limit` (which caps post-rerank).
8. **Cutoffs are only meaningful with neural rerankers.** Multilingual `customer_reranker` scores are normalized 0.0–1.0; raw hybrid/BM25/UDF scores are unbounded.
9. **Don't re-sort `search_results[]` by `score` client-side.** Each reranker rescores; the response order is the final order.
10. **`generation` is optional.** Omit for pure retrieval. When present, reference presets by name (`generation_preset_name`) instead of building them inline.
11. **`max_used_search_results` caps what the LLM sees**, independent of `search.limit` (which controls retrieval). Default `25`. Lower it first if a prompt fails to produce output.

## The three query endpoints

| Endpoint | Method | Body shape | Use when |
|---|---|---|---|
| `/v2/corpora/{corpus_key}/search` | `GET` | none (query in URL) | One-shot lookups. Only `query`, `limit`, `offset`, `save_history`, `intelligent_query_rewriting`. No filters, no reranker, no hybrid, no generation. |
| `/v2/corpora/{corpus_key}/query` | `POST` | `{query, search, generation?}` — `search` *without* `corpora[]` | Single-corpus, full tuning surface. |
| `/v2/query` | `POST` | `{query, search: {corpora: [...]}, generation?}` | Multi-corpus. Each corpus carries its own filter/hybrid setting; results interleave by score. |

### Simple GET search

```bash
curl -G "https://api.vectara.io/v2/corpora/my-corpus/search" \
  -H "x-api-key: $VECTARA_API_KEY" \
  --data-urlencode "query=What is the refund policy?" \
  --data-urlencode "limit=10"
```

### Single-corpus POST query

```bash
curl -X POST "https://api.vectara.io/v2/corpora/my-corpus/query" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the refund policy for enterprise plans?",
    "search": {
      "limit": 50,
      "metadata_filter": "doc.year = 2024 AND doc.status = '\''Published'\''",
      "lexical_interpolation": 0.025,
      "context_configuration": { "sentences_before": 2, "sentences_after": 2 },
      "reranker": {
        "type": "customer_reranker",
        "reranker_name": "Rerank_Multilingual_v1",
        "cutoff": 0.5,
        "limit": 10
      }
    },
    "generation": {
      "generation_preset_name": "mockingbird-2.0",
      "max_used_search_results": 8,
      "response_language": "eng"
    }
  }'
```

### Multi-corpus POST query

```json
{
  "query": "How do we handle expired tokens across services?",
  "search": {
    "corpora": [
      { "corpus_key": "support-articles",     "metadata_filter": "doc.product = 'auth-service'", "lexical_interpolation": 0.025 },
      { "corpus_key": "engineering-runbooks", "metadata_filter": "doc.team = 'platform'",        "lexical_interpolation": 0.1   }
    ],
    "limit": 25,
    "context_configuration": { "sentences_before": 2, "sentences_after": 2 },
    "reranker": {
      "type": "chain",
      "rerankers": [
        { "type": "customer_reranker", "reranker_name": "Rerank_Multilingual_v1", "cutoff": 0.5, "limit": 30 },
        { "type": "mmr", "diversity_bias": 0.3, "limit": 10 }
      ]
    }
  }
}
```

Each corpus gets its own filter and `lexical_interpolation`. `limit`, `context_configuration`, `reranker` operate on the merged result set. Use `request_corpora_index` or `corpus_key` on each result to tell origin.

## Hybrid search

`lexical_interpolation` (often called λ) blends BM25-style lexical scoring into the semantic embedding score. Lives per-corpus (multi-corpus call) or under `search` (single-corpus). Defaults to `0.025`.

| Query style | Suggested λ |
|---|---|
| Conceptual ("how does X work") | `0.0–0.025` |
| Mixed natural language with key terms | `0.025–0.1` |
| Identifier/codename heavy (`error E_42`, SKU `PN-998`) | `0.1–0.3` |
| Exact-string lookup | `0.3–0.6` |
| Pure keyword (≡ BM25) | `1.0` |

Query-string special syntax: `"blue shoes"` forces exact-phrase order; `Miss*` is a prefix match; `-Mississippi` excludes; `-Miss*` excludes a prefix.

## Filter expression syntax

Filters reference declared metadata at document level (`doc.<name>`) or part level (`part.<name>`).

| Operator | Example | Notes |
|---|---|---|
| `=`, `!=` | `doc.status = 'Published'` | String literals in single quotes. |
| `<`, `<=`, `>`, `>=` | `doc.year > 2024`, `doc.price <= 50.0` | Numeric. |
| `AND`, `OR`, `NOT` | `doc.status = 'Published' AND doc.year = 2024` | Parenthesize for clarity. |
| `IN (...)` | `doc.category IN ('Science', 'History')` | Multi-value membership. |
| Boolean literals | `part.is_title = true` | Unquoted. |
| Epoch / int date | `1609459200 < doc.pub_epoch` | Store dates as epoch ints or `YYYYMMDD` ints, not strings. |

- **`doc.X`** — attribute on the whole document (status, year, author). Use for things that don't change across the document.
- **`part.X`** — attribute on a specific chunk (`part.lang = 'eng'`, `part.is_title = true`). Vectara auto-populates `part.lang` for multilingual content.
- **Escape a single quote** inside a string by doubling it: `'Vectara''s docs'`.
- **Discover legal values:** `GET /v2/corpora/{corpus_key}/filter_attribute_stats` returns distinct values and counts per declared attribute. Useful when building filters programmatically.
- **Auto-extract from natural language:** Set `intelligent_query_rewriting: true` at the request top level (Tech Preview). The response includes `rewritten_queries[].filter_extraction` showing what was extracted.

## Reranker taxonomy

| `type` | Use when | Key params |
|---|---|---|
| `customer_reranker` | Always — Slingshot, the multilingual neural reranker. The strongest single relevance signal Vectara ships. | `reranker_name: "Rerank_Multilingual_v1"`, `cutoff` (start at 0.5), `limit`, `include_context` (default `true`), `instructions` |
| `mmr` | After the neural reranker, to dedupe near-duplicate passages. | `diversity_bias` (0.0–1.0, start at 0.3), `limit`, `cutoff` |
| `userfn` | Business signal: recency, popularity, geographic proximity, promoted flags. Also: `user_function: "knee()"` for auto-cutoff. | `user_function` (string expression), `limit`, `cutoff` |
| `chain` | The recommended production setup. Combine 2–3 stages: neural → MMR → optional UDF. | `rerankers: [...]` (up to 50) |
| `none` | Debugging: see raw pre-rerank results to verify retrieval is sound. | `limit` |

`reranker_id` / `rnk_272725719` are deprecated — always use `type: "customer_reranker"` with `reranker_name`.

### customer_reranker (Slingshot multilingual)

```json
{
  "reranker": {
    "type": "customer_reranker",
    "reranker_name": "Rerank_Multilingual_v1",
    "cutoff": 0.5,
    "limit": 50,
    "include_context": true,
    "instructions": "Prefer policy documents over marketing material."
  }
}
```

Normalized 0.0–1.0 scores. `cutoff: 0.5` is the recommended default; below `0.3` noise leaks in, above `0.7` zero-result responses on real queries become common. `instructions` is only honored by instruction-following rerankers.

### chain — production pattern

```json
{
  "reranker": {
    "type": "chain",
    "rerankers": [
      { "type": "customer_reranker", "reranker_name": "Rerank_Multilingual_v1", "cutoff": 0.5, "limit": 50 },
      { "type": "mmr", "diversity_bias": 0.3, "limit": 10 },
      { "type": "userfn", "user_function": "if (hours(now() - iso_datetime_parse(get('$.document_metadata.created_at'))) > 8760) get('$.score') * 0.5 else get('$.score')", "limit": 10 }
    ]
  }
}
```

Stages run top-down. Each stage's `limit` caps input to the next. Each stage can have its own `cutoff` (runs before `limit`).

### knee — auto-detect score drop-off

```json
{
  "reranker": {
    "type": "chain",
    "rerankers": [
      { "type": "customer_reranker", "reranker_name": "Rerank_Multilingual_v1" },
      { "type": "userfn", "user_function": "knee()", "cutoff": 0.5 }
    ]
  }
}
```

Tunables on the function call: `sensitivity` (default 0.5) — how sharply the score must drop to mark a cutoff; `early_bias` (default 0.2) — prefer cutoffs closer to the top. High+high narrows aggressively; low+low keeps broad.

### userfn — UDF language

Small custom language. Types: number, string, boolean, datetime, duration. Read values via `get('$.path')` JSONPath against the result object (`$.score`, `$.document_metadata.*`, `$.part_metadata.*`, `$.document_id`, `$.text`).

| Pattern | Expression |
|---|---|
| Multiply by document-level boost | `get('$.score') * get('$.document_metadata.boost')` |
| Boost French content by 60% | `if (get('$.part_metadata.lang') == 'fra') get('$.score') * 1.6 else get('$.score')` |
| Drop results scoring under 0.5 | `if (get('$.score') < 0.5) null else get('$.score')` |
| Halve stale results (>1 year) | `if (hours(now() - iso_datetime_parse(get('$.document_metadata.created_at'))) > 8760) get('$.score') * 0.5 else get('$.score')` |
| Sort by price ascending, missing-last | `get('$.document_metadata.price', -999999)` |

Functions: `now()`, `iso_datetime_parse(str)`, `datetime_parse(str, fmt)`, `to_unix_timestamp(dt)`, `seconds/minutes/hours(dur)`, `abs`, `power`, `min`, `max`, `sqrt`, `log`, `log10`, `ln`. `get('$.path', default)` returns the default when null. **Returning `null` removes the result** before later stages see it — the only reranker with that capability.

### Discover installed rerankers

```bash
curl -s -H "x-api-key: $VECTARA_API_KEY" "https://api.vectara.io/v2/rerankers"
```

## Context configuration

`context_configuration` (under `search`) is the biggest lever on response size and readability.

| Field | Notes |
|---|---|
| `sentences_before` / `sentences_after` | int (default `0`). Preferred. |
| `characters_before` / `characters_after` | Fallback for content without sentence terminators (code, CSV). **Ignored if sentence-based is also set.** |
| `start_tag` / `end_tag` | Wraps the matched part for UI highlighting (e.g. `<mark>...</mark>`). |

Sizing: long parts (full sections) — drop to `0`, duplication is wasted bytes. Short parts (FAQs, captions) — bump to 3–5 so each result is self-contained.

## Response shape (POST endpoints)

```json
{
  "summary": "Generated answer text — only present if generation enabled.",
  "search_results": [
    {
      "text": "<mark>The matched part,</mark> with surrounding sentences.",
      "score": 0.87,
      "document_id": "doc_4f3a",
      "document_metadata": { "title": "Refund Policy", "year": 2024 },
      "part_metadata": { "section": "policies", "lang": "eng", "offset": 1086, "len": 188 },
      "request_corpora_index": 0,
      "corpus_key": "support-articles",
      "table": null,
      "image": null
    }
  ],
  "factual_consistency_score": 0.97,
  "warnings": [],
  "rewritten_queries": []
}
```

- `text` is wrapped by `start_tag`/`end_tag` and padded by `context_configuration`.
- `score` is final-stage. Don't re-sort by it.
- `document_id` is opaque and stable — use as a cache key or for `GET /v2/corpora/{key}/documents/{id}`.
- `request_corpora_index` / `corpus_key` identify origin in multi-corpus responses (results interleave by score).
- `table` / `image` populated when the part is one. Tables aren't split and can be many KB.
- `warnings[]`: non-fatal — e.g. `exceeded_max_input_length_fcs`, `fcs_language_not_supported`.

### Pagination

`search.offset` walks the post-rerank set. `{"limit": 10, "offset": 20}` returns results 21–30. Deep pagination with a heavy reranker chain re-runs the full chain each call — cache client-side when possible.

## Generation block

When you want a synthesized answer over retrieved results, add a `generation` block.

```json
"generation": {
  "generation_preset_name": "mockingbird-2.0",
  "max_used_search_results": 8,
  "response_language": "eng"
}
```

### Preset selection — most common

| Preset name | Backing LLM | When to use |
|---|---|---|
| `mockingbird-2.0` | Mockingbird 2.0 (Vectara) | **Default for RAG.** Best citation accuracy, multilingual, lowest hallucination rate when paired with HHEM. |
| `vectara-summary-ext-24-05-med-omni` | `gpt-4o` | Long-form / creative synthesis. |
| `vectara-summary-ext-24-05-sml` | `gpt-3.5-turbo` | High-volume, cost-sensitive. |
| `vectara-summary-table-query-ext-dec-2024-gpt-4o` | `gpt-4o` | Table-heavy corpora. |

Discover everything available on the tenant:

```bash
curl -s -H "x-api-key: $VECTARA_API_KEY" "https://api.vectara.io/v2/generation_presets?limit=100"
curl -s -H "x-api-key: $VECTARA_API_KEY" "https://api.vectara.io/v2/llms?limit=100"
```

### Per-call overrides

Every preset field except `llm_name` can be overridden in the query's `generation` block. To override the LLM without forking the preset:

```json
"generation": {
  "generation_preset_name": "mockingbird-2.0",
  "model_parameters": {
    "llm_name": "Claude 3.7 Sonnet",
    "temperature": 0.0,
    "max_tokens": 800
  }
}
```

### Response languages

Set `generation.response_language`. Accepts ISO 639-1 (`en`, `fr`) or 639-3 (`eng`, `fra`). `"auto"` detects from the query. Mockingbird 2.0 is fully cross-lingual across English, Spanish, French, Arabic, Chinese, Japanese, Korean — query in one language, retrieve in another, summarize in a third. Factual consistency scoring (`enable_factual_consistency_score`) is only computed for `eng`, `deu`, `fra` — see `vectara-hallucination-and-eval`.

## Custom prompts (Apache Velocity)

When the platform presets aren't enough, create a customer preset with a `prompt_template`. Templates render to a JSON array of OpenAI-style messages — the *output* of the template must be valid JSON.

### Variables

| Variable | Description |
|---|---|
| `$vectaraQuery` | The end-user's query string. |
| `$vectaraQueryResults` | The array of retrieved results. Iterate with `#foreach`. |
| `$vectaraLangCode` / `$vectaraLangName` | ISO 639-3 code / readable name of the requested response language. |
| `$vectaraIdxWord[i]` | 0-indexed ordinal word: `first`, `second`, ..., `tenth`. |
| `$qResult.getText()` | Result snippet text. **The canonical loop variable name is `$qResult`** — examples, defaults, and Mockingbird's built-in prompt all use it. |
| `$qResult.docMetadata().get("title")` | Read a document-level metadata field. |
| `$qResult.partMetadata().get("page")` | Read a part-level metadata field. |
| `$qResult.docMetadata().present()` / `.partMetadata().present()` | `true` if metadata exists — guard with `#if`. |

### Create a customer preset

```bash
curl -X POST "https://api.vectara.io/v2/generation_presets" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rfi-assistant",
    "llm_name": "mockingbird-2.0",
    "prompt_template": "[ { \"role\": \"system\", \"content\": \"You are an RFI answering assistant. Use only information from the provided results. Cite using [number] notation.\" }, #foreach ($qResult in $vectaraQueryResults) { \"role\": \"user\", \"content\": \"Result $foreach.count from \\\"$qResult.docMetadata().get('"'"'title'"'"')\\\":\" }, { \"role\": \"assistant\", \"content\": \"${qResult.getText()}\" }, #end { \"role\": \"user\", \"content\": \"Answer ${vectaraQuery} in ${vectaraLangName}. If two results conflict, prefer the most recent answerDate.\" } ]",
    "max_used_search_results": 8,
    "max_tokens": 1200,
    "temperature": 0.2
  }'
```

Reference it via `"generation_preset_name": "rfi-assistant"`. See [Vectara Prompt Engine](https://docs.vectara.com/docs/prompts/vectara-prompt-engine.md) for the full Velocity surface.

### Mockingbird custom-prompt rules

Mockingbird requires: `system` allowed once at start; `user` and `assistant` must alternate; last message must be `user`. Validate before saving — broken prompts fail at query time.

## Bring Your Own LLM

`POST /v2/llms` registers a third-party model. Three `type`s, each with its own `auth` shape:

| `type` | For | Typical `auth` |
|---|---|---|
| `openai-compatible` | OpenAI, Anthropic Claude, Azure OpenAI, OpenRouter, on-prem vLLM/TGI | `header` (e.g. `x-api-key`, `Authorization: Bearer`) |
| `openai-responses` | OpenAI Responses API (`o1`, `o3`, future reasoning) | `bearer` |
| `vertex-ai` | Google Cloud Vertex AI (Gemini family) | `api_key` or `service_account` |

```bash
curl -X POST "https://api.vectara.io/v2/llms" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "openai-compatible",
    "name": "Claude 3.7 Sonnet",
    "model": "claude-3-7-sonnet-20250219",
    "uri": "https://api.anthropic.com/v1/chat/completions",
    "auth": { "type": "header", "header": "x-api-key", "value": "sk-ant-..." },
    "test_model_parameters": { "max_tokens": 512 }
  }'
```

Vectara test-calls the endpoint during registration. If it fails (bad auth, wrong URI/model), the LLM is rejected — no "enabled but broken" records.

**A custom LLM cannot be referenced directly from a query.** Wire it via a preset (`llm_name` in `POST /v2/generation_presets`) or override via `generation.model_parameters.llm_name`.

## Citations

When generation is enabled, the platform inserts `[N]` markers into `summary`, where `N` is a 1-indexed reference into `search_results[]`. Numbers are assigned in order of appearance in the summary, not by relevance score.

Configure via `generation.citations`:

```json
"citations": {
  "style": "markdown",
  "url_pattern": "https://docs.example.com/{doc.id}",
  "text_pattern": "{doc.title}"
}
```

| `style` | Output |
|---|---|
| `numeric` (default) | `[1]`, `[2]`, ... |
| `none` | Stripped from summary text. |
| `html` | `<a href="<url_pattern>"><text_pattern></a>` |
| `markdown` | `[<text_pattern>](<url_pattern>)` |

`url_pattern` / `text_pattern` accept `{doc.<field>}` and `{part.<field>}` placeholders that interpolate from result metadata.

## Fuzzy metadata search

Different endpoint. `POST /v2/corpora/{corpus_key}/metadata_query` does **typo-tolerant matching against metadata fields** (titles, categories, SKUs) — *not* against document text. Tech Preview.

```json
{
  "level": "document",
  "queries": [
    { "field": "title",    "query": "lease agreement", "weight": 2.0 },
    { "field": "category", "query": "contract",        "weight": 1.0 }
  ],
  "metadata_filter": "doc.status = 'Active'",
  "limit": 10
}
```

Response is a flat list of documents (`doc_id`, `score`, `metadata`) — not the regular `search_results[]` shape. Use this for "find me the SLA doc even if I type Servce Levl Agrement," not for semantic search.

## Tuning guidance

1. **Ingest is the heart of retrieval quality.** No query knob recovers information lost at ingest. Declare `filter_attributes` early. Generate real descriptions for tables and images (those descriptions are what gets embedded). See `vectara-corpus-and-ingestion`.
2. **Start with a strong default body.** 50 candidates, λ `0.025`, 2 sentences context before/after, chain reranker: `customer_reranker` (cutoff `0.5`, limit `50`) → `mmr` (diversity_bias `0.3`, limit `10`).
3. **Filter before reranking.** Filters are the highest-precision lever — every constraint the user's intent gives you (year, tenant, product, type) should become a `metadata_filter` clause.
4. **Tune λ by query type, not intuition.** See the hybrid-search table above. If you reach for λ > 0.5, fix tokenization at ingest instead.
5. **Iterate with an eval set.** 20–50 real queries with the right-answer documents labeled. Run with `reranker: {"type": "none"}` first — if the right doc isn't in the top-50 there, no reranker will save you.
6. **Inspect history.** Set `save_history: true` and `GET /v2/queries/{query_id}` returns per-stage spans (pre-rerank, post-rerank, rewritten queries).

## Common Mistakes

- **Using `corpus_id` (number) instead of `corpus_key` (string).** Especially common when porting from v1.
- **Confusing the three endpoints.** `corpora[]` in a body posted to `/v2/corpora/{key}/query` fails — that endpoint takes the corpus from the path. Conversely, omitting `corpora[]` against `/v2/query` fails because multi-corpus requires it.
- **Filtering on a non-declared field.** Returns an error, not zero results.
- **Setting both `sentences_before` and `characters_before`.** Sentences win; characters are silently ignored.
- **Treating `search.limit` as the candidate cap for the neural reranker.** It caps post-rerank only. To rerank 100 candidates, set `limit: 100` on the reranker stage.
- **Using `cutoff` against raw hybrid/`mmr`/`userfn` outputs and expecting `0.5` to mean "above half."** Only `customer_reranker` produces normalized 0.0–1.0 scores.
- **Re-sorting `search_results[]` by `score` client-side.** Each reranker rescored; the response order is final.
- **Cranking `lexical_interpolation` above 0.5 to force an exact-term hit.** Fix tokenization at ingest instead.
- **Using `reranker_id` / `rnk_272725719`.** Deprecated. Use `type: "customer_reranker"` + `reranker_name`.
- **Assuming multi-corpus results are grouped by corpus.** They interleave by score. Use `request_corpora_index` / `corpus_key` if origin matters.
- **Setting `search.limit: 50` and expecting all 50 to reach the LLM.** `max_used_search_results` (default 25) caps what the prompt sees. Independent budget.
- **Hard-coding the LLM at query time instead of in a preset.** Bundle prompt + LLM + parameters in `POST /v2/generation_presets` and reference by name.
- **Using a custom LLM directly as `generation.llm_name`.** Custom LLMs reach queries through a preset (or `generation.model_parameters.llm_name` override on a preset-named query).
- **Forgetting `$` on Velocity variables** — `qResult.getText()` (no `$`) renders as literal text in the prompt.
- **Looping with the wrong variable name.** It's `$qResult in $vectaraQueryResults`. The platform binds those exact names.
- **Putting consecutive `user` or `assistant` turns in a Mockingbird custom prompt.** Mockingbird requires strict alternation and a final `user` message.
- **Posting fuzzy-match needs against `/query`.** Use `/metadata_query` — different endpoint, different shape.

## References

### Concepts
- [Search and retrieval overview](https://docs.vectara.com/docs/search-and-retrieval/search-and-retrieval.md)
- [Search quick start](https://docs.vectara.com/docs/search-and-retrieval/search-quick-start.md)
- [Hybrid search](https://docs.vectara.com/docs/search-and-retrieval/hybrid-search.md)
- [Filters](https://docs.vectara.com/docs/search-and-retrieval/filters.md)
- [Citations](https://docs.vectara.com/docs/search-and-retrieval/citations.md)
- [Fuzzy metadata search](https://docs.vectara.com/docs/search-and-retrieval/fuzzy-metadata-search.md)
- [Tune retrieval](https://docs.vectara.com/docs/search-and-retrieval/tune-retrieval.md)
- [Grounded Generation (RAG) overview](https://docs.vectara.com/docs/learn/grounded-generation/grounded-generation-overview.md)
- [Configure query summarization](https://docs.vectara.com/docs/learn/grounded-generation/configure-query-summarization.md)
- [Model selection](https://docs.vectara.com/docs/learn/grounded-generation/model-selection.md)
- [Response languages](https://docs.vectara.com/docs/learn/grounded-generation/grounded-generation-response-languages.md)
- [Mockingbird 2 LLM](https://docs.vectara.com/docs/learn/mockingbird-llm.md)

### Rerankers
- [Reranking overview](https://docs.vectara.com/docs/search-and-retrieval/reranking.md)
- [Chain reranker](https://docs.vectara.com/docs/search-and-retrieval/rerankers/chain-reranker.md)
- [Knee reranker](https://docs.vectara.com/docs/search-and-retrieval/rerankers/knee-reranking.md)
- [MMR reranker](https://docs.vectara.com/docs/search-and-retrieval/rerankers/mmr-reranker.md)
- [Vectara multilingual reranker (Slingshot)](https://docs.vectara.com/docs/search-and-retrieval/rerankers/vectara-multi-lingual-reranker.md)
- [User Defined Function reranker](https://docs.vectara.com/docs/search-and-retrieval/rerankers/user-defined-function-reranker.md)
- [Limits and cutoffs](https://docs.vectara.com/docs/search-and-retrieval/rerankers/limits-and-cutoffs.md)

### Prompts
- [Vectara Prompt Engine](https://docs.vectara.com/docs/prompts/vectara-prompt-engine.md)
- [Custom prompts with metadata](https://docs.vectara.com/docs/prompts/custom-prompts-with-metadata.md)
- [Apache Velocity 1.7 User Guide](https://velocity.apache.org/engine/1.7/user-guide.html)

### REST API
- [Multi-corpus query (POST)](https://docs.vectara.com/docs/rest-api/query.md)
- [Single-corpus advanced query (POST)](https://docs.vectara.com/docs/rest-api/query-corpus.md)
- [Single-corpus simple search (GET)](https://docs.vectara.com/docs/rest-api/search-corpus.md)
- [Metadata fuzzy query (POST)](https://docs.vectara.com/docs/rest-api/query-metadata.md)
- [List rerankers](https://docs.vectara.com/docs/rest-api/list-rerankers.md)
- [Create generation preset](https://docs.vectara.com/docs/rest-api/create-generation-preset.md)
- [List generation presets](https://docs.vectara.com/docs/rest-api/list-generation-presets.md)
- [Bring Your Own LLM](https://docs.vectara.com/docs/search-and-retrieval/bring-your-own-llm.md)
- [Create LLM](https://docs.vectara.com/docs/rest-api/create-llm.md)
- [List LLMs](https://docs.vectara.com/docs/rest-api/list-ll-ms.md)
- [OpenAPI spec](https://api.vectara.io/v2/openapi.json)
- [Full docs index](https://docs.vectara.com/llms.txt)

### Related skills
- `vectara-agents` — for chat-style RAG, tool-using agents, multi-turn conversation state.
- `vectara-corpus-and-ingestion` — declaring `filter_attributes` so `metadata_filter` works.
- `vectara-hallucination-and-eval` — `enable_factual_consistency_score`, HHEM, Open RAG Eval.
