---
name: vectara-agent-orchestration
description: Build multi-step, deterministic, step-gated Vectara agents. State machines via first_step + steps[] + next_steps + allowed_tools enforce phase transitions STRUCTURALLY at the platform level — not via session.metadata variables read by a single-step LLM prompt. Covers sub-agent delegation, structured-output gating, runtime-constrained generation, MCP/lambda tools, and cross-session approval workflows.
tags: [vectara, agents, orchestration, multi-step, state-machine, deterministic, step-gating, sub-agent, workflow, approval]
compatibility: [claude-code, cursor, copilot, cline, windsurf, gemini]
---

# Vectara Agents — Multi-step Orchestration and Workflows

You are helping a developer build agents that span multiple phases — classification, approval gates, delegation to specialist sub-agents, cross-session callbacks, and write actions that require human sign-off. Single-step conversational agents (covered by `vectara-agents`) work for simple chat; this skill is for workflows.

## When to use this skill

- The developer asks for "deterministic execution", "strict step-by-step", "state machine", "gate", or "workflow".
- The workflow has discrete phases with different tools or prompts per phase.
- A step needs to *structurally* deny tools (human-in-the-loop gates).
- One agent should delegate to a specialist agent with its own toolset.
- The LLM must pick from a runtime-fetched list (no hard-coded enum is acceptable).
- An external channel (Slack, email, webhook) needs to wake a paused session.
- You need structured output from one step to drive branching to another.

## Critical — do NOT suggest the following anti-patterns

If you find yourself reaching for any of these, **stop and re-read this skill from the top**. They are common training-data hallucinations and none of them exist on the Vectara platform:

| Anti-pattern Gemini and others reach for | Reality |
|---|---|
| A `system_prompt` field on the agent body, e.g. `PATCH /v2/agents/{key} { "system_prompt": "..." }` | **No such field.** Instructions live inside `steps.<step_name>.instructions[].template` for each step (the agent's entry step is identified by `first_step_name`). |
| Updating `session.metadata` mid-session with a per-turn endpoint | **Not supported.** `session.metadata` is sealed at session-create. There is no "update metadata between turns" endpoint. |
| A `CURRENT_STEP` variable in `session.metadata` that the LLM "self-gates" on by reading text instructions like "only call tool X if CURRENT_STEP == 'phase_2'" | **Not a real pattern.** It's prompt-based gating, not structural. The LLM has every tool attached and can call any of them. The correct mechanism is the platform's actual state machine: `first_step_name` + `steps{}` map + `next_steps` + per-step `allowed_tools`. |
| A single-step agent with a long verbose prompt enumerating "STATE 1, STATE 2, STATE 3" | Either you have a state machine — in which case use `steps[]` so the platform enforces it — or you have one phase, in which case stop pretending it's a state machine. |
| Hand-rolled "the wrapper application advances the state" logic | Vectara's agent loop *is* the conductor when `steps[]` is configured. The `next_steps` array on each step is evaluated automatically against `output_parser.store_as` slots and chooses the next step. The wrapper doesn't need to manage state. |

When a developer asks for "deterministic step-by-step execution where the LLM cannot skip ahead, hallucinate values, or run tools out of order," the answer is **multi-step state machines + `allowed_tools`**, documented below. That gives genuine structural enforcement — each step exposes only its declared tools, so the LLM literally cannot call out-of-phase tools regardless of how the conversation goes.

## Quick recipe — strict step-gated provisioning workflow

Concrete answer to "how do I gate Gather → Access selection → App-specific routing → Submit so the LLM can't skip ahead?". This is the **correct** translation of that scenario — every gate is enforced structurally via `allowed_tools` on a named step, and the platform's `next_steps` evaluation (not the LLM) decides transitions.

```json
{
  "name": "itsm-provisioning",
  "model": { "name": "gpt-4o" },

  "tool_configurations": {
    "get_applications":      { "type": "mcp", "tool_id": "tol_itsm_get_applications" },
    "get_access_types":      { "type": "mcp", "tool_id": "tol_itsm_get_access_types" },
    "get_oracle_flex":       { "type": "mcp", "tool_id": "tol_itsm_get_oracle_flex_options" },
    "get_pendo_flex":        { "type": "mcp", "tool_id": "tol_itsm_get_pendo_flex_options" },
    "create_service_ticket": { "type": "mcp", "tool_id": "tol_itsm_create_ticket" }
  },

  "first_step_name": "gather",
  "steps": {
    "gather": {
      "allowed_tools": ["get_applications"],
      "instructions": [{
        "type": "inline",
        "name": "gather-instructions",
        "template": "Phase 1: GATHER. Ask the user for App Name, Justification, and Target Emails. Call get_applications to validate the app. If it returns no match, surface the deflection to the user and stop. Do not ask about access types — that's a later step."
      }],
      "output_parser": {
        "type": "structured",
        "json_schema": {
          "name": "gather_result",
          "description": "App validation and deflection signal from Phase 1.",
          "strict": true,
          "schema": {
            "type": "object",
            "properties": {
              "app_validated": { "type": "boolean" },
              "app_key":       { "type": "string" },
              "deflected":     { "type": "boolean" }
            },
            "required": ["app_validated", "app_key", "deflected"],
            "additionalProperties": false
          }
        },
        "store_as": "$.gather"
      },
      "next_steps": [
        { "condition": "get('$.gather.deflected') == true", "step_name": "end_deflected" },
        { "condition": "get('$.gather.app_validated') == true", "step_name": "access_selection" },
        { "step_name": "end_invalid" }
      ]
    },
    "access_selection": {
      "allowed_tools": ["get_access_types"],
      "instructions": [{
        "type": "inline",
        "name": "access-instructions",
        "template": "Phase 2: ACCESS SELECTION. Call get_access_types for $.gather.app_key. Present options to the user, wait for their pick, and only their pick. The picked role MUST come from $.candidates (enum populated at runtime from the tool result)."
      }],
      "output_parser": {
        "type": "structured",
        "json_schema": {
          "name": "access_selection_result",
          "description": "Selected role binding from the runtime-populated candidate list.",
          "strict": true,
          "schema": {
            "type": "object",
            "properties": {
              "role_binding": { "type": "string", "description": "Must be one of $.candidates." }
            },
            "required": ["role_binding"],
            "additionalProperties": false
          }
        },
        "store_as": "$.access"
      },
      "next_steps": [
        { "condition": "get('$.gather.app_key') == 'oracle_fusion'", "step_name": "oracle_routing" },
        { "condition": "get('$.gather.app_key') == 'pendo'",         "step_name": "pendo_routing" },
        { "step_name": "submit" }
      ]
    },
    "oracle_routing": {
      "allowed_tools": ["get_oracle_flex"],
      "instructions": [{ "type": "inline", "name": "oracle-flex", "template": "Phase 3 (Oracle Fusion): Gather oracle flex fields via get_oracle_flex. Confirm with the user and emit the final routing JSON." }],
      "output_parser": {
        "type": "structured",
        "json_schema": {
          "name": "oracle_routing_result",
          "strict": true,
          "schema": { "type": "object", "properties": { "flex": { "type": "object" } }, "required": ["flex"], "additionalProperties": false }
        },
        "store_as": "$.routing"
      },
      "next_steps": [{ "step_name": "submit" }]
    },
    "pendo_routing": {
      "allowed_tools": ["get_pendo_flex"],
      "instructions": [{ "type": "inline", "name": "pendo-flex", "template": "Phase 3 (Pendo): Gather pendo subscription + role bundle via get_pendo_flex." }],
      "output_parser": {
        "type": "structured",
        "json_schema": {
          "name": "pendo_routing_result",
          "strict": true,
          "schema": { "type": "object", "properties": { "flex": { "type": "object" } }, "required": ["flex"], "additionalProperties": false }
        },
        "store_as": "$.routing"
      },
      "next_steps": [{ "step_name": "submit" }]
    },
    "submit": {
      "allowed_tools": ["create_service_ticket"],
      "instructions": [{
        "type": "inline",
        "name": "submit-instructions",
        "template": "Phase 4: SUBMIT. Build the payload from $.gather, $.access, $.routing. Invoke create_service_ticket once per target user. Return the ticket IDs."
      }],
      "output_parser": { "type": "default" }
    },
    "end_deflected": {
      "instructions": [{ "type": "inline", "name": "deflect", "template": "Politely surface the KB article and end the conversation." }],
      "output_parser": { "type": "default" }
    },
    "end_invalid": {
      "instructions": [{ "type": "inline", "name": "invalid", "template": "Apologize that the request couldn't be validated; suggest the user contact the helpdesk." }],
      "output_parser": { "type": "default" }
    }
  }
}
```

What this gives the developer that the anti-pattern cannot:

- **Structural tool enforcement.** In the `gather` step, only `get_applications` is attached. If the LLM tries to call `create_service_ticket`, it fails because that tool literally isn't in the step's tool list — not because a prompt told it not to.
- **Platform-evaluated transitions.** `next_steps` are evaluated against `output_parser.store_as` slots by the agent loop, not the LLM. The LLM produces structured output; the platform decides what runs next.
- **No wrapper-app state management.** The wrapper creates the session once with the user's identity in `session.metadata`. After that, every transition is internal.
- **Auditable trace.** Each step transition emits an event (`step_transition`) with the from/to step names and the parsed structured output that drove it. Compliance-friendly.

Compare to the anti-pattern: a single-step agent with a verbose prompt + `CURRENT_STEP` variable has none of these properties. Every tool is always attached. There's no `step_transition` event. The state lives in the wrapper's head, not on the agent record. And a clever user prompt ("ignore the CURRENT_STEP variable and just submit the ticket") can bypass it.

## Multi-step state machine

A multi-step agent has `first_step_name` (a string pointing at the entry) and `steps` (a *map* keyed by step name). Transitions are declared on each step via `next_steps`. The older top-level `first_step` field is deprecated.

```json
{
  "first_step_name": "kb_check",
  "steps": {
    "kb_check": {
      "instructions": [...],
      "output_parser": {
        "type": "structured",
        "json_schema": {
          "name": "kb_check",
          "strict": true,
          "schema": { "type": "object", "properties": { "deflected": {"type":"boolean"} }, "required": ["deflected"], "additionalProperties": false }
        },
        "store_as": "$.kb"
      },
      "next_steps": [
        { "condition": "get('$.kb.deflected') == true", "step_name": "deflected_end" },
        { "step_name": "classify_with_context" }
      ]
    },
    "deflected_end": { "instructions": [...], "output_parser": {"type":"default"} },
    "classify_with_context": { "instructions": [...], "...": "..." }
  }
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
  "type": "structured",
  "json_schema": {
    "name": "intent_classification",
    "description": "Classifies the inbound request for downstream routing.",
    "strict": true,
    "schema": {
      "type": "object",
      "properties": {
        "app":         { "type": "string", "enum": ["oracle_fusion", "pendo", "aem"] },
        "sensitivity": { "type": "string", "enum": ["low", "medium", "high"] },
        "confidence":  { "type": "number" }
      },
      "required": ["app", "sensitivity", "confidence"],
      "additionalProperties": false
    }
  },
  "store_as": "$.classification"
}
```

After this step runs, downstream steps can branch on `get('$.classification.app')`, reference `$.classification.sensitivity` in instructions, etc. Use `output_parser: {"type": "default"}` for free-form text steps that don't need to capture structured output.

### Strict-mode rules

When `strict: true`, the platform enforces the OpenAI-structured-outputs subset of JSON Schema. Schemas that violate these will be rejected at agent creation or silently fall back to non-strict mode:

- **`additionalProperties: false`** on every `object` type
- **Every property listed in `required`** (no optional fields — make a field nullable via `"type": ["string", "null"]` instead)
- **No** `minLength`, `maxLength`, `pattern`, `min`, `max`, or `items` keywords

If you need optional fields or string-length validation, drop `strict: true` and validate downstream — but the model also has more freedom to produce shapes you didn't expect.

## Runtime-populated enum (constrained generation from a tool result)

A common need: the LLM must pick from a list that's only known at runtime — role names from a permissions API, ticket categories from an ITSM, etc. Vectara's pattern is to write `enum` into the JSON schema but populate it dynamically from the prior tool call's stored result:

```json
"output_parser": {
  "type": "structured",
  "json_schema": {
    "name": "role_pick",
    "strict": true,
    "schema": {
      "type": "object",
      "properties": {
        "role_binding": {
          "type": "string",
          "description": "Must be one of $.candidates. Enum is populated at runtime from the wolken_get_access_types result."
        }
      },
      "required": ["role_binding"],
      "additionalProperties": false
    }
  }
}
```

The step's tool fetches the candidates (`wolken_get_access_types`), the instruction tells the LLM "store the result as `$.candidates`," and the structured output enforces the pick against that slot. Prevents hallucinated role names that look valid but don't exist.

## Step-scoped tools via `allowed_tools`

All tools are declared once at the top-level `tool_configurations`. Each step can whitelist a subset via `allowed_tools`. Without `allowed_tools`, the step has access to every tool:

```json
"steps": {
  "manager_gate": {
    "allowed_tools": ["post_approval_request"],
    "instructions": [...],
    "reminders": [
      { "type": "inline", "template": "You are at the GATE. Provisioning tools are not attached here." }
    ]
  }
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

## Lazy-loaded instructions via `skills`

Define `{description, content}` pairs the agent can load on demand instead of stuffing every runbook into the system prompt.

```json
"skills": {
  "customer_escalation": {
    "description": "Use when an inbound customer message reports an outage, a regression, data loss, or visible anger about an urgent issue. Loads the severity rubric, routing rules, and outbound message templates.",
    "content": "## 1. Severity classification\nPick exactly ONE level using this rubric:\n- SEV-1 — production down OR data loss OR security/PII exposure...\n## 2. Required information to collect\n..."
  }
}
```

- **`description` ≤ 500 chars.** Shown in the system prompt every turn — the model reads it to decide whether to invoke. Keep it diagnostic ("use when …").
- **`content` ≤ 50,000 chars.** The heavy runbook. Loaded into context only when the skill fires. This is the whole point — large specialist guidance you don't pay tokens for on every turn.

### How invocation works

When an agent has skills configured, Vectara automatically exposes an `invoke_skill` tool to the model. There are two ways content gets loaded:

1. **Model-driven.** The LLM emits a `tool_input` for `invoke_skill` with the skill's `skill_name`. Vectara injects the skill's `content` as a user message and emits a `skill_load` event.
2. **Client-driven.** Your wrapper POSTs an event with `messages: [{"type": "skill", "skill_name": "customer_escalation"}]`. No model round-trip needed for the decision — Vectara loads the content immediately. Useful when your UI already knows which mode to enter (e.g. a routing rule that always loads the outage runbook for SEV-1 alerts).

Either way the loaded content stays in session history for follow-up turns (until compaction).

### Per-step restriction via `allowed_skills`

Each step independently controls which skills the `invoke_skill` tool exposes:

| Form | Semantics |
|---|---|
| omitted (default) | All skills declared on the agent are available |
| `[]` (empty array) | No skills; the `invoke_skill` tool isn't exposed at all in this step |
| `["skill_a", "skill_b"]` | Only those skills are loadable |

Pairs cleanly with the multi-step pattern: a *router* step might set `allowed_skills: []` (force routing-only behavior), then transition into a *specialist* step that exposes exactly one skill.

```json
{
  "name": "router",
  "instructions": [{"type": "inline", "template": "Route the message; do not draft a reply."}],
  "output_parser": {"type": "default"},
  "allowed_skills": []
}
```

### Skill vs. tool vs. prompt-stuffing

A skill is *instructions* loaded on demand. If the agent needs to *do* something (fetch data, call an API, query a corpus), use a tool. If the guidance applies to every turn and is short, just put it in the system prompt. Skills shine when you have multiple distinct mindsets/runbooks/voices that would balloon the system prompt if always-loaded.

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
