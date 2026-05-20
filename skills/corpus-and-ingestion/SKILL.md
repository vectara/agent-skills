---
name: vectara-corpus-and-ingestion
description: Create Vectara corpora, declare filter attributes, ingest documents (structured / core / file upload), write SQL-ish metadata filter expressions, update or replace document metadata, run bulk-delete jobs, extract tables, and choose between the API and Vectara Ingest crawlers.
tags: [vectara, corpus, ingestion, indexing, metadata, filters, rag]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Corpus and Data Ingestion

You are helping a developer get data into a Vectara corpus and make it filterable at query time. This skill covers everything between "I have raw files / structured records" and "I can query them with a `metadata_filter`": corpus creation, the three ingestion paths, filter-attribute declaration, the filter expression language, document/metadata updates, async bulk delete, table extraction, and the Vectara Ingest crawler framework.

## When to use this skill

- Creating a new corpus or modifying its filter attributes.
- Deciding between `upload_file`, `structured` document indexing, and `core` document indexing.
- Writing a `metadata_filter` expression and the corresponding `filter_attributes` declaration.
- Mutating documents: merging metadata, replacing metadata, deleting documents by ID or filter.
- Ingesting PDFs that contain tables, or building a JSON table directly.
- Deciding whether to call the API directly or use the Vectara Ingest open-source crawlers (SharePoint, Confluence, Notion, Jira, Slack, ServiceNow, HubSpot, websites, RSS, CSV, GitHub, etc.).
- Referencing a corpus or document in a URL — `corpus_key` vs `corpus_id`, document_id escaping.

For querying (`POST /v2/queries`, rerankers, `lexical_interpolation`, generation), use the `vectara-agents` skill or the OpenAPI spec directly. For agent integration of search, see `vectara-agent-orchestration`.

## Critical Rules

1. **API base URL** is `https://api.vectara.io/v2/`. Auth is `x-api-key: <key>` or `Authorization: Bearer <token>` — never both.
2. **Address corpora by `corpus_key` (string) in URLs, not `corpus_id` (number).** The path pattern is `/v2/corpora/{corpus_key}/...`. `corpus_id` (`crp_1234`) only appears in API responses. `corpus_key` must match `[a-zA-Z0-9_=\-]+$`, max length 50, and is unique per account.
3. **A metadata field cannot appear in `metadata_filter` until it has been declared as a `filter_attribute` on the corpus, with the matching `name`, `level`, and `type`.** Indexing a document with arbitrary metadata is allowed, but only declared attributes are filterable. Use `indexed: true` to also build a query index (recommended for any attribute used in selective filters).
4. **The `encoder_name` is set at corpus creation and is immutable.** Pick deliberately. To change encoder you must create a new corpus and re-index.
5. **Pick the right document type up front.** `structured` lets Vectara chunk for you (sections → parts); `core` requires you to specify each `document_parts[]` entry yourself (1 part = 1 vector). You cannot mix the two shapes in one request, and the response uses a `type` discriminator.
6. **Metadata field names must match `filter_attributes.name` exactly** (case-sensitive). Types must align with the declared `type` (`integer`, `real_number`, `text`, `boolean`, `list[...]`). Use integers for years and YYYYMMDD-style dates so range comparisons work; storing `'2021'` as `text` blocks numeric comparison.
7. **`doc.X` and `part.X` are different scopes.** Document-level metadata goes on the top-level `metadata` object of a structured/core document; part-level metadata goes on `sections[].metadata` (structured) or `document_parts[].metadata` (core). Filter expressions must use the prefix that matches the declared `level`.
8. **`PATCH /documents/{id}` merges metadata; `PUT /documents/{id}/metadata` replaces it.** Both are document-level only — there is no API to update part-level metadata or document body text in place. To change part text, delete the document and re-index.
9. **Bulk delete is asynchronous by default and best-effort.** `document_ids` deletes from primary storage (reliable). `metadata_filter` deletes against the search index (subject to indexing lag — may miss recently added docs).
10. **Replacing `filter_attributes` on an existing corpus is async** (`POST /replace_filter_attributes` returns a `job_id`). The new attributes are not usable until the job completes.

## Resource addressing

| Field | Type | Where it appears | Notes |
|---|---|---|---|
| `corpus_key` | string, `[a-zA-Z0-9_=\-]+$`, ≤50 chars | path: `/v2/corpora/{corpus_key}` | User-chosen, unique per account. Use everywhere you address a corpus. |
| `corpus_id` | string `crp_<n>` | response bodies only | System-generated. Never put in a path. |
| `document_id` | string | path: `/v2/corpora/{key}/documents/{document_id}` | User-chosen, unique within the corpus. **Must be percent-encoded in URLs.** |
| `agent_key` / `session_key` | strings | agents API | Same pattern as `corpus_key`. |

## Choosing an ingestion path

| Path | Endpoint | Input | Vectara handles | Use when |
|---|---|---|---|---|
| File upload | `POST /v2/corpora/{key}/upload_file` (multipart) | Binary file ≤10 MB | parsing, chunking, optional table extraction | You have PDF/DOCX/PPTX/HTML/MD/RTF/EPUB/TXT/ODT/email and no extraction code |
| Structured indexing | `POST /v2/corpora/{key}/documents` with `type: "structured"` | JSON with `sections[]` | chunking (1 chunk per sentence by default) | You already have hierarchical content — articles, FAQs, DB rows, ERP records |
| Core indexing | `POST /v2/corpora/{key}/documents` with `type: "core"` | JSON with `document_parts[]` | nothing — you control chunks 1:1 with vectors | ML team wants explicit chunk control, custom dimensions, fine-tuned retrieval |
| Vectara Ingest | open-source CLI/library | data-source-specific | crawling + parsing + indexing | Sources not covered by File Upload (SharePoint, Confluence, Slack, Jira, Notion, ServiceNow, HubSpot, RSS, CSV, GitHub, Twitter/X, YouTube, MediaWiki, websites) |

**File Upload supported types:** `.md`, `.pdf`, `.odt`, `.doc`, `.docx`, `.ppt`, `.pptx`, `.txt`, `.html`, `.lxml`, `.rtf`, `.epub`, RFC 822 email. JSON files with `.json` extension are also accepted — the platform detects whether the JSON is a structured or core document.

## Create a corpus (with filter attributes)

`POST /v2/corpora`. `key` is the only required field. Declare every metadata attribute you intend to filter on, including its `level` and `type`.

```json
{
  "key": "fin_esg_docs",
  "name": "EU Bank ESG Compliance",
  "description": "ESG filings for European banks, 2023-onward.",
  "save_history": true,
  "encoder_name": "boomerang-2023-q3",
  "filter_attributes": [
    { "name": "industry",     "level": "document", "type": "text",    "indexed": true, "description": "Sector (banking, energy)." },
    { "name": "region",       "level": "document", "type": "text",    "indexed": true },
    { "name": "year",         "level": "document", "type": "integer", "indexed": true },
    { "name": "doc_type",     "level": "document", "type": "text",    "indexed": true },
    { "name": "is_published", "level": "document", "type": "boolean", "indexed": true },
    { "name": "tags",         "level": "document", "type": "list[text]", "indexed": true },
    { "name": "section_type", "level": "part",     "type": "text",    "indexed": true }
  ],
  "custom_dimensions": [
    { "name": "importance", "indexing_default": 0, "querying_default": 0 }
  ]
}
```

Returns `201` with `id` (the `crp_*`), the assigned `key`, and the full filter-attribute list. Notable optional flags:

- `queries_are_answers` / `documents_are_questions` — swap encoder semantics for inverted-search use cases.
- `save_history` — default `false`; if `true`, queries against this corpus are saved to query history.
- `custom_dimensions` — Pro/Enterprise only; numeric weights you multiply against query-time custom dimensions to bias ranking.

**Encoder.** `encoder_name` (e.g. `boomerang-2023-q3`) is set once and cannot be changed. `encoder_id` (`enc_*`) is deprecated; use `encoder_name`.

## Filter expression syntax

A `metadata_filter` is a SQL-`WHERE`-style expression evaluated per document/part at query time. Every metadata reference must be prefixed with `doc.` or `part.` to match the declared `level`.

### Operators (precedence high → low)

| Operator | Associativity | Notes |
|---|---|---|
| `+`, `-` (unary) | — | numeric sign |
| `*`, `/`, `%` | left | arithmetic on numeric fields |
| `+`, `-` (binary) | left | addition / subtraction |
| `<`, `<=`, `>`, `>=` | left | comparison |
| `=`, `==`, `!=`, `<>` | left | equality (`=` and `==` synonyms; `!=` and `<>` synonyms) |
| `IS NULL`, `IS NOT NULL` | — | presence check |
| `IN (...)` | — | range containment for scalars |
| `NOT` | — | logical negation; wrap operand in parens |
| `AND` | left | conjunction |
| `OR` | left | disjunction |

All operators at the same level are left-associative. Use parentheses to override precedence.

### Data types (literal syntax)

| Declared `type` | Literal syntax | Example |
|---|---|---|
| `integer` | digits, no period | `doc.year = 2024` |
| `real_number` (float, IEEE-754 double) | digits with period | `part.score > 0.7` |
| `text` (UTF-8) | single-quoted; escape `'` by doubling (`''`) | `doc.category = 'Technology'` |
| `boolean` | `true` / `false` | `doc.is_featured = true` |
| `null` | `null` | `doc.author IS NULL` |
| `list[integer]` | flip operand order: `<num> IN doc.<field>` | `2023 IN doc.years` |
| `list[real_number]` | `<num> IN doc.<field>` | `3.14 IN doc.scores` |
| `list[text]` | `'<str>' IN doc.<field>` | `'India' IN doc.country` |

Note: for list types, `IN` reads "list contains value" — the literal goes on the **left**, the field on the right. This is the inverse of scalar `IN`. Multiple-value containment is `AND`-of-`IN`s: `'India' IN doc.country AND 'Denmark' IN doc.country`.

### Built-in fields (always available, no declaration required)

| Field | Type | What |
|---|---|---|
| `doc.id` | text | The document_id you supplied at indexing time. |
| `part.lang` | text | ISO 639-2 3-letter code (e.g. `eng`, `deu`), auto-detected per part. |
| `part.is_title` | boolean (3-valued) | `true` for detected titles, `false` for non-titles, `unset` otherwise. Use `<> true` to include unset. |

### Worked filter examples

```sql
-- Published documents in finance or business, after Jan 2022:
doc.category IN ('Business', 'Finance') AND doc.publishdate > '2022-01-01' AND doc.status = 'published'

-- German-language reviews above 3 stars (cross-scope):
doc.rating > 3 AND part.lang = 'deu'

-- Exclude drafts in the Technology category:
NOT (doc.status = 'draft' AND doc.category = 'Technology')

-- Year as integer (range works); year-as-text would not:
doc.year >= 2021 AND doc.year <= 2024

-- List membership:
'oncology' IN doc.tags AND 2024 IN doc.review_years
```

## Structured document ingestion

`POST /v2/corpora/{corpus_key}/documents?wait_for=searchable`. `wait_for` controls when the response returns: `searchable` (default — wait until indexed and queryable) or `indexed` (faster, document is durable but may not appear in search for a brief moment).

```json
{
  "type": "structured",
  "id": "esg_report_2024",
  "title": "2024 ESG Annual Report - EuroBank",
  "description": "Comprehensive ESG report for 2024.",
  "metadata": {
    "industry": "banking",
    "region": "EU",
    "year": 2024,
    "doc_type": "esg_report",
    "is_published": true,
    "tags": ["sustainability", "compliance"]
  },
  "sections": [
    {
      "id": 1,
      "title": "Environmental Initiatives",
      "text": "EuroBank reduced carbon emissions by 22% in 2024 ...",
      "metadata": { "section_type": "environmental" }
    },
    {
      "id": 2,
      "title": "Social Responsibility",
      "text": "Community outreach expanded to 500+ initiatives ...",
      "metadata": { "section_type": "social" },
      "sections": [
        { "title": "Volunteer Hours", "text": "Employee volunteer hours grew 30% YoY." }
      ]
    }
  ],
  "chunking_strategy": { "type": "max_chars_chunking_strategy", "max_chars_per_chunk": 768 }
}
```

- Top-level `metadata` populates `doc.*` filter attributes.
- `sections[].metadata` populates `part.*` filter attributes for every chunk derived from that section.
- Sections may nest arbitrarily; each nested section can carry its own `metadata`, `tables`, `images`.
- `chunking_strategy` defaults to `sentence_chunking_strategy` (one chunk per sentence). `max_chars_chunking_strategy` (with `max_chars_per_chunk >= 100`) balances retrieval latency and context. 3-7 sentences per chunk (≈512-1024 chars) is the recommended starting point.

### Recognized "special" document metadata

| Field | Effect |
|---|---|
| `date` | Shown in Console search UI as document date. |
| `url` | Shown as clickable link in Console UI. |
| `ts_create` | Epoch seconds; shown as creation timestamp. |
| `author` | String or array of strings; shown in Console UI. |

These are conventions for Console visibility; they still need a matching `filter_attribute` if you want to filter on them at query time.

## Core document ingestion

Use when you need 1:1 control of vectors. Each `document_parts[]` entry is exactly one chunk → one embedding → one search result candidate.

```json
{
  "type": "core",
  "id": "invoice-001",
  "metadata": { "doc_type": "invoice", "industry": "manufacturing" },
  "document_parts": [
    {
      "text": "Q1 billing for Acme Corp: $10,230.25.",
      "context": "Line items table summary.",
      "metadata": { "quarter": 1, "year": 2023 },
      "custom_dimensions": { "importance": 1.5 }
    },
    {
      "text": "Q1 billing for Beta Industries: $8,750.00.",
      "metadata": { "quarter": 1, "year": 2023 }
    }
  ]
}
```

- `document_parts` is required and must contain at least one part.
- `text` is required per part. `context` is optional supplemental text used for ranking but not as a primary search match.
- `metadata` on a part populates `part.*` filter attributes (and must match the declared name/type).
- `custom_dimensions` values are multiplied against query-time custom dimensions to bias ranking; the dimension must be declared on the corpus.
- `table_id` / `image_id` on a part link it to a `tables[]` / `images[]` entry on the same document.

## File upload (auto-extraction)

`POST /v2/corpora/{corpus_key}/upload_file` as `multipart/form-data`. Max 10 MB per file. The filename becomes the `document_id` by default; override with the `filename` form field (or via `Content-Disposition`). Non-ASCII filenames must be URL-encoded.

```bash
curl -L -X POST "https://api.vectara.io/v2/corpora/employee-handbook/upload_file" \
  -H "Accept: application/json" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -F 'file=@"./pto_policy.pdf"' \
  -F 'filename=pto_policy_v3' \
  -F 'metadata={"doc_type":"policy","year":2025};type=application/json' \
  -F 'chunking_strategy={"type":"max_chars_chunking_strategy","max_chars_per_chunk":768};type=application/json' \
  -F 'table_extraction_config={"extract_tables":true};type=application/json'
```

Multipart form fields:

| Field | Required | Notes |
|---|---|---|
| `file` | yes | Binary file. |
| `filename` | optional | Override the document_id. |
| `metadata` | optional | JSON object; populates `doc.*` filter attributes. Must include `;type=application/json`. |
| `chunking_strategy` | optional | `sentence_chunking_strategy` (default) or `max_chars_chunking_strategy` with `max_chars_per_chunk >= 100`. |
| `table_extraction_config` | optional | `{"extract_tables": true, "extractor": {"name": "..."}, "generation": {...}}`. Tables become separate document parts. |

Responses: `201` on success, `409` if a document with the same ID already exists with different content (delete it first or use a unique id), `415` for unsupported media types.

## Document and metadata updates

| Operation | Endpoint | Body | Semantics |
|---|---|---|---|
| Merge document metadata | `PATCH /v2/corpora/{key}/documents/{document_id}` | `{ "metadata": {...} }` | Adds new keys and overwrites listed keys; leaves other keys intact. |
| Replace document metadata | `PUT /v2/corpora/{key}/documents/{document_id}/metadata` | `{ "metadata": {...} }` | Wholesale replace — keys not present in body are removed. |
| Delete a document | `DELETE /v2/corpora/{key}/documents/{document_id}` | — | Synchronous, single doc. |
| Bulk delete | `DELETE /v2/corpora/{key}/documents` | query params | Async by default. See below. |

`document_id` in any path must be percent-encoded. Both update operations are **document-level metadata only** — there is no API to update part-level metadata or change a document's body text. To modify body text or part metadata, delete the document and re-index it.

## Bulk delete (async job)

`DELETE /v2/corpora/{corpus_key}/documents`. Specify `document_ids`, `metadata_filter`, or both. By default the operation is async and returns immediately.

```bash
# Delete a specific set of documents (most reliable):
curl -X DELETE "https://api.vectara.io/v2/corpora/$CORPUS_KEY/documents?document_ids=doc-1,doc-2,doc-3" \
  -H "x-api-key: $VECTARA_API_KEY"

# Delete by metadata filter (best-effort, subject to indexing lag):
curl -X DELETE "https://api.vectara.io/v2/corpora/$CORPUS_KEY/documents" \
  --data-urlencode "metadata_filter=doc.status = 'archived' AND doc.year < 2020" \
  -H "x-api-key: $VECTARA_API_KEY"

# Synchronous mode (waits up to 7 days or Request-Timeout):
curl -X DELETE "https://api.vectara.io/v2/corpora/$CORPUS_KEY/documents?async=false&document_ids=doc-1" \
  -H "x-api-key: $VECTARA_API_KEY"
```

| Mode | Returns | Notes |
|---|---|---|
| `async=true` (default) | `202` with `{job_id, cutoff_timestamp}` | Track via Jobs API. `cutoff_timestamp` (ISO 8601) — only docs created/updated before this moment are deleted. |
| `async=false` | `200` with `{job_id, deleted_count, skipped_count, cutoff_timestamp}` | Times out → `504` with `job_id` in the message; workflow keeps running. |

- `document_ids` accepts up to 10,000 IDs per request and deletes from primary storage (no indexing-lag risk).
- `metadata_filter` deletes against the search index — recently indexed documents may be missed. For mission-critical wipes, prefer `document_ids` or repeat the filter-based call after the indexing pipeline catches up.

### Per-document iteration with retry (full-clear pattern)

When you need to clear *every* document and want resilience to transient gateway errors, iterate the list endpoint and DELETE each id with a small retry loop on `502`/`503`/`504` and `RequestException`. A single network blip on one DELETE shouldn't kill the whole sweep:

```python
import time, requests

_TRANSIENT = (502, 503, 504)

def _request_with_retry(method, url, *, max_retries=2, **kwargs):
    for attempt in range(max_retries + 1):
        try:
            r = requests.request(method, url, **kwargs)
        except requests.exceptions.RequestException:
            if attempt < max_retries:
                time.sleep(2 ** attempt); continue
            return None
        if r.status_code in _TRANSIENT and attempt < max_retries:
            time.sleep(2 ** attempt); continue
        return r

def clear_corpus_documents(base_url, headers, corpus_key, timeout=30):
    page_key = None
    deleted = 0
    while True:
        params = {"limit": 100, **({"page_key": page_key} if page_key else {})}
        r = _request_with_retry("GET", f"{base_url}/corpora/{corpus_key}/documents",
                                headers=headers, params=params, timeout=timeout)
        if r is None or r.status_code != 200: return deleted
        data = r.json()
        for doc in data.get("documents", []):
            dr = _request_with_retry("DELETE",
                f"{base_url}/corpora/{corpus_key}/documents/{doc['id']}",
                headers=headers, timeout=timeout)
            if dr is not None and dr.status_code in (200, 204):
                deleted += 1
        page_key = data.get("metadata", {}).get("page_key")
        if not page_key: return deleted
```

Same retry policy applies to idempotent (re)create for agents and tools — see `vectara-agents`. Reference implementation: [`vectara_utils.py`](https://github.com/vectara/example-notebooks/blob/main/notebooks/api-examples/vectara_utils.py).

## Replacing filter attributes on an existing corpus

`POST /v2/corpora/{corpus_key}/replace_filter_attributes` with `{ "filter_attributes": [...] }`. The endpoint returns `200 { "job_id": "job_..." }` immediately. The new attribute set is not usable in filter expressions until the async re-index job completes — poll the Jobs API for status.

The body **replaces** the entire attribute list, so include every attribute you want to keep, not just the new ones.

## Tables

Tables can enter the corpus three ways:

1. **PDF upload with `table_extraction_config.extract_tables: true`.** Vectara runs a table extractor (default, `gmft`, or `textract` per `GET /v2/table_extractors`) and indexes each extracted table as additional document parts. Optionally provide `generation.llm_name` + `generation.prompt_template` (Apache Velocity) to customize the per-table summary; built-in template variables include `$vectaraTable.markdown()`, `$vectaraTable.json()`, `$vectaraTable.title()`, `$vectaraTable.description()`.
2. **Structured document** with `sections[].tables[]` — each table has `id`, `title`, `data.headers[][]`, `data.rows[][]`, optional `description`. Cells use `text_value` / `int_value` / `float_value` / `bool_value` and optional `colspan` / `rowspan`.
3. **Core document** with top-level `tables[]` + `document_parts[].table_id` to link each summarizing part to its table.

At query time, search results that match table content carry a `vectara` metadata block:

```json
{
  "vectara": {
    "table_id": "billing_table_1",
    "is_table_summary": true,
    "is_table_title": false,
    "row_num": 3
  }
}
```

Use `table_id` to fetch the full table via the documents API. Limitations: English-only; cannot do cross-cell math at query time; merged-row cells may be reported as empty; scanned images of tables are not OCR'd by the table extractor (use Vectara Ingest with an OCR-capable parser for that).

## Vectara Ingest (open-source crawlers)

[github.com/vectara/vectara-ingest](https://github.com/vectara/vectara-ingest) is a Python framework with pre-built crawlers for sources the File Upload API doesn't cover: SharePoint, Confluence, Notion, Jira, Slack, ServiceNow, HubSpot, MediaWiki, GitHub, Twitter/X, YouTube, RSS feeds, CSV/databases, generic websites, and more. It also extends file-type support with advanced PDF processing, OCR for scanned docs, image summarization via vision models, and alternate parsers (`unstructured`, `llama_parse`, `docupanda`, `docling`).

Choose Vectara Ingest when:
- The source isn't a supported upload type (e.g. a Confluence space or RSS feed).
- You need OCR or vision-model image summarization.
- You want PII masking, custom metadata extraction, or per-source chunking strategies.
- Batch-crawling many sources on a schedule is the right shape.

Stick with the official APIs when:
- The source is already a supported file format.
- You need a vendor-supported integration (Vectara Ingest is provided as-is, no official support).
- The volume is low enough that an in-application call beats running a crawler container.

## End-to-end flow

```bash
API=https://api.vectara.io/v2
H=(-H "x-api-key: $VECTARA_API_KEY" -H "Content-Type: application/json")

# 1. Create the corpus with filter attributes declared up front.
curl -X POST "$API/corpora" "${H[@]}" -d '{
  "key": "support_kb",
  "name": "Support KB",
  "encoder_name": "boomerang-2023-q3",
  "filter_attributes": [
    { "name": "product",  "level": "document", "type": "text",    "indexed": true },
    { "name": "year",     "level": "document", "type": "integer", "indexed": true },
    { "name": "category", "level": "part",     "type": "text",    "indexed": true }
  ]
}'

# 2a. Upload an unstructured PDF (Vectara parses + chunks).
curl -X POST "$API/corpora/support_kb/upload_file" \
  -H "x-api-key: $VECTARA_API_KEY" \
  -F 'file=@"./troubleshooting.pdf"' \
  -F 'metadata={"product":"portal_app","year":2025};type=application/json'

# 2b. Or index a structured document with part-level metadata.
curl -X POST "$API/corpora/support_kb/documents?wait_for=searchable" "${H[@]}" -d '{
  "type": "structured",
  "id": "kb-sso-001",
  "title": "SSO token refresh failures",
  "metadata": { "product": "portal_app", "year": 2025 },
  "sections": [
    { "title": "Symptoms",  "text": "Users see 401 after 60 minutes.", "metadata": {"category":"sso"} },
    { "title": "Diagnosis", "text": "Clock skew on the IdP exceeds 5 minutes.", "metadata": {"category":"sso"} }
  ]
}'

# 3. Merge in new metadata later.
curl -X PATCH "$API/corpora/support_kb/documents/kb-sso-001" "${H[@]}" \
  -d '{"metadata": {"reviewed_by": "rima"}}'

# 4. Bulk-delete stale docs by filter (returns job_id).
curl -X DELETE "$API/corpora/support_kb/documents" \
  -H "x-api-key: $VECTARA_API_KEY" \
  --data-urlencode "metadata_filter=doc.year < 2023"
```

## Common Mistakes to Avoid

- Using `corpus_id` (`crp_1234`) in a URL path. Paths take `corpus_key` (the string you assigned).
- Writing a `metadata_filter` against a field that was never declared in `filter_attributes`. The filter parses but matches nothing.
- Declaring a year as `text` (`"year": "2024"`) and then trying `doc.year > 2020`. String comparison, not numeric — use `integer`.
- Putting metadata at the wrong level. `doc.section_type` won't match a value attached to `sections[].metadata` (that's `part.section_type`).
- Mismatching the declared `level`. If `filter_attributes` says `"level": "part"`, the filter must say `part.X`, not `doc.X`.
- Forgetting that `list[...]` filtering uses inverted syntax — `'India' IN doc.country`, not `doc.country IN ('India')`.
- Expecting `PATCH /documents/{id}` to update part-level metadata or document text. It only merges top-level document metadata.
- Re-creating a document with the same `id` to "update" it — returns `409` if content differs. Delete first, then re-index.
- Treating bulk delete as synchronous and assuming all matches are gone immediately. Default is async, returns a `job_id`; `metadata_filter` mode is best-effort against the indexing pipeline.
- Treating `replace_filter_attributes` as instant. It returns a `job_id`; the new attributes are unusable until the job finishes.
- Trying to change `encoder_name` after corpus creation. It's immutable — make a new corpus.
- Uploading a file larger than 10 MB to `/upload_file`. Use Vectara Ingest, or split the file before upload.
- Sending the `metadata` form field without `;type=application/json`. The form part may be parsed as a string and silently dropped.

## References

Append `.md` to any Vectara docs URL to get the markdown source.

### Guides

- [Build overview](https://docs.vectara.com/docs/build) — `build.md`
- [Data ingestion](https://docs.vectara.com/docs/build/data-ingestion) — `data-ingestion.md`
- [Metadata filters](https://docs.vectara.com/docs/build/prepare-data/metadata-filters) — `metadata-filters.md`
- [Data types](https://docs.vectara.com/docs/build/prepare-data/metadata-filters/data-types) — `data-types.md`
- [Functions and operators](https://docs.vectara.com/docs/build/prepare-data/metadata-filters/func-opr) — `func-opr.md`
- [Metadata examples and use cases](https://docs.vectara.com/docs/build/prepare-data/metadata-filters/metadata-examples-and-use-cases) — `metadata-examples-and-use-cases.md`
- [Working with tables](https://docs.vectara.com/docs/build/working-with-tables) — `working-with-tables.md`
- [Vectara Ingest overview](https://docs.vectara.com/docs/build/vectara-ingest) — `vectara-ingest.md`
- [Resource addressing](https://docs.vectara.com/docs/build/resource-addressing) — `resource-addressing.md`

### REST API

- [Create corpus](https://docs.vectara.com/docs/rest-api/create-corpus) — `create-corpus.md`
- [List corpora](https://docs.vectara.com/docs/rest-api/list-corpora) — `list-corpora.md`
- [Upload file](https://docs.vectara.com/docs/rest-api/upload-file) — `upload-file.md`
- [Add document to corpus](https://docs.vectara.com/docs/rest-api/create-corpus-document) — `create-corpus-document.md`
- [Update document (merge)](https://docs.vectara.com/docs/rest-api/update-corpus-document) — `update-corpus-document.md`
- [Replace document metadata](https://docs.vectara.com/docs/rest-api/replace-corpus-document-metadata) — `replace-corpus-document-metadata.md`
- [Replace filter attributes](https://docs.vectara.com/docs/rest-api/replace-filter-attributes) — `replace-filter-attributes.md`
- [Bulk delete documents](https://docs.vectara.com/docs/rest-api/bulk-delete-corpus-documents) — `bulk-delete-corpus-documents.md`
- [Compute corpus size](https://docs.vectara.com/docs/rest-api/compute-corpus-size) — `compute-corpus-size.md`
- [List table extractors](https://docs.vectara.com/docs/rest-api/list-table-extractors) — `list-table-extractors.md`
- [OpenAPI spec](https://api.vectara.io/v2/openapi.json)
- [Full docs index](https://docs.vectara.com/llms.txt)

### Tooling

- [Vectara Ingest (GitHub)](https://github.com/vectara/vectara-ingest)
