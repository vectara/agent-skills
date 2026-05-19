---
name: vectara-search-and-retrieval
description: Build Vectara queries — single-corpus, multi-corpus, and GET search endpoints; hybrid (lexical+semantic) blending; metadata filters; rerankers (chain, knee, MMR, multilingual, UDF); context configuration; citations; fuzzy metadata search; tune retrieval.
tags: [vectara, search, retrieval, rag, hybrid-search, reranker, filters, metadata, mmr, slingshot, udf]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara — Search and Retrieval

You are helping a developer build retrieval against the Vectara platform. This skill covers the retrieval surface: the three query endpoints, hybrid search, filters, context configuration, rerankers, citations, fuzzy metadata search, and tuning. Generation (`summary`, `prompt_template`, `model_parameters`) is a separate concern handled by the `vectara-generation-and-prompts` skill — this skill mentions the `generation` block when it interacts with retrieval, but does not deep-dive its parameters.

## When to use this skill

- The developer is building any direct call against `/v2/query`, `/v2/corpora/{corpus_key}/query`, or `/v2/corpora/{corpus_key}/search`.
- The developer is configuring the `query_configuration.search` block of a `corpora_search` agent tool — same JSON shape, same tuning concerns.
- Result quality is poor: missing the right document, too many duplicates, stale results dominating, off-language results bleeding in.
- The developer needs filters (`doc.X`, `part.X`), reranking (`customer_reranker`, `mmr`, `chain`, `userfn`, knee), hybrid search (`lexical_interpolation`), or citation formatting.
- The developer needs fuzzy matching against metadata fields (typo-tolerant title/category/SKU lookup) via `/v2/corpora/{corpus_key}/metadata_query`.

If the task is "create an agent" or "wire a tool," use `vectara-agents` first; come back here when you actually need to tune what the search returns. If the task is "control how the summary is written," use `vectara-generation-and-prompts`.

## Documentation access

Append `.md` to any docs.vectara.com URL to get the page as markdown — e.g. `https://docs.vectara.com/docs/search-and-retrieval/hybrid-search.md`. The OpenAPI spec lives at `https://api.vectara.io/v2/openapi.json`. The full doc index is at `https://docs.vectara.com/llms.txt`.

## Critical Rules

1. **API base URL** is `https://api.vectara.io/v2/`. Auth is `x-api-key: <key>` or `Authorization: Bearer <jwt>` — never both.
2. **Use `corpus_key` (string), not `corpus_id` (number).** The `corpus_key` is the user-provided slug created with the corpus; `corpus_id` is deprecated.
3. **Three distinct query endpoints** — choose the one that matches the shape of the call. They take different bodies; don't try to use a multi-corpus `corpora[]` array against a single-corpus endpoint, and don't put a path-level `corpus_key` body field into a multi-corpus call.
   - `GET /v2/corpora/{corpus_key}/search` — quick lookups, query in the URL, no body.
   - `POST /v2/corpora/{corpus_key}/query` — single-corpus, full tuning.
   - `POST /v2/query` — multi-corpus, full tuning, takes `search.corpora[]`.
4. **`corpora` is an array on the multi-corpus endpoint.** Each element has its own `corpus_key`, `metadata_filter`, `lexical_interpolation`, `custom_dimensions`, `semantics`. `limit`, `offset`, `context_configuration`, and `reranker` are top-level under `search` and apply to the merged result set.
5. **`lexical_interpolation` semantics:** `0.0` = pure semantic, `1.0` = pure keyword (BM25-equivalent). The hybrid sweet spot is `0.005–0.05` (default `0.025`). Reach for higher values only when the term you care about isn't surviving the embedder's tokenization — fix it at ingest first.
6. **`metadata_filter` only works on declared+indexed attributes.** A filter against a field that isn't declared on the corpus as a `filter_attribute` returns an error, not zero results. Declare with `PUT /v2/corpora/{corpus_key}/replace_filter_attributes` and reindex.
7. **`context_configuration` controls the snippet size returned per result** — sentences before/after the matched part, plus optional `start_tag`/`end_tag` for highlighting. You can use sentence-based **or** character-based bounds, never both. If both are set, sentences win.
8. **Rerankers use `type` as the discriminator.** Valid types: `customer_reranker`, `mmr`, `userfn`, `chain`, `none`. The shape of the body varies per type — `customer_reranker` takes `reranker_name`, `mmr` takes `diversity_bias`, `userfn` takes `user_function`, `chain` takes `rerankers[]`.
9. **`search.limit` caps the post-rerank count.** To control how many candidates the neural reranker sees, set `limit` *on the reranker stage*, not on `search`. Otherwise you may rerank fewer candidates than you think.
10. **Cutoffs are only meaningful with neural rerankers.** Multilingual reranker scores are normalized 0.0–1.0, so `cutoff: 0.5` is a real threshold. With raw hybrid or BM25 scores, cutoffs are unreliable — scores are unbounded.
11. **Don't re-sort the response by `score`.** Each reranker stage rescores; the final order in `search_results[]` is the order — scores are only comparable within a single response.
12. **The `generation` block is optional.** Omit it for pure retrieval. When present, retrieval still runs first; `generation` only controls the summary. See the `vectara-generation-and-prompts` skill for the full surface.

## The three query endpoints

| Endpoint | Method | Body shape | Use when |
|---|---|---|---|
| `/v2/corpora/{corpus_key}/search` | `GET` | none (query in URL) | One-shot lookups, debugging, when you only need `query` + `limit` + `offset`. No filters, no reranker, no hybrid control. |
| `/v2/corpora/{corpus_key}/query` | `POST` | `{query, search, generation?}` with `search` *without* `corpora[]` | Single-corpus retrieval with the full tuning surface (filter, hybrid, reranker, context config). |
| `/v2/query` | `POST` | `{query, search: {corpora: [...]}, generation?}` | Multi-corpus retrieval. Each corpus can carry its own filter/hybrid setting; results interleave by score. |

### Simple GET search

```bash
curl -G "https://api.vectara.io/v2/corpora/my-corpus/search" \
  -H "x-api-key: $VECTARA_API_KEY" \
  --data-urlencode "query=What is the refund policy?" \
  --data-urlencode "limit=10"
```

Only `query`, `limit`, `offset`, `save_history`, and `intelligent_query_rewriting` are accepted on the GET form. Need a filter or reranker? Use the POST form.

### Single-corpus POST query

```bash
curl -X POST "https://api.vectara.io/v2/corpora/my-corpus/query" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the refund policy for enterprise plans?",
    "search": {
      "limit": 50,
      "offset": 0,
      "metadata_filter": "doc.year = 2024 AND doc.status = '\''Published'\''",
      "lexical_interpolation": 0.025,
      "context_configuration": {
        "sentences_before": 2,
        "sentences_after": 2,
        "start_tag": "<mark>",
        "end_tag": "</mark>"
      },
      "reranker": {
        "type": "customer_reranker",
        "reranker_name": "Rerank_Multilingual_v1",
        "cutoff": 0.5,
        "limit": 10
      }
    }
  }'
```

### Multi-corpus POST query

```json
{
  "query": "How do we handle expired tokens across services?",
  "search": {
    "corpora": [
      {
        "corpus_key": "support-articles",
        "metadata_filter": "doc.product = 'auth-service'",
        "lexical_interpolation": 0.025
      },
      {
        "corpus_key": "engineering-runbooks",
        "metadata_filter": "doc.team = 'platform'",
        "lexical_interpolation": 0.1
      }
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

Each corpus gets its own `metadata_filter` and `lexical_interpolation`. The `limit`, `context_configuration`, and `reranker` operate on the merged result set. Use `request_corpora_index` or `corpus_key` on each result to tell origin.

## Hybrid search

`lexical_interpolation` (often called λ or lambda) blends BM25-style lexical scoring into the semantic embedding score. It lives per-corpus (multi-corpus call) or directly under `search` (single-corpus call). Defaults to `0.025`.

| Query style | Suggested λ | Why |
|---|---|---|
| Conceptual ("how does X work") | `0.0–0.025` | Embedding already captures meaning; keyword bias just noise. |
| Mixed natural language with key terms | `0.025–0.1` | Light keyword sprinkle helps surface terms the embedder may underweight. |
| Identifier or codename heavy (`error E_42`, SKU `PN-998`) | `0.1–0.3` | The exact token is what matters; embeddings blur it. |
| Exact-string lookup | `0.3–0.6` | You're mostly doing BM25. Consider whether retrieval is even the right tool here. |
| Pure keyword (BM25 equivalent) | `1.0` | Vectara documents this — `1.0` ≡ traditional BM25. |

**Real example — bias slightly toward keywords for a query that contains an SKU:**

```json
{
  "query": "Replacement firmware for PN-998 enterprise switch",
  "search": {
    "corpora": [
      { "corpus_key": "product-docs", "lexical_interpolation": 0.15 }
    ],
    "limit": 25
  }
}
```

Vectara also gives the query string itself special handling: `"blue shoes"` (quoted) forces exact-phrase order, `Miss*` is a prefix match, `-Mississippi` excludes results referencing the term, and `-Miss*` excludes both Mississippi and Missouri.

## Filter expression syntax

Filters reference declared metadata at either document level (`doc.<name>`) or part level (`part.<name>`). Filtering on an undeclared field returns an error.

| Operator | Example | Notes |
|---|---|---|
| `=`, `!=` | `doc.status = 'Published'`, `doc.status != 'Draft'` | String literals in single quotes. |
| `<`, `<=`, `>`, `>=` | `doc.year > 2024`, `doc.price <= 50.0` | Numeric. |
| `AND`, `OR`, `NOT` | `doc.status = 'Published' AND doc.year = 2024` | Standard logical. Parenthesize for clarity: `NOT (doc.author = 'John Doe')`. |
| `IN (...)` | `doc.category IN ('Science', 'History')` | Multi-value membership. |
| Boolean literals | `part.is_title = true` | `true` / `false`, unquoted. |
| Date / epoch comparison | `1609459200 < doc.pub_epoch AND doc.pub_epoch < 1640995200` | Store dates as epoch ints or as `YYYYMMDD` ints — not strings — for reliable comparisons. |

### `doc.X` vs `part.X`

- **`doc.X`** — attribute that lives on the whole document. Indexed at document level. Use it for things that don't change across a document: status, year, author, jurisdiction.
- **`part.X`** — attribute that lives on a specific chunk. Use it for things that vary within a document: language (`part.lang = 'eng'`), section type (`part.is_title = true`), per-chunk tags. Vectara auto-populates `part.lang` for multilingual content.

### Escaping

String literals are single-quoted. To include a literal single quote inside a string, double it: `'Vectara''s docs'`. URLs and special characters need no other escaping inside a string literal.

### Discovering legal filter values

`GET /v2/corpora/{corpus_key}/filter_attribute_stats` returns distinct values and counts for each declared filter attribute. Useful when constructing filters programmatically from a UI.

### Auto-extracting filters from natural language

Set `intelligent_query_rewriting: true` at the request top level (Tech Preview) and the platform attempts to extract a `metadata_filter` from the query text. The response includes `rewritten_queries[].filter_extraction` showing what was extracted. Useful for low-effort filter coverage; for high-precision use cases, write filters explicitly.

## Reranker taxonomy

| `type` | Use when | Key params | Notes |
|---|---|---|---|
| `customer_reranker` | Always — this is Slingshot, the multilingual neural reranker. The strongest single relevance signal Vectara ships. | `reranker_name: "Rerank_Multilingual_v1"`, `cutoff` (start at 0.5), `limit`, `include_context` (default `true`), `instructions` | Normalized 0.0–1.0 scores make cutoffs meaningful. `reranker_id` and `rnk_272725719` are deprecated — use `reranker_name`. |
| `mmr` | After the neural reranker, to deduplicate near-duplicate passages. | `diversity_bias` (0.0–1.0, start at 0.3), `limit`, `cutoff` | Higher bias = more diverse and less relevance-driven. Without MMR you routinely get back 5 near-duplicates of the highest-scoring chunk. |
| `userfn` | Apply business signal: recency, popularity, geographic proximity, promoted flags. Also: knee filtering via `user_function: "knee()"`. | `user_function` (string expression), `limit`, `cutoff` | The only reranker that can return `null` to drop a result entirely. Null-removal happens before `limit` and before the next chain stage. |
| `chain` | The recommended production setup. Combine 2–3 stages: neural → MMR → optional UDF. | `rerankers: [...]` (up to 50) | Each stage's `limit` caps input to the next. Each stage can have its own `cutoff`. |
| `none` | Debugging: see raw pre-rerank results to verify retrieval itself is sound. | `limit` | If the right document isn't in the top 50 here, no reranker will save you — fix ingest, filters, or λ first. |

### `customer_reranker` (Slingshot multilingual)

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

`cutoff: 0.5` is Vectara's recommended default. Below `0.3`, noise leaks in; above `0.7`, zero-result responses on real queries become common. `instructions` is only honored by instruction-following rerankers. `include_context: true` makes the reranker score the snippet with its surrounding `context_configuration` text — usually what you want.

### `mmr`

```json
{ "reranker": { "type": "mmr", "diversity_bias": 0.3, "limit": 10 } }
```

`diversity_bias` 0.2 emphasizes relevance, 0.4–0.5 emphasizes variety. Common pattern: chain it after `customer_reranker` so neural relevance does the heavy lifting and MMR removes duplicates.

### `chain`

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

Stages run top-down. The neural reranker rescores against the merged candidate set and applies the 0.5 cutoff. MMR then takes those 50, deduplicates, returns 10. The UDF halves the score of anything older than ~1 year (8760 hours) and returns the top 10.

### Knee reranking

The knee reranker is a special UDF that auto-detects the natural drop-off point in the score distribution. Use it after Slingshot when fixed cutoffs are too coarse:

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

Knee has two tunables (set on the function call, see the doc): `sensitivity` (default 0.5, range 0–1) controls how sharply the score must drop to mark a cutoff; `early_bias` (default 0.2, range 0–1) prefers cutoffs closer to the top of the list. High sensitivity + high early bias narrows aggressively; low + low keeps it broad.

### `userfn` (UDF)

The User Defined Function language is custom and small. Types: number, string, boolean, datetime, duration. Get values via `get('$.path')` JSONPath against the search result object:

```json
{
  "score": 0.87,
  "text": "...",
  "document_metadata": { "title": "...", "created_at": "..." },
  "part_metadata": { "lang": "eng" },
  "document_id": "doc_4f3a"
}
```

| Pattern | Expression |
|---|---|
| Multiply by document-level boost | `get('$.score') * get('$.document_metadata.boost')` |
| Boost French content by 60% | `if (get('$.part_metadata.lang') == 'fra') get('$.score') * 1.6 else get('$.score')` |
| Drop results scoring under 0.5 | `if (get('$.score') < 0.5) null else get('$.score')` |
| Halve stale results (>1 year old) | `if (hours(now() - iso_datetime_parse(get('$.document_metadata.created_at'))) > 8760) get('$.score') * 0.5 else get('$.score')` |
| Sort by price ascending, missing-last | `get('$.document_metadata.price', -999999)` |

Functions you'll reach for: `now()`, `iso_datetime_parse(str)`, `datetime_parse(str, fmt)`, `to_unix_timestamp(dt)`, `seconds(dur)`, `minutes(dur)`, `hours(dur)`, `abs`, `power`, `min`, `max`, `sqrt`, `log`, `log10`, `ln`. `get('$.path', default)` returns the default when the path is null.

Returning `null` from a UDF *removes* the result before later stages see it. This is the only reranker with that capability.

### Limits and cutoffs

`limit` and `cutoff` live on every reranker stage, not on `search`. They behave per-stage:

- `cutoff` runs first, then `limit`.
- In a chain, each stage's `limit` caps the *input* to the next stage.
- Setting `search.limit` only caps the *final* count after the last reranker.

Typical setup:

```json
{
  "reranker": {
    "type": "chain",
    "rerankers": [
      { "type": "customer_reranker", "reranker_name": "Rerank_Multilingual_v1", "cutoff": 0.75, "limit": 10 },
      { "type": "userfn", "user_function": "get('$.document_metadata.publish_ts')" }
    ]
  }
}
```

Drops anything Slingshot scores under 0.75, takes top 10, then re-sorts by publish timestamp.

### Discovering installed rerankers

`GET /v2/rerankers` returns the catalog of rerankers on your account (id, name, description, enabled). Useful when wiring a `customer_reranker` by name programmatically.

## Context configuration and response shape

`context_configuration` (under `search`) controls how much surrounding text comes back with each chunk. The single biggest lever on response size and readability.

```json
{
  "context_configuration": {
    "sentences_before": 2,
    "sentences_after": 2,
    "start_tag": "<mark>",
    "end_tag": "</mark>"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `sentences_before` / `sentences_after` | int (default `0`) | Preferred. Pad context by sentence count. |
| `characters_before` / `characters_after` | int | Fallback for content without sentence terminators (code, CSV). Ignored if sentence-based is set. |
| `start_tag` / `end_tag` | string | Wraps the matched part inside the expanded text for UI highlighting. `<mark>...</mark>` is a common choice. |

**Sizing:** for long parts (full sections), drop `sentences_before/after` to `0` — duplicating is wasted bytes. For short parts (FAQ entries, captions), bump to 3–5 so each result is self-contained.

### Response shape (POST endpoints)

```json
{
  "summary": "Generated answer text — only present if generation enabled.",
  "search_results": [
    {
      "text": "<mark>The matching part text wrapped by start_tag/end_tag,</mark> with surrounding sentences.",
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

| Field | What |
|---|---|
| `search_results[].text` | The matched part, wrapped with `start_tag`/`end_tag` if configured, padded by `context_configuration`. |
| `search_results[].score` | Final-stage score. Don't re-sort by this — each reranker rescored. |
| `search_results[].document_id` | Opaque, stable. Use as a cache key or for `GET /v2/corpora/{key}/documents/{id}`. |
| `search_results[].document_metadata` / `part_metadata` | Everything you indexed, unfiltered. Anything you need at display time goes here. |
| `search_results[].request_corpora_index` / `corpus_key` | For multi-corpus responses, identifies which corpus this came from. Multi-corpus results interleave by score — don't assume the first N are all from one corpus. |
| `search_results[].table` / `image` | Populated when the part is a table or image; otherwise `null`. Tables are not split and can be many KB on their own. |
| `warnings[]` | Non-fatal: `exceeded_max_input_length_fcs`, `intelligent_query_rewriting_failed`, `fcs_language_not_supported`. |
| `rewritten_queries[]` | Populated only when `intelligent_query_rewriting: true`. |

### Pagination

`search.offset` walks the post-rerank result set. `{ "limit": 10, "offset": 20 }` returns results 21–30. Deep pagination with a heavy reranker chain is expensive — each call reranks the full candidate set again. Cache client-side after the first call when possible.

### GET-endpoint response

Same shape, but with `summary`, `response_language`, `factual_consistency_score`, and `rendered_prompt` only populated when summarization is on (which the GET endpoint can't request). For the GET form, you'll typically see just `search_results` and `warnings`.

## Citations

Citations are part of the generation surface but live in retrieval responses. When generation is enabled, the platform inserts `[N]` markers into the `summary` text, where `N` is a 1-indexed reference into `search_results[]`. Numbers are assigned in order of appearance in the summary, not by relevance score.

```json
{
  "summary": "The Infinite Improbability Drive allows interstellar travel without hyperspace [3]. It is rare with only rumors prior to its development [1].",
  "search_results": [
    { "text": "...", "score": 0.92, "document_id": "doc_123" },
    { "text": "...", "score": 0.89, "document_id": "doc_456" }
  ]
}
```

Configure the format with `generation.citations`:

```json
{
  "generation": {
    "generation_preset_name": "mockingbird-2.0",
    "max_used_search_results": 10,
    "citations": {
      "style": "markdown",
      "url_pattern": "https://docs.example.com/{doc.id}",
      "text_pattern": "{doc.title}"
    }
  }
}
```

| `style` | Output |
|---|---|
| `numeric` (default) | `[1]`, `[2]`, ... — bare numbered references. |
| `none` | Citations stripped from summary text. Use for clean conversational UI. |
| `html` | `<a href="<url_pattern>"><text_pattern></a>`. |
| `markdown` | `[<text_pattern>](<url_pattern>)`. Renders cleanly in Markdown apps and supports code blocks. |

`url_pattern` and `text_pattern` accept `{doc.<field>}` and `{part.<field>}` placeholders that interpolate from the result's metadata. `{doc.id}` and `{doc.title}` are the most common.

The full generation surface (`generation_preset_name`, `prompt_template`, `model_parameters`, streaming, factual consistency scoring) is covered by the `vectara-generation-and-prompts` skill.

## Fuzzy metadata search

Different endpoint, different shape. `POST /v2/corpora/{corpus_key}/metadata_query` does **typo-tolerant matching against metadata fields** (titles, categories, SKUs) — not against document text. Tech Preview.

```json
{
  "level": "document",
  "queries": [
    { "field": "title",    "query": "lease agreement", "weight": 2.0 },
    { "field": "category", "query": "contract",        "weight": 1.0 }
  ],
  "metadata_filter": "doc.status = 'Active'",
  "limit": 10,
  "offset": 0
}
```

| Field | Notes |
|---|---|
| `level` | `document` (unique documents) or `part` (multiple parts from same doc). |
| `queries[].field` | Metadata field name, no `doc.`/`part.` prefix. |
| `queries[].query` | Text to fuzzy-match. |
| `queries[].weight` | Relative importance. Suggested: 2.0–3.0 critical, 1.0–1.5 supporting, 0.5–1.0 context. |
| `queries[].fuzzy` | Default `true`. Tolerates typos and missing characters. |
| `queries[].prefix` | Min query length to enable prefix matching; default `3`, set `null` to disable. |
| `metadata_filter` | Same syntax as regular filters — applied *before* fuzzy matching to narrow the candidate set. |

Response is a flat list of documents with relevance scores, not the retrieval search-results shape:

```json
{
  "documents": [
    { "doc_id": "document123", "score": 0.92, "metadata": { "title": "Master Lease Agreement (2025)", "status": "Active" } }
  ],
  "total_count": 42
}
```

Use this for "find me the SLA doc even if I type Servce Levl Agrement," not for semantic search.

## Tuning guidance

Quick distillation of the long tune-retrieval doc. Work top-to-bottom; earlier fixes pay back the most.

1. **Ingest is the heart of retrieval quality.** No query knob recovers information lost at ingest. Declare `filter_attributes` early. Generate real descriptions for tables and images (those descriptions are what gets embedded). For structured input, use the structured/core indexing APIs instead of file upload.
2. **Start with a strong default body.** 50 candidates, λ `0.025`, 2 sentences of context before/after, chain reranker: `customer_reranker` (cutoff `0.5`, limit `50`) → `mmr` (diversity_bias `0.3`, limit `10`).
3. **Filter before reranking.** Filters are the highest-precision lever — every constraint the user's intent gives you (year, tenant, product, type) should become a `metadata_filter` clause.
4. **Match context to chunk size.** Long chunks: drop context to 0. Short chunks (FAQs, captions): bump to 3–5. Code: prefer character-based bounds.
5. **Tune λ by query type, not intuition.** See the hybrid-search table above. If you reach for λ > 0.5, fix tokenization at ingest instead.
6. **Iterate with an eval set.** 20–50 real queries with the right-answer document(s) labeled. Run with `reranker: { "type": "none" }` first to verify retrieval has the right doc in top-50; if not, no reranker will save you. Then turn the neural reranker on, then MMR, then UDF — checking that each stage helps.
7. **Inspect history.** Set `save_history: true` and `GET /v2/queries/{query_id}` returns `spans[]` showing pre-rerank, post-rerank, and rewritten-query data per stage.

## Common Mistakes

- **Using `corpus_id` (number) instead of `corpus_key` (string).** Especially common when porting from the deprecated v1 API.
- **Confusing the three query endpoints.** Putting a `corpora[]` array in a body posted to `/v2/corpora/{key}/query` fails — that endpoint takes the corpus from the path and doesn't accept `corpora[]`. Conversely, posting `{query, search: {limit, ...}}` without `corpora[]` to `/v2/query` fails because the multi-corpus endpoint requires `corpora[]`.
- **Filtering on a non-declared field.** Returns an error, not zero results. Declare it on the corpus and reindex (`PUT /v2/corpora/{key}/replace_filter_attributes`).
- **Setting both `sentences_before` and `characters_before`.** Sentences win; the character setting is silently ignored. Pick one.
- **Treating `search.limit` as the candidate cap for the neural reranker.** It caps post-rerank count only. To rerank 100 candidates, set `limit: 100` on the reranker stage (or on a `type: "none"` stage upstream).
- **Using `cutoff` against raw hybrid scores or `mmr`/`userfn` outputs and expecting `0.5` to mean "above half."** Only the neural `customer_reranker` produces normalized 0.0–1.0 scores where a static cutoff has a stable meaning.
- **Re-sorting `search_results[]` by `score` client-side.** Each reranker rescores; the response order is the final order.
- **Cranking `lexical_interpolation` above 0.5 to make an exact-term hit show up.** The real fix is at ingest — the term isn't surviving tokenization. Confirm by querying for the term verbatim with λ `1.0` and check whether any result returns at all.
- **Using `reranker_id` / `rnk_272725719`.** Deprecated. Use `type: "customer_reranker"` with `reranker_name: "Rerank_Multilingual_v1"`.
- **Forgetting that `userfn` can return `null` to drop results.** A null return removes the result *before* `limit` is applied and *before* later chain stages see it — handy for filtering but easy to over-prune.
- **Assuming multi-corpus results are grouped by corpus.** They interleave by score. Use `request_corpora_index` or `corpus_key` on each result if origin matters.
- **Posting fuzzy-match needs against the regular `/query` endpoint.** Fuzzy metadata matching is a separate endpoint (`/metadata_query`) with a different request shape.
- **Confusing the GET `search` endpoint with the POST `query` endpoint.** The GET endpoint is path `/v2/corpora/{key}/search` (no body), the POST endpoint is `/v2/corpora/{key}/query` (full body). Same corpus, different capabilities.

## References

Source documentation. Append `.md` to any URL to get the markdown form.

### Concepts
- Search and retrieval overview — `https://docs.vectara.com/docs/search-and-retrieval/search-and-retrieval` (`search-and-retrieval.md`)
- Search quick start — `https://docs.vectara.com/docs/search-and-retrieval/search-quick-start` (`search-quick-start.md`)
- Hybrid search — `https://docs.vectara.com/docs/search-and-retrieval/hybrid-search` (`hybrid-search.md`)
- Filters — `https://docs.vectara.com/docs/search-and-retrieval/filters` (`filters.md`)
- Citations — `https://docs.vectara.com/docs/search-and-retrieval/citations` (`citations.md`)
- Fuzzy metadata search — `https://docs.vectara.com/docs/search-and-retrieval/fuzzy-metadata-search` (`fuzzy-metadata-search.md`)
- Tune retrieval — `https://docs.vectara.com/docs/search-and-retrieval/tune-retrieval` (`tune-retrieval.md`)

### Rerankers
- Reranking overview — `https://docs.vectara.com/docs/search-and-retrieval/reranking` (`reranking.md`)
- Chain reranker — `https://docs.vectara.com/docs/search-and-retrieval/rerankers/chain-reranker` (`chain-reranker.md`)
- Knee reranker — `https://docs.vectara.com/docs/search-and-retrieval/rerankers/knee-reranking` (`knee-reranking.md`)
- MMR reranker — `https://docs.vectara.com/docs/search-and-retrieval/rerankers/mmr-reranker` (`mmr-reranker.md`)
- Vectara multilingual reranker (Slingshot) — `https://docs.vectara.com/docs/search-and-retrieval/rerankers/vectara-multi-lingual-reranker` (`vectara-multi-lingual-reranker.md`)
- User Defined Function reranker — `https://docs.vectara.com/docs/search-and-retrieval/rerankers/user-defined-function-reranker` (`user-defined-function-reranker.md`)
- Limits and cutoffs — `https://docs.vectara.com/docs/search-and-retrieval/rerankers/limits-and-cutoffs` (`limits-and-cutoffs.md`)

### REST API
- Multi-corpus query (POST) — `https://docs.vectara.com/docs/rest-api/query` (`query.md`)
- Single-corpus advanced query (POST) — `https://docs.vectara.com/docs/rest-api/query-corpus` (`query-corpus.md`)
- Single-corpus simple search (GET) — `https://docs.vectara.com/docs/rest-api/search-corpus` (`search-corpus.md`)
- Metadata fuzzy query (POST) — `https://docs.vectara.com/docs/rest-api/query-metadata` (`query-metadata.md`)
- List rerankers — `https://docs.vectara.com/docs/rest-api/list-rerankers` (`list-rerankers.md`)
- OpenAPI spec — `https://api.vectara.io/v2/openapi.json`
- Full doc index — `https://docs.vectara.com/llms.txt`

### Related skills
- `vectara-agents` — corpus-search agent tool wiring, sessions, the 3-step chat flow.
- `vectara-agent-orchestration` — multi-step state machines that use `corpora_search` as one tool among many.
- `vectara-generation-and-prompts` — the `generation` block: presets, custom `prompt_template`, `model_parameters`, streaming, factual consistency score.
