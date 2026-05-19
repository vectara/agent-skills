---
name: vectara-hallucination-and-eval
description: Score factual consistency with HHEM, rewrite ungrounded sentences with Vectara Hallucination Corrector, inspect query/agent traces in-console, and run offline RAG evaluation with the open-source Open RAG Eval Python toolkit. Covers when each surface is the right tool and how they compose end-to-end.
tags: [vectara, hallucination, hhem, factual-consistency, eval, rag, observability, open-rag-eval]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Hallucination Detection, Correction, and Evaluation

You are helping a developer measure and reduce hallucinations in a Vectara RAG or agent pipeline. Vectara exposes four distinct surfaces for this; they are not interchangeable, and conflating them is the most common mistake.

## When to use this skill

- The developer wants a numeric "is this generation grounded in the retrieved passages" signal — that is **HHEM** (factual consistency score), inline on `POST /v2/query` summarization or standalone via `POST /v2/evaluate_factual_consistency`.
- A flagged generation needs to be **rewritten** to remove unsupported claims — that is the **Vectara Hallucination Corrector (VHC)** at `POST /v2/hallucination_correctors/correct_hallucinations`.
- The developer is debugging why a specific past query returned what it did, or wants per-stage latency — that is **query/agent observability** in the console (and `GET /v2/agent_analytics/traces` for sessions).
- The developer wants to benchmark different model/reranker/lambda configurations offline against a labeled dataset before promoting a change — that is the **Vectara Open RAG Eval** open-source Python package.

If the developer only wants to build an agent or run a query, use `vectara-agents` instead. If they want to wire eval into CI as a gate on a PR, this skill is correct.

## Critical Rules

1. **HHEM scores groundedness, not truth.** A score of `0.95` means "the generated text is supported by the supplied source passages with high confidence." It is **not** a probability that the generation is correct about the real world. If the retrieved sources are wrong, HHEM will happily score a grounded-but-wrong answer high.
2. **HHEM and the Hallucination Corrector are separate steps.** HHEM returns a score; VHC returns rewritten text plus per-span explanations. VHC does **not** invent facts to fill gaps — if a claim isn't supported by the source documents, VHC removes or hedges it; in the extreme case it returns an empty `corrected_text` and recommends discarding the generation entirely.
3. **Open RAG Eval is an external open-source Python package (`open-rag-eval` on PyPI / GitHub), not a Vectara API.** There is no `POST /v2/eval` endpoint. It runs offline in your CI / notebook, and only talks to Vectara via the normal query API.
4. **Query observability is a platform/console feature, not a public REST endpoint for arbitrary trace export.** Per-query traces are visible in the Vectara console (Analysis tab on a corpus, Sessions tab on an agent). Agent sessions can be pulled in bulk via `GET /v2/agent_analytics/traces`, but corpus query history is console-first.
5. **Inline HHEM and standalone HHEM are the same model, different call shapes.** Inline (`enable_factual_consistency_score: true` on a summarization query) scores the generated summary against the search results that produced it. Standalone (`POST /v2/evaluate_factual_consistency`) scores any `generated_text` against any `source_texts[]` you supply — useful when the generation didn't come from Vectara, or when you want to re-score after VHC.
6. **HHEM supports only listed languages.** `eng`, `deu`, `fra`, `spa`, `por`, `ara`, `kor`, `zho`, `rus`, `jpn`, `hin`. A 422 from the eval endpoint with "language not supported" means exactly that — set `language` correctly using the ISO 639-3 code.

## HHEM — Factual Consistency Score

HHEM (Hughes Hallucination Evaluation Model, currently `hhem_v2.3`) returns a calibrated score in `[0.0, 1.0]`. Higher = more supported by the source. There are two ways to call it.

### Inline on a summarization query

Set `enable_factual_consistency_score: true` inside `generation`. The score is returned as `factual_consistency_score` alongside the summary.

```json
{
  "query": "What is the answer to life, the universe, and everything?",
  "search": {
    "corpora": [{ "corpus_key": "hitchhikers" }],
    "limit": 10
  },
  "generation": {
    "generation_preset_name": "vectara-summary-ext-24-05-med-omni",
    "max_used_search_results": 5,
    "enable_factual_consistency_score": true,
    "response_language": "eng"
  }
}
```

Response excerpt:

```json
{
  "summary": "According to 'The Hitchhiker's Guide to the Galaxy' by Douglas Adams, the answer is 42.",
  "summary_language": "en",
  "factual_consistency_score": 0.98,
  "search_results": [ /* ... */ ]
}
```

### Standalone — `POST /v2/evaluate_factual_consistency`

Score an arbitrary `generated_text` against arbitrary `source_texts`. Use this when the generation came from a non-Vectara model, when re-scoring after VHC, or when running offline eval.

```bash
curl -s -X POST https://api.vectara.io/v2/evaluate_factual_consistency \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model_parameters": { "model_name": "hhem_v2.3" },
    "generated_text": "The Eiffel Tower is located in Berlin.",
    "source_texts": [
      "The Eiffel Tower is a famous landmark located in Paris, France.",
      "It was built in 1889 and remains one of the most visited monuments in the world."
    ],
    "language": "eng"
  }'
```

Response:

```json
{
  "score": 0.23,
  "p_consistent": 0.12,
  "p_inconsistent": 0.88
}
```

### Interpreting the score

There is no single universal threshold — calibrate against your own data. Sensible defaults:

| Score range | Typical decision |
|---|---|
| `>= 0.7`   | Accept and serve to the user. |
| `0.5 – 0.7` | Borderline. Run the Hallucination Corrector and re-score the corrected text. Serve corrected output if the post-correction score crosses your accept threshold. |
| `< 0.5`    | Reject or correct. If post-correction still under threshold, fall back to "I don't have enough information" rather than serving the original. |

The docs recommend starting with `0.5` as the gate when you're new to the score. Tighten upward in regulated domains (legal, healthcare, finance).

## Vectara Hallucination Corrector (VHC)

VHC takes a `generated_text` plus its `documents[]` and returns a `corrected_text` with minimal edits to remove unsupported claims, plus a `corrections[]` array of per-span explanations.

### List available correctors

```bash
curl -s -X GET "https://api.vectara.io/v2/hallucination_correctors?limit=10" \
  -H "x-api-key: $VECTARA_API_KEY"
```

Response items look like:

```json
{
  "id": "hcm_520721853",
  "name": "vhc-small-1.0",
  "type": "vectara",
  "description": "Basic model for hallucination correction in AI-generated text.",
  "enabled": true
}
```

Common models: `vhc-small-1.0` (fast, cheaper), `vhc-large-1.0` (higher quality, used in the example below). Use the `name` (not the `id`) in the correction request body's `model_name` field.

### Correct hallucinations

```bash
curl -s -X POST https://api.vectara.io/v2/hallucination_correctors/correct_hallucinations \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "vhc-large-1.0",
    "generated_text": "The Eiffel Tower is located in Berlin and was built in 1925.",
    "documents": [
      { "text": "The Eiffel Tower is a famous landmark located in Paris, France." },
      { "text": "It was built in 1889 and remains one of the most visited monuments in the world." }
    ],
    "query": "Where is the Eiffel Tower and when was it built?"
  }'
```

Response:

```json
{
  "corrected_text": "The Eiffel Tower is located in Paris and was built in 1889.",
  "corrections": [
    {
      "original_text": "Berlin",
      "corrected_text": "Paris",
      "explanation": "The source documents state the Eiffel Tower is in Paris, France."
    },
    {
      "original_text": "1925",
      "corrected_text": "1889",
      "explanation": "Sources indicate the Eiffel Tower was built in 1889."
    }
  ],
  "model": "vhc-large-1.0"
}
```

### Output cases

| Case | What it means |
|---|---|
| `corrected_text == generated_text`, `corrections == []` | VHC found no unsupported claims. The original is fine. |
| `corrected_text` is a minimally edited version, `corrections[]` non-empty | Some spans were rewritten. Serve `corrected_text`; surface `corrections[]` in your debug UI or audit log. |
| `corrected_text == ""` (empty string) | VHC determined the entire input was unsupported and recommends discarding it. **Do not serve the original.** Fall back to a "no answer" message or retry with broader retrieval. |

The optional `query` field enables query-aware correction — it lets VHC consider the expected response format and what claims the question implies, which tightens corrections on incomplete or off-topic generations. Pass it whenever you have it.

### What VHC will not do

- It will not pull in facts that aren't in `documents[]`. If the user asked "when was it built?" and the documents don't mention a date, VHC will remove a hallucinated date — it won't go look one up.
- It will not improve retrieval. If your retrieved passages are off-topic, VHC will mostly delete the generated text. The fix is better retrieval, not more aggressive correction.

## Vectara Open RAG Eval

`open-rag-eval` is an open-source Python toolkit (`pip install open-rag-eval`) for benchmarking RAG pipelines offline. It has native Vectara support but also works with LlamaIndex- and LangChain-built pipelines. It is **not** a Vectara API surface; it runs in your environment, fires queries at whatever RAG endpoint you point it at, and scores the results.

### Metrics it computes

- **UMBRELA** — TREC-RAG-style retrieval relevance.
- **AutoNuggetizer** — answer-coverage scoring without requiring "golden answers."
- **BERTScore**, **ROUGE-L** — text similarity to references when you do have them.
- **Consistency** evaluation across multiple generations (stability under repeated sampling).

No "golden chunk" or "golden answer" labels required to get useful signal — the framework leans on LLM-as-judge techniques researched at the University of Waterloo. You can add custom metrics.

### Sketch — running it

```python
# pip install open-rag-eval
from open_rag_eval import Evaluator, VectaraConnector
from open_rag_eval.metrics import UMBRELA, AutoNuggetizer

connector = VectaraConnector(
    api_key=os.environ["VECTARA_API_KEY"],
    corpus_key="prod-knowledge-base",
    generation_preset_name="vectara-summary-ext-24-05-med-omni",
)

evaluator = Evaluator(
    connector=connector,
    metrics=[UMBRELA(), AutoNuggetizer()],
    queries_path="eval/queries.jsonl",
)

results = evaluator.run()
results.to_csv("eval/results.csv")
```

Inspect runs interactively at [openevaluation.ai](https://openevaluation.ai), or use the bundled Streamlit viewer. Check the GitHub repo for the current connector/metric class names — the package is under active development.

### In CI

Wire it into a `pytest` job or a standalone GitHub Action that runs on PRs touching corpus configs, rerankers, or generation presets. Fail the build if a key metric regresses beyond a tolerance vs. the main branch baseline.

```bash
# pseudo-CI step
pip install open-rag-eval
python eval/run_eval.py --config eval/baseline.yaml --out eval/pr.json
python eval/compare.py --baseline eval/main.json --candidate eval/pr.json --tolerance 0.02
```

Open RAG Eval addresses the offline, pre-deployment question ("does config A beat config B on my labeled set?"). HHEM addresses the per-request, online question ("is this specific generation grounded?"). Use both.

## Query observability (in-console)

For diagnosing individual past queries, Vectara's console provides query-level and session-level traces. There is **no general-purpose corpus-query trace REST endpoint** — this is a UI feature.

- **Corpus queries:** Open the corpus, go to the **Analysis** tab (or the corpus **Usage** tab). The query history table lists each query ID, time, mode, text, response, latency, and any error. Click a query ID to see configuration (lambda, reranker, context window, generation preset), retrieved chunks with scores, the generated summary, and the factual consistency score (when enabled at query time). Per-stage latency is broken out across **search**, **rerank**, **generation**, and **factual consistency score** spans.
- **Agent sessions:** Open the agent, go to the **Sessions** tab. Each row is a session; click in to see every `input_message`, `tool_input`, `tool_output`, and `agent_output` with timing. Toggle the Debug panel for full event JSON.
- **Bulk export for agents only:** `GET /v2/agent_analytics/traces` returns one record per session with spans for every tool call, step transition, and model turn. Ship to Datadog/BigQuery/Snowflake for cross-session dashboards. Corpus query history does not have a parallel bulk export — use the console.

To get factual consistency scores to *appear* in the query history, you must have set `enable_factual_consistency_score: true` on the original query. Otherwise the column is empty.

## End-to-end worked example — query, score, correct

Putting it all together: query a corpus with inline HHEM, fall through to VHC if the score is below threshold, re-score after correction.

```bash
#!/usr/bin/env bash
set -euo pipefail
: "${VECTARA_API_KEY:?}"

QUERY="Where is the Eiffel Tower and when was it built?"
THRESHOLD=0.7

# 1. Query the corpus with inline HHEM.
RESP=$(curl -s -X POST https://api.vectara.io/v2/query \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<JSON
{
  "query": "$QUERY",
  "search": {
    "corpora": [{ "corpus_key": "monuments-kb" }],
    "limit": 10
  },
  "generation": {
    "generation_preset_name": "vectara-summary-ext-24-05-med-omni",
    "max_used_search_results": 5,
    "enable_factual_consistency_score": true,
    "response_language": "eng"
  }
}
JSON
)

SUMMARY=$(echo "$RESP" | jq -r '.summary')
SCORE=$(echo "$RESP"  | jq -r '.factual_consistency_score')
echo "HHEM score: $SCORE"

# 2. If grounded enough, serve as-is.
if (( $(echo "$SCORE >= $THRESHOLD" | bc -l) )); then
  echo "ACCEPT: $SUMMARY"
  exit 0
fi

# 3. Below threshold -> run VHC on the summary, using the same retrieved
#    passages as the source documents (this is the contract: VHC corrects
#    against what was actually shown to the generator).
DOCS=$(echo "$RESP" | jq '[.search_results[].text | {text: .}]')

CORRECTED=$(curl -s -X POST https://api.vectara.io/v2/hallucination_correctors/correct_hallucinations \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg q "$QUERY" --arg s "$SUMMARY" --argjson d "$DOCS" '{
    model_name: "vhc-large-1.0",
    generated_text: $s,
    documents: $d,
    query: $q
  }')")

CORRECTED_TEXT=$(echo "$CORRECTED" | jq -r '.corrected_text')

# 4. If VHC discarded everything, fall back.
if [[ -z "$CORRECTED_TEXT" ]]; then
  echo "REJECT: VHC removed entire summary as unsupported. Returning fallback."
  echo "I don't have enough information in the knowledge base to answer that."
  exit 0
fi

# 5. Re-score the corrected text standalone to confirm we improved.
RESCORE=$(curl -s -X POST https://api.vectara.io/v2/evaluate_factual_consistency \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg s "$CORRECTED_TEXT" --argjson srcs "$(echo "$DOCS" | jq '[.[].text]')" '{
    model_parameters: { model_name: "hhem_v2.3" },
    generated_text: $s,
    source_texts: $srcs,
    language: "eng"
  }')")

NEW_SCORE=$(echo "$RESCORE" | jq -r '.score')
echo "Post-VHC HHEM score: $NEW_SCORE"
echo "SERVE: $CORRECTED_TEXT"
```

Two notes on the flow:

- VHC's `documents[]` should be the same passages that were retrieved and used in generation — i.e. `search_results[*].text` from the query response, not a fresh search. You're correcting against what the LLM actually saw.
- The post-correction HHEM score is the source of truth for whether to serve the corrected text. Don't assume "corrected" means "good" — if the post-VHC score is still below threshold, fall back.

## Common Mistakes

- **Treating the HHEM score as a probability of correctness.** It is a groundedness score against the supplied sources. Garbage sources, high score, wrong answer is a valid (and common) failure mode. Fix retrieval, not the threshold.
- **Expecting VHC to invent missing facts.** If the documents don't say when the tower was built, VHC will remove the hallucinated date — it will not look one up. If you need synthesis from outside the corpus, that's a retrieval or web-tool problem.
- **Treating in-console query observability as a public REST API.** There is no general "fetch the full trace for query ID X" endpoint for corpus queries. For agent sessions, `GET /v2/agent_analytics/traces` exists; for corpus queries, the console's Analysis tab is the surface.
- **Calling Open RAG Eval a Vectara API.** It's an open-source Python package on GitHub (`vectara/open-rag-eval`) / PyPI (`open-rag-eval`). No API key sends data to a Vectara eval service — your dataset stays local; the package calls whatever RAG endpoint you configure it against.
- **Mixing inline HHEM and standalone HHEM thresholds.** They are the same model and the scores are comparable, but only when the `source_texts` you pass standalone are the same passages that were used to generate. If you pass a broader or narrower set, you're measuring something different.
- **Setting `language` to a 2-letter code on the eval endpoint.** It expects ISO 639-3 (`eng`, `fra`, `deu`, `spa`, `por`, `ara`, `kor`, `zho`, `rus`, `jpn`, `hin`). `en`/`fr`/`de` will 422.
- **Calling VHC without the `query` field when you have one.** Query-aware correction is materially better on incomplete or off-topic generations — pass the original user question whenever you have it.
- **Re-correcting an already-corrected output in a loop.** One VHC pass is the intended pattern. If the post-correction score is still bad, the answer is to fall back or fix retrieval, not to chain corrections.

## References

### Concepts
- [Hallucination evaluation (HHEM)](https://docs.vectara.com/docs/hallucination-and-evaluation/hallucination-evaluation.md)
- [Vectara Hallucination Corrector](https://docs.vectara.com/docs/hallucination-and-evaluation/vectara-hallucination-corrector.md)
- [Query observability](https://docs.vectara.com/docs/hallucination-and-evaluation/query-observability.md)
- [Open RAG Eval framework](https://docs.vectara.com/docs/hallucination-and-evaluation/open-eval-framework.md)
- [Factual consistency evaluation](https://docs.vectara.com/docs/hallucination-and-evaluation/factual-consistency-evaluation.md)
- [Hallucination correctors](https://docs.vectara.com/docs/hallucination-and-evaluation/hallucination-correctors.md)

### REST API
- [Evaluate factual consistency](https://docs.vectara.com/docs/rest-api/evaluate-factual-consistency.md) — `POST /v2/evaluate_factual_consistency`
- [Correct hallucinations](https://docs.vectara.com/docs/rest-api/correct-hallucinations.md) — `POST /v2/hallucination_correctors/correct_hallucinations`
- [List hallucination correctors](https://docs.vectara.com/docs/rest-api/list-hallucination-correctors.md) — `GET /v2/hallucination_correctors`
- [OpenAPI spec](https://api.vectara.io/v2/openapi.json) — `evaluate_factual_consistency`, `hallucination_correctors` schemas

### Open RAG Eval
- [github.com/vectara/open-rag-eval](https://github.com/vectara/open-rag-eval) — source, install, docs
- [openevaluation.ai](https://openevaluation.ai) — hosted results viewer
