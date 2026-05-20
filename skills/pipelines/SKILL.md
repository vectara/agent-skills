---
name: vectara-pipelines
description: Configure Vectara pipelines — automated flows that pull records from a source (S3, web, SharePoint) and fan each one into a fresh agent session. Covers source config, cron/interval/manual triggers, incremental vs full_refresh watermarks, condition and judge-agent verification, the dead-letter queue lifecycle (list, process, manual add, delete), run management (trigger, list, get, events, cancel), and pipeline CRUD with PUT-vs-PATCH semantics.
tags: [vectara, pipelines, ingestion, s3, dead-letters, watermark, agents]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Pipelines — Continuous Ingestion into Agents

You are helping a developer configure a Vectara pipeline. A pipeline is the platform's bulk-ingest abstraction: it pulls records from an external system on a schedule, opens a fresh agent session for each record, and uploads that record as the session's first input. The agent decides what to do with it — index into a corpus, extract structured fields, route, or discard. The pipeline handles fetching, concurrency, watermarking, retries, and dead-lettering.

## When to use this skill

- Continuously sync an S3 bucket (or other source) into a Vectara corpus through an agent that does the indexing.
- Run an extraction/classification agent over every object that lands in a bucket — periodic, not interactive.
- Build a recurring data sync where each source record needs full agent autonomy (multiple tool calls, structured output, verification) rather than a one-shot transform.
- Inspect or retry records that failed an earlier ingest (dead-letter queue).

If you need an interactive agent that responds to user messages, use `vectara-agents`. If you need a single scheduled execution of an agent with a fixed prompt (one session per trigger, not one per record), that's an **agent schedule**, not a pipeline.

## Critical Rules

1. **API base URL** is `https://api.vectara.io/v2/`. Pipelines live at `/v2/pipelines/{pipeline_key}`.
2. **A pipeline targets an agent, not a corpus.** The `transform` field points at an `agent_key`. The agent owns the indexing logic. Don't try to point a pipeline directly at a corpus — there's no such field.
3. **Each source record creates one fresh agent session.** The record's contents are uploaded as the session's first input. One record → one session, every time. There is no batching of records into a single session.
4. **The watermark is the source of truth for what's been processed.** In `incremental` mode (the default), the pipeline stores a source-specific watermark (e.g. an S3 `lastModified` upper bound) and only fetches records past it on the next run. Don't bypass this by mutating source data manually — the pipeline won't pick up the change.
5. **Only one run can be active per pipeline at a time.** Scheduled triggers that fire mid-run are silently skipped; manual triggers and dead-letter retries return `409 Conflict`.
6. **Cancel is cooperative.** `POST /runs/{run_id}/cancel` requests a stop; the run finishes the current checkpoint, then transitions to `cancelled`. It's not immediate. Runs that already reached a terminal state return `409`.
7. **PUT replaces, PATCH merges.** `PUT /v2/pipelines/{key}` (replace) requires the full pipeline body. `PATCH` (update) only changes the fields you send; omitted fields are preserved. For source credential rotation, use PATCH — re-sending `access_key_id` and `secret_access_key` updates only those.
8. **Deleting a pipeline doesn't delete its agent or any sessions the pipeline created.** Those are independent resources. Clean them up separately if needed.
9. **Authentication**: `x-api-key: <key>` or `Authorization: Bearer <token>`. Source credentials (e.g. `secret_access_key`) are encrypted at rest with a customer-specific key and never returned on read — GET responses omit them.

## Pipeline anatomy

Every pipeline has five parts. Get them straight before writing config.

| Part | Field | What it does |
|---|---|---|
| **Source** | `source.type` (`s3`, `web`, ...) | Where records come from. Each object/page/item = one record. Credentials are write-only. |
| **Trigger** | `trigger.type` (`cron`, `interval`, `manual`) | When the pipeline runs. Cron in UTC; interval as ISO-8601 duration; manual = `/trigger` only. |
| **Transform** | `transform.type: "agent"` + `agent_key` | The agent that processes each record. Currently only agent transforms exist. |
| **Sync mode** | `sync_mode` (`incremental` \| `full_refresh`) | Whether to advance a watermark and skip unchanged records, or reprocess everything every run. |
| **Verification** (optional) | `transform.verification` | Stricter success criteria — a UserFn condition or a judge agent. Failures land in the DLQ. |

The watermark itself is read-only state, not configuration. It appears on `GET /pipelines/{key}` as `watermark.value` (opaque, source-specific) and `watermark.updated_at`. It's null until the pipeline has completed at least one successful run.

## S3 source

S3 is the canonical source. It also covers any S3-compatible service (MinIO, Ceph, R2) via `endpoint_url`.

```json
{
  "source": {
    "type": "s3",
    "bucket": "my-documents-bucket",
    "region": "us-east-1",
    "prefix": "legal/contracts/",
    "access_key_id": "AKIA...",
    "secret_access_key": "wJal...",
    "endpoint_url": "https://minio.example.com:9000"
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `bucket` | yes | S3 bucket name. |
| `region` | yes | E.g. `us-east-1`. Required even for non-AWS S3-compatible services. |
| `access_key_id` | yes | Write-only. Never returned on read. |
| `secret_access_key` | yes | Write-only. Never returned on read. |
| `prefix` | no | Scopes ingestion to a key prefix. Folder markers (keys ending `/`) are skipped automatically. |
| `endpoint_url` | no | For non-AWS S3. Omit for AWS. |

**Polling and watermark behaviour.** Each run lists objects via `ListObjectsV2` (scoped by `prefix`), captures an upper-bound timestamp at the start of the run, and only processes objects with `lastModified <= upper_bound`. Objects added mid-run wait for the next run. After a successful run, the watermark advances to that upper bound; subsequent incremental runs filter by `lastModified > watermark`.

**Required IAM**: `s3:ListBucket` on the bucket and `s3:GetObject` on the prefix.

**Deletes aren't tracked.** A deleted S3 object simply stops appearing in listings — the pipeline won't reach back and remove anything the agent already indexed downstream. Handle deletions separately if your corpus needs to shrink.

## Worked example — S3 ingest, end to end

Assume the agent `doc-indexer` already exists (created via `POST /v2/agents` per `vectara-agents`) and uses the `structured_document_index` tool to push each file into corpus `legal-corpus`.

```bash
# 1. Create the pipeline — manual trigger so we can drive it explicitly.
curl -X POST https://api.vectara.io/v2/pipelines \
  -H "x-api-key: $VECTARA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "key": "legal-docs-ingest",
    "name": "Legal Docs Ingest",
    "description": "S3 -> doc-indexer agent -> legal-corpus",
    "source": {
      "type": "s3",
      "bucket": "acme-legal",
      "region": "us-east-1",
      "prefix": "contracts/",
      "access_key_id": "AKIA...",
      "secret_access_key": "wJal..."
    },
    "trigger": { "type": "manual" },
    "transform": {
      "type": "agent",
      "agent_key": "doc-indexer",
      "verification": {
        "type": "condition",
        "expression": "get('$.tools.structured_document_index.outputs.latest') != null"
      }
    },
    "sync_mode": "incremental"
  }'

# 2. Trigger a run.
RUN_ID=$(curl -s -X POST https://api.vectara.io/v2/pipelines/legal-docs-ingest/trigger \
  -H "x-api-key: $VECTARA_API_KEY" | jq -r '.id')

# 3. Poll the run.
curl -s https://api.vectara.io/v2/pipelines/legal-docs-ingest/runs/$RUN_ID \
  -H "x-api-key: $VECTARA_API_KEY"
# { "id": "run_...", "status": "running", "records_fetched": 42, "records_processed": 10, "records_failed": 0, ... }

# 4. Stream the event timeline (record-level lifecycle).
curl -s "https://api.vectara.io/v2/pipelines/legal-docs-ingest/runs/$RUN_ID/events?order=asc" \
  -H "x-api-key: $VECTARA_API_KEY"

# 5. After completion, list dead letters for this run.
curl -s "https://api.vectara.io/v2/pipelines/legal-docs-ingest/dead_letters?last_run_id=$RUN_ID" \
  -H "x-api-key: $VECTARA_API_KEY"

# 6. Reprocess all pending dead letters in a fresh retry run.
curl -X POST https://api.vectara.io/v2/pipelines/legal-docs-ingest/dead_letters/process \
  -H "x-api-key: $VECTARA_API_KEY"
```

Switch `trigger.type` to `cron` (`"expression": "0 */6 * * *"` = every 6 hours UTC) or `interval` (`"duration": "PT1H"`) once the manual loop looks healthy.

## Pipeline status and run status

These are two separate state machines — don't conflate them.

**Pipeline status** (`GET /pipelines/{key}.status`):

| Status | Meaning |
|---|---|
| `initializing` | Just created; first run hasn't happened. |
| `active` | Healthy; will run on its trigger. |
| `paused` | Disabled or stopped accepting triggers. Set `enabled: false` to pause. |
| `error` | Configuration or repeated-run failure; `status_message` has details. |

**Pipeline run status** (`GET /pipelines/{key}/runs/{run_id}.status`):

| Status | Meaning |
|---|---|
| `running` | Currently executing — records are being fetched and dispatched to agent sessions. |
| `completed` | All discovered records have been attempted. `records_failed > 0` is still a `completed` run; check the DLQ for details. |
| `failed` | Run itself errored (source unreachable, internal error). `error` field has the message. |
| `cancelled` | A `/cancel` request was honored at the next checkpoint. |

Run `trigger_type` is `scheduled`, `manual`, or `retry` (the value used when `dead_letters/process` creates a run).

## Run events

`GET /pipelines/{key}/runs/{run_id}/events` returns a customer-observability timeline (not raw Temporal history). Filter by `type` (repeatable) or `source_record_id`.

| Event type | What it says |
|---|---|
| `run_started` | Includes `trigger_type`, `sync_mode`, `start_watermark`, `end_watermark` (the upper bound this run won't cross). |
| `record_processing` | Per-record lifecycle. `status` is `started`, `completed`, or `failed`. `session_key` is present on `completed` (always) and `failed` (if a session was opened); `skipped: true` means a prior successful session at the same watermark already exists; `dead_lettered: true` flags whether a `failed` record was written to the DLQ. |
| `watermark_advanced` | Emitted at the end of the run with the new `watermark` value. Absent on `cancelled`/`failed`/`timed_out`. |
| `run_completed` | Terminal. `status` is `completed`, `failed`, `cancelled`, or `timed_out`. |

This is the right endpoint to debug "why didn't record X get processed?" — filter by `source_record_id` and look at the lifecycle.

## Dead-letter lifecycle

A dead letter is created when the agent session throws, or when verification (condition or judge) returns false. The rest of the run continues; the run finishes `completed` with `records_failed > 0`.

DLQ entries are scoped per `(pipeline, source_record_id)` — the same record failing across multiple runs **upserts** the existing entry and bumps `attempt_count`. Successful reprocessing **deletes** the entry from the table.

```json
{
  "id": "pdl_a1b2c3d4-...",
  "source_record_id": "contracts/broken.pdf",
  "status": "pending",
  "error_message": "Failed to process content: unsupported format",
  "attempt_count": 2,
  "origin": "pipeline",
  "last_run_id": "run_legal-docs_scheduled_abc123",
  "created_at": "2026-05-14T10:00:00Z",
  "updated_at": "2026-05-14T16:00:00Z"
}
```

`source_record_id` format depends on the source: S3 = object key (`contracts/broken.pdf`), SharePoint = drive item ID, Web = canonicalized URL.

| Field | Values | Meaning |
|---|---|---|
| `status` | `pending`, `retrying` | `pending` = waiting for someone to retry it. `retrying` = currently being reprocessed by an in-flight retry run. |
| `origin` | `pipeline`, `manual` | `pipeline` = auto-created by a failed record. `manual` = you explicitly POSTed it via `/dead_letters`. |
| `attempt_count` | integer | Number of times this record has been attempted. Increments on every retry that re-fails. |

### Actions

| Goal | Call |
|---|---|
| Inspect failures | `GET /pipelines/{key}/dead_letters?status=pending&last_run_id=run_...&origin=pipeline` |
| Retry all pending | `POST /pipelines/{key}/dead_letters/process` with empty body |
| Retry specific records | `POST /pipelines/{key}/dead_letters/process` with `{"source_record_ids": ["contracts/a.pdf", ...]}` |
| Retry everything from a bad run | `POST /pipelines/{key}/dead_letters/process` with `{"last_run_id": "run_..."}` |
| Force-reprocess a record the agent incorrectly skipped | `POST /pipelines/{key}/dead_letters` with `{"source_record_id": "contracts/x.pdf", "error_message": "Agent skipped this in error"}` — creates a DLQ entry with `origin: manual` |
| Permanently dismiss a known-bad record | `DELETE /pipelines/{key}/dead_letters/{dead_letter_id}` |

`/dead_letters/process` creates a new pipeline run with `trigger_type: retry`. If the pipeline already has a run in flight, the call returns `409 Conflict`. Deleting a DLQ entry only removes the queue entry; if the same record fails again on a future run, a fresh entry is created.

## Verification

Default behaviour: any agent session that doesn't throw counts as success. Add `transform.verification` to enforce stricter criteria. A failed verification dead-letters the record.

**Condition verification.** Lightweight, evaluated locally with no extra agent call. Uses the same UserFn language as User Defined Function rerankers — `get('$.path')` reads, comparators, `and`/`or`.

```json
{
  "transform": {
    "type": "agent",
    "agent_key": "doc-indexer",
    "verification": {
      "type": "condition",
      "expression": "get('$.output.status') == 'indexed' and get('$.output.sections_indexed', 0) > 0"
    }
  }
}
```

Context available to the expression:

| Path | Value |
|---|---|
| `$.agent.key` | Worker agent's key. |
| `$.session.key` | The session created for this record. |
| `$.output` | Final agent output. Structured output agents expose fields directly (`$.output.confidence`); plain-text agents expose `$.output.text`. Absent if no output. |
| `$.tools.<tool_name>.outputs.latest` | Most recent output of each tool the agent called (raw string, typically JSON). |

**Agent (judge) verification.** A second agent reviews the worker's session and returns `{success, reason}`.

```json
{
  "transform": {
    "type": "agent",
    "agent_key": "doc-indexer",
    "verification": { "type": "agent", "agent_key": "doc-indexer-judge" }
  }
}
```

The judge agent's entry step must have a structured `output_parser` whose schema declares exactly `success: boolean` and `reason: string` (both required). Pipeline create/update rejects judge agents that don't match this shape with `400 Bad Request`. The `reason` is surfaced in `error_message` on the resulting DLQ entry, so write it for a human to read.

```json
"first_step_name": "judge",
"steps": {
  "judge": {
    "instructions": [{ "type": "inline", "name": "judge", "template": "Inspect the worker session and decide success." }],
    "output_parser": {
      "type": "structured",
      "json_schema": {
        "name": "judge_verdict",
        "strict": true,
        "schema": {
          "type": "object",
          "properties": {
            "success": { "type": "boolean" },
            "reason":  { "type": "string" }
          },
          "required": ["success", "reason"],
          "additionalProperties": false
        }
      }
    }
  }
}
```

Use condition when success is a structural check; use a judge when it requires semantic reasoning ("did the agent actually summarize the contract or just paraphrase the title?"). Judges add a full extra agent session per record — much slower and more expensive.

## Pipeline CRUD reference

| Operation | Endpoint | Notes |
|---|---|---|
| Create | `POST /v2/pipelines` | Body: full pipeline. Returns the created pipeline (without secret fields). |
| List | `GET /v2/pipelines?source_type=s3&status=active&enabled=true&filter=legal&limit=10` | All query params optional. `filter` is a regex over name+description. |
| Get | `GET /v2/pipelines/{key}` | Includes `watermark`, `status`, `status_message`. Source credentials omitted. |
| Replace | `PUT /v2/pipelines/{key}` | Full body required. Use when reshaping the pipeline. |
| Update | `PATCH /v2/pipelines/{key}` | Partial. Omit fields to preserve them. Common: rotate `secret_access_key`, swap `agent_key`, toggle `enabled`. |
| Delete | `DELETE /v2/pipelines/{key}` | Does **not** delete the target agent or any sessions the pipeline created. |
| Trigger | `POST /v2/pipelines/{key}/trigger` | Returns the new run. `409` if a run is already in flight. |
| List runs | `GET /v2/pipelines/{key}/runs?status=failed&after=2026-05-01T00:00:00Z` | |
| Get run | `GET /v2/pipelines/{key}/runs/{run_id}` | |
| List run events | `GET /v2/pipelines/{key}/runs/{run_id}/events?type=record_processing&source_record_id=...` | |
| Cancel run | `POST /v2/pipelines/{key}/runs/{run_id}/cancel` | `204` on accept. Cooperative — actual transition to `cancelled` lags. `409` if the run is already terminal. |

## Common mistakes to avoid

- **Pointing the pipeline at a corpus.** There is no `corpus_key` on a pipeline. Pipelines flow into agents. The agent does the indexing via `structured_document_index` or another tool.
- **Forgetting the agent exists.** Pipeline create/update doesn't deeply validate the target agent (beyond key format and, for judge verification, the output schema). If `agent_key` doesn't resolve at run time, every record fails.
- **Manually mutating source records to force reprocessing.** In `incremental` mode, only records with a source watermark past the stored value are picked up. If you re-upload an S3 object with the same key, S3 updates `lastModified` and it'll be reprocessed; but if you "edit" by writing a new object with the same content, the watermark won't notice. To reprocess explicitly, use `dead_letters/process` with `source_record_ids` (creates a retry run that re-fetches just those keys) or temporarily switch the pipeline to `full_refresh`.
- **Watermark misalignment after a manual reprocess.** `dead_letters/process` re-fetches the named records but does **not** rewind the persisted watermark. That's deliberate — you're patching specific failures, not redoing the run. If you actually want to redo a whole time range, switch `sync_mode` to `full_refresh`, trigger a run, then switch back.
- **Expecting `cancel` to be instant.** It isn't — it requests a stop at the next checkpoint. Don't immediately re-trigger; you'll get `409` until the run transitions.
- **Confusing pipeline `status` with run `status`.** Pipeline = `initializing | active | paused | error`. Run = `running | completed | failed | cancelled`. A `completed` run with `records_failed: 17` is normal: the run ran fine, the agent threw on 17 records, and those are now in the DLQ.
- **Deleting a pipeline expecting cleanup.** It removes the pipeline config and its DLQ entries; it does not delete the worker agent, the judge agent, or any of the (potentially thousands of) sessions those agents created. Those persist independently.
- **Using PUT when PATCH would do.** PUT requires re-sending the entire body, including a credential block. PATCH lets you change one field. Use PATCH for routine updates; reserve PUT for full reshapes.
- **Writing a judge agent without the exact output schema.** It must declare `success: boolean` and `reason: string`, both required, in the entry step's `output_parser` (`type: "structured"`, `json_schema.schema` shape — see the verification section). Anything else is rejected at pipeline create/update with `400`.
- **Forgetting only one run runs at a time.** Both scheduled triggers (silently skipped) and manual/retry triggers (`409`) honor this. Stagger pipelines that target the same agent if you care about throughput.

## References

Doc pages — append `.md` to any `https://docs.vectara.com/docs/...` URL to fetch markdown.

### Concepts and guides
- [Pipeline concepts](https://docs.vectara.com/docs/pipelines/pipeline-concepts.md) — five-part anatomy, sync modes, pipeline vs. run.
- [Pipelines quickstart](https://docs.vectara.com/docs/pipelines/pipelines-quickstart.md) — create → trigger → check progress.
- [S3 source](https://docs.vectara.com/docs/pipelines/sources/s3.md) — config, listing semantics, IAM, incremental watermark.
- [Dead letters](https://docs.vectara.com/docs/pipelines/dead-letters.md) — creation rules, dedupe, retry, manual add, delete.
- [Verification](https://docs.vectara.com/docs/pipelines/verification.md) — condition expressions, judge agent schema, when to use which.

### REST API — pipelines
- [Create pipeline](https://docs.vectara.com/docs/rest-api/create-pipeline.md) — `POST /v2/pipelines`
- [List pipelines](https://docs.vectara.com/docs/rest-api/list-pipelines.md) — `GET /v2/pipelines`
- [Get pipeline](https://docs.vectara.com/docs/rest-api/get-pipeline.md) — `GET /v2/pipelines/{key}`
- [Replace pipeline](https://docs.vectara.com/docs/rest-api/replace-pipeline.md) — `PUT /v2/pipelines/{key}`
- [Update pipeline](https://docs.vectara.com/docs/rest-api/update-pipeline.md) — `PATCH /v2/pipelines/{key}`
- [Delete pipeline](https://docs.vectara.com/docs/rest-api/delete-pipeline.md) — `DELETE /v2/pipelines/{key}`
- [Trigger pipeline](https://docs.vectara.com/docs/rest-api/trigger-pipeline.md) — `POST /v2/pipelines/{key}/trigger`

### REST API — runs
- [List pipeline runs](https://docs.vectara.com/docs/rest-api/list-pipeline-runs.md) — `GET /v2/pipelines/{key}/runs`
- [Get pipeline run](https://docs.vectara.com/docs/rest-api/get-pipeline-run.md) — `GET /v2/pipelines/{key}/runs/{run_id}`
- [List pipeline run events](https://docs.vectara.com/docs/rest-api/list-pipeline-run-events.md) — `GET /v2/pipelines/{key}/runs/{run_id}/events`
- [Cancel pipeline run](https://docs.vectara.com/docs/rest-api/cancel-pipeline-run.md) — `POST /v2/pipelines/{key}/runs/{run_id}/cancel`

### REST API — dead letters
- [List dead letters](https://docs.vectara.com/docs/rest-api/list-pipeline-dead-letter-entries.md) — `GET /v2/pipelines/{key}/dead_letters`
- [Get dead letter](https://docs.vectara.com/docs/rest-api/get-pipeline-dead-letter-entry.md) — `GET /v2/pipelines/{key}/dead_letters/{id}`
- [Create dead letter (manual)](https://docs.vectara.com/docs/rest-api/create-pipeline-dead-letter-entry.md) — `POST /v2/pipelines/{key}/dead_letters`
- [Process dead letters (retry)](https://docs.vectara.com/docs/rest-api/process-pipeline-dead-letter-entries.md) — `POST /v2/pipelines/{key}/dead_letters/process`
- [Delete dead letter](https://docs.vectara.com/docs/rest-api/delete-pipeline-dead-letter-entry.md) — `DELETE /v2/pipelines/{key}/dead_letters/{id}`

### Related skills
- `vectara-agents` — building the agent the pipeline targets (tool config, `first_step_name` + `steps{}`, sessions).
- `vectara-agent-orchestration` — multi-step worker agents and judge-agent structured output.

### API reference
- [OpenAPI Spec](https://api.vectara.io/v2/openapi.json) — schemas: `Pipeline`, `PipelineRun`, `PipelineRunEvent`, `PipelineDeadLetterEntry`, `S3SourceConfiguration`.
- [Full documentation index](https://docs.vectara.com/llms.txt)
