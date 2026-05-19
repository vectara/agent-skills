---
name: vectara-agent-orchestration
description: Build multi-step Vectara agents — state machines via first_step + steps[] + next_steps, structured-output gating, sub-agent delegation, MCP/lambda tools, runtime-constrained generation, and cross-session approval workflows.
tags: [vectara, agents, orchestration, multi-step, sub-agent, workflow, approval]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Agents — Multi-step Orchestration and Workflows

You are helping a developer build agents that span multiple phases — classification, approval gates, delegation to specialist sub-agents, cross-session callbacks, and write actions that require human sign-off. Single-step conversational agents (covered by `vectara-agents`) work for simple chat; this skill is for workflows.

## When to use this skill

- The workflow has discrete phases with different tools or prompts per phase.
- A step needs to *structurally* deny tools (human-in-the-loop gates).
- One agent should delegate to a specialist agent with its own toolset.
- The LLM must pick from a runtime-fetched list (no hard-coded enum is acceptable).
- An external channel (Slack, email, webhook) needs to wake a paused session.
- You need structured output from one step to drive branching to another.

## Multi-step state machine

A multi-step agent has `first_step` (the entry, a single object) and `steps[]` (an array of named steps). Transitions are declared on each step via `next_steps`:

```json
{
  "first_step": {
    "name": "kb_check",
    "type": "conversational",
    "instructions": [...],
    "output_parser": {
      "type": "json",
      "json_schema": { "type": "object", "properties": { "deflected": {"type":"boolean"} }, "required": ["deflected"] },
      "store_as": "$.kb"
    },
    "next_steps": [
      { "condition": "get('$.kb.deflected') == true", "step_name": "deflected_end" },
      { "step_name": "classify_with_context" }
    ]
  },
  "steps": [
    { "name": "deflected_end", "type": "conversational", "instructions": [...], "output_parser": {"type":"default"} },
    { "name": "classify_with_context", ... }
  ]
}
```

A step with no `next_steps` is terminal — the agent loop ends after it.

## Conditional `next_steps` via JSONPath

`next_steps` is evaluated top-to-bottom. Each entry has an optional `condition` (a JSONPath-expression-with-comparators) and a `step_name`. The first matching condition wins; the final entry with no `condition` is the default:

```json
"next_steps": [
  { "condition": "get('$.classification.has_sufficient_info') == false", "step_name": "gather_info" },
  { "condition": "get('$.classification.recommended_decision') == 'deny_with_reason'", "step_name": "notify_denial" },
  { "condition": "get('$.classification.app') == 'oracle_fusion'", "step_name": "pick_oracle_role" },
  { "step_name": "manager_gate" }
]
```

JSONPath reads use `get('$.path')`, not bare `$.path`. The path must point at something an earlier step wrote via `store_as`.

## `output_parser.json_schema` + `store_as`

`output_parser` does two things at once: constrain LLM output to a JSON schema, and write the parsed value to a named slot the rest of the session can reference.

```json
"output_parser": {
  "type": "json",
  "json_schema": {
    "type": "object",
    "properties": {
      "app":         { "type": "string", "enum": ["oracle_fusion", "pendo", "aem"] },
      "sensitivity": { "type": "string", "enum": ["low", "medium", "high"] },
      "confidence":  { "type": "number" }
    },
    "required": ["app", "sensitivity", "confidence"]
  },
  "store_as": "$.classification"
}
```

After this step runs, downstream steps can branch on `get('$.classification.app')`, reference `$.classification.sensitivity` in instructions, etc. Use `output_parser: {"type": "default"}` for free-form text steps that don't need to capture structured output.

## Runtime-populated enum (constrained generation from a tool result)

A common need: the LLM must pick from a list that's only known at runtime — role names from a permissions API, ticket categories from an ITSM, etc. Vectara's pattern is to write `enum` into the JSON schema but populate it dynamically from the prior tool call's stored result:

```json
"output_parser": {
  "type": "json",
  "json_schema": {
    "type": "object",
    "properties": {
      "role_binding": {
        "type": "string",
        "description": "Must be one of $.candidates. Enum is populated at runtime from the wolken_get_access_types result."
      }
    }
  }
}
```

The step's tool fetches the candidates (`wolken_get_access_types`), the instruction tells the LLM "store the result as `$.candidates`," and the structured output enforces the pick against that slot. Prevents hallucinated role names that look valid but don't exist.

## Step-scoped tools via `allowed_tools`

All tools are declared once at the top-level `tool_configurations`. Each step can whitelist a subset via `allowed_tools`. Without `allowed_tools`, the step has access to every tool:

```json
{
  "name": "manager_gate",
  "type": "conversational",
  "allowed_tools": ["post_approval_request"],
  "instructions": [...],
  "reminders": [
    { "type": "inline", "template": "You are at the GATE. Provisioning tools are not attached here." }
  ]
}
```

This is structural enforcement of a human-in-the-loop gate — the LLM *cannot* call the write tool here because it's not attached, regardless of what its prompt says. Pair with `reminders` to keep the constraint in-context.

## `reminders` — step-scoped guardrail prompts

```json
"reminders": [
  { "type": "inline", "template": "Your only job in this step is to post the approval request to Slack and notify the requestor. The handler agent creates the Wolken ticket on approval." }
]
```

Distinct from `instructions` (which set up the step once); reminders re-inject every turn as a system nudge. Use sparingly — they cost context tokens on every turn.

## `sub_agent` tool with session forwarding

Delegate a focused task to a specialist agent. The parent calls a `sub_agent` tool; the platform spins up (or resumes) a session on the named child agent and runs it. Parent metadata can be forwarded into the child's `session.metadata`:

```json
"wolken_specialist": {
  "type": "sub_agent",
  "description_template": "Delegate to the Wolken specialist for approver lookup and ticket creation.",
  "sub_agent_configuration": {
    "agent_key": "wolken-connector-agent",
    "session_mode": "persistent",
    "session_metadata": {
      "parent_session":      { "$ref": "session.key" },
      "requestor_email":     { "$ref": "session.metadata.user_email" },
      "requestor_name":      { "$ref": "session.metadata.user_name" },
      "requestor_department":{ "$ref": "session.metadata.user_department" }
    }
  }
}
```

`session_mode`:
- `persistent` — reuse the same sub-agent session across parent turns
- `ephemeral` — fresh session per invocation
- `llm_controlled` — the LLM picks via tool arg

`{"$ref": "session.key"}` forwards the parent session id, useful when the child needs to call back via `POST /v2/agents/{parent_key}/sessions/{key}/events`.

## `argument_override` to lock tool inputs

Any tool argument set in `argument_override` is *locked* — the LLM cannot override it. Use for tenant IDs, target corpora, channel IDs, anything that must not be model-controlled:

```json
"index_decision": {
  "type": "dynamic_vectara",
  "tool_id": "tol_vectara_core_document_index_20260220",
  "description_template": "Index the decision into the precedent corpus. corpus_key is LOCKED via argument_override.",
  "argument_override": {
    "corpus_key": "csg-past-approvals"
  }
}
```

Works on every tool type (`mcp`, `dynamic_vectara`, `lambda`, `web_get`). Locked keys win the merge with model-supplied args. Document the lock in the tool's `description_template` so the model knows not to try.

## Tool types: `mcp`, `dynamic_vectara`, `lambda`

| Type | What | When to use |
|---|---|---|
| `mcp` | Tool exposed by a registered tool_server (Model Context Protocol) | Customer-built integration servers, anything beyond Vectara built-ins |
| `dynamic_vectara` | Vectara-shipped built-in tool (Slack post, core_document_index, Wolken toolkit) | Use canonical `tol_vectara_*` ids documented in platform release notes |
| `lambda` | Customer Python code in a sandboxed runtime (no network) | Deterministic transforms, payload builders, validators, format adapters |

```json
"workday_identity_lookup": {
  "type": "mcp",
  "tool_id": "tol_workday_mcp_identity_lookup",
  "argument_override": { "tenant": "csg-prod" }
}
```

```json
"post_approval_request": {
  "type": "dynamic_vectara",
  "tool_id": "tol_vectara_slack_post_message",
  "argument_override": {
    "connector_id": { "$ref": "session.metadata.connector_id" },
    "channel_id":   "#access-approvals"
  }
}
```

```json
"build_payload": {
  "type": "lambda",
  "tool_id": "tol_my_payload_builder",
  "description_template": "Build the per-application request payload. Runs in the sandbox; no network."
}
```

**Lambda gotcha:** bare `list` parameters in the lambda's input schema break the agent loop with a schema error. Use `str` + `json.loads()` inside the lambda instead.

`mcp` tool_ids must exist on the account — register the tool_server first (`POST /v2/tool_servers`, then `POST /v2/tool_servers/{id}/sync`, then `GET /v2/tool_servers/{id}/tools` for the canonical names). Agents fail at import time if a referenced `tool_id` doesn't resolve.

## Connector-driven agents (Slack inbound)

A `Connector` turns external events into `input_message` events on an agent session — e.g. every Slack message in a watched channel triggers the agent to process it. Register a connector *after* the agent is created:

```bash
POST /v2/agents/{agent_key}/connectors
{
  "name": "csg-access-approvals-slack",
  "type": "slack",
  "configuration": {
    "type": "slack",
    "bot_token":      "$CSG_APPROVALS_SLACK_BOT_TOKEN",
    "signing_secret": "$CSG_APPROVALS_SLACK_SIGNING_SECRET",
    "api_app_id":     "$CSG_APPROVALS_SLACK_APP_ID"
  }
}
```

Use this for the *handler* side of an approval workflow: the intake agent posts a request to Slack, a separate handler agent watches the channel via its Connector and reacts to replies.

## Cross-session event POST (waking up a paused session)

A step can be designed to *wait* for an external signal — e.g. `manager_gate` sits until an approval arrives. The external system (or a handler agent) wakes it by POSTing an `input_message` to the same session:

```bash
POST /v2/agents/{intake_agent_key}/sessions/{intake_session_key}/events
{
  "type": "input_message",
  "messages": [{
    "type": "text",
    "content": "approval_status_update: approved by bob.kerns@csg.com, wolken_ticket_id=WLK-48211"
  }]
}
```

The paused step re-evaluates its `next_steps` against the new state. This is how the intake → Slack → handler → intake bridge works.

## Two-agent gate (structural human-in-the-loop)

The pattern for irreversible writes:

1. **Intake agent** has the classification and read-only tools. Its `manager_gate` step has `allowed_tools: ["post_approval_request"]` and **does not include** the write tool. The model literally can't call it.
2. **Handler agent** has only the write tool (`wolken_create_ticket`) and is fed by a Slack Connector. It only fires when a human replies in the approvals channel.

This is *deployment-pattern* enforcement, not prompt enforcement. Don't trust "the model won't call the write tool" — split the tool into a separate agent so it's literally unreachable from the intake side.

## Reusable instruction blocks via `skills`

Define named instruction snippets once and reference them across steps:

```json
"skills": {
  "citation_requirements": {
    "description": "Citation rules for classification, approver messages, and audit-log entries.",
    "content": "Every policy claim cites [policy:<doc_id>#<section>]. Every precedent claim cites [precedent:<request_id>]..."
  },
  "kb_deflection_guard": {
    "description": "How to use the KB search result. Invoke at the start of every conversation.",
    "content": "..."
  }
}
```

Step instructions invoke them by name: *"Apply the kb_deflection_guard skill."* Skills are not auto-applied — the model must be told to invoke them. Cleaner than inlining the same policy text into multiple step templates.

## `corpora_search` configuration

Standard retrieval tool config; the same shape mirrors `POST /v2/queries`:

```json
"precedent_search": {
  "type": "corpora_search",
  "query_configuration": {
    "search": {
      "corpora": [{
        "corpus_key": "csg-past-approvals",
        "lexical_interpolation": 0.025,
        "metadata_filter": "doc.decision IN ('approved','denied','approved_with_conditions')"
      }],
      "limit": 6,
      "context_configuration": { "sentences_before": 1, "sentences_after": 4 },
      "reranker": { "type": "customer_reranker", "reranker_name": "Rerank_Multilingual_v1" }
    }
  }
}
```

- `lexical_interpolation` 0.025–0.05 for vector-heavy semantic search; higher for keyword bias.
- `metadata_filter` uses SQL-ish syntax over `doc.<filter_attribute>`. The attribute must be declared `indexed: true` in the corpus config.
- `reranker.type: "customer_reranker"` with `reranker_name: "Rerank_Multilingual_v1"` is the standard multilingual rerank shape.

## Corpus + seed documents

Filter attributes must be declared on the corpus before they can appear in `metadata_filter`:

```json
{
  "key": "csg-past-approvals",
  "filter_attributes": [
    { "name": "decision",      "level": "document", "indexed": true, "type": "text" },
    { "name": "duration_days", "level": "document", "indexed": true, "type": "integer" },
    { "name": "sox_in_scope",  "level": "document", "indexed": true, "type": "boolean" }
  ]
}
```

Seed structured documents so the agent has context from day one (denial precedents matter as much as approval precedents):

```json
{
  "id": "req_2024Q4_mark_audit",
  "type": "structured",
  "metadata": {
    "decision": "approved",
    "system":   "oracle_fusion",
    "duration_days": 60
  },
  "sections": [{ "text": "Mark Tobias (senior auditor, internal audit) requested..." }]
}
```

`metadata` field names must match `filter_attributes.name` exactly and types must align (integer dates as YYYYMMDD ints, not strings). `sections[].text` is the searchable body.

## Template-string interpolation vs `$ref`

Two distinct syntaxes — don't mix them:

- `{"$ref": "session.metadata.user_email"}` — inside a JSON value position (substitutes a typed value at session-create time).
- `"$session.metadata.user_email"` — inside an instruction template *string* (substituted as text when the prompt is rendered).

Use `$ref` for tool config, template strings for prompts and reminders.

## Hidden registration metadata (`_status`, `_note`, `_comment`)

Underscore-prefixed keys are convention for "ignored by the runtime, read by humans." Embed deployment notes in config files without affecting the API call:

```json
{
  "_status": "POC week 1 deliverable. Vectara field engineering builds this.",
  "_note":   "POST this to /v2/agents/{key}/connectors AFTER the agent is created."
}
```

Strict-mode schema validators may reject them — keep them out of any payload that needs to survive a validator (use them in source JSON your import script reads and strips).

## Common mistakes to avoid

- Putting transitions on the wrong step. `next_steps` lives on the step it's *leaving from*, not the target.
- Reading a `store_as` slot before the step that writes it has run.
- Hard-coding an enum the model can't trust — use the runtime-populated-enum pattern.
- Trusting prompt-based "don't call this tool" instead of removing the tool from `allowed_tools`.
- Registering a connector inside the agent JSON (the connectors API is separate — `POST /v2/agents/{key}/connectors`).
- Referencing an `mcp` `tool_id` that doesn't exist on the account. Register the tool_server and sync first.
- Mixing `{"$ref": "..."}` and `"$session.metadata.X"` syntaxes in the wrong positions.
- Including underscore-prefixed `_note`/`_status` fields in payloads sent to a strict-mode validator.

## Working examples

The full multi-step + sub-agent + connector pattern is implemented in:

- **CSG access orchestrator** (multi-step state machine with kb_check → classify → per-app picker → manager_gate → provision_and_learn): `vectara-config/access-orchestrator.json` in the CSG demo package.
- **Wolken specialist sub-agent** (focused write-only specialist): `wolken-connector-agent.json`.
- **Approval handler** (Slack-connector-driven cross-session callback): `csg-approval-handler.json`.

## References

- [Multi-step Agents](https://docs.vectara.com/docs/agents/agents.md)
- [Sub-agents](https://docs.vectara.com/docs/agents/subagents.md)
- [Tools](https://docs.vectara.com/docs/agents/tools.md) — `mcp`, `dynamic_vectara`, `lambda`, `web_get`
- [Instructions](https://docs.vectara.com/docs/agents/instructions.md)
- [Sessions](https://docs.vectara.com/docs/agents/sessions.md)
- [OpenAPI Spec](https://api.vectara.io/v2/openapi.json) — `agents`, `agent_sessions`, `agent_connectors`, `tool_servers`
