# Vectara Agent Skills

Agent skills that teach coding agents how to build with the [Vectara](https://vectara.com) platform correctly.

## What are Agent Skills?

Agent skills are instruction files that help AI coding agents (Claude Code, Cursor, Codex, etc.) understand how to use the Vectara platform. Instead of relying on potentially outdated training data, the agent gets current, accurate patterns.

## Available Skills

| Skill | Description |
|-------|-------------|
| [agents](skills/agents/SKILL.md) | Build AI agents with tools, sessions, and instructions |
| [agent-auth-and-secrets](skills/agent-auth-and-secrets/SKILL.md) | Wire OAuth, API-key, and service-account credentials into agent tools via `$ref`, `agent.secrets`, and `session.metadata` |
| [agent-orchestration](skills/agent-orchestration/SKILL.md) | Multi-step state-machine agents with `first_step`/`steps[]`/`next_steps`/`allowed_tools`, sub-agents, structured outputs, connectors |
| [corpus-and-ingestion](skills/corpus-and-ingestion/SKILL.md) | Create corpora, declare filter attributes, ingest structured / core / file-upload documents, write metadata filter expressions, run bulk-delete jobs |
| [search-and-retrieval](skills/search-and-retrieval/SKILL.md) | Query corpora (single, multi, GET); hybrid (lexical+semantic) blending; metadata filters; rerankers (chain, knee, MMR, multilingual, UDF); citations |
| [generation-and-prompts](skills/generation-and-prompts/SKILL.md) | Generation presets, Apache Velocity prompt templates, custom prompts with metadata, response languages, LLM selection, bring-your-own LLM |
| [chat](skills/chat/SKILL.md) | Vectara Chat — persistent chats and turns, the OpenAI-compatible `/chat/completions` endpoint, and custom LLM registration |
| [pipelines](skills/pipelines/SKILL.md) | Automated ingestion flows from a source (S3, etc.) into agent sessions — watermarks, dead-letter queue, runs, verification |
| [hallucination-and-eval](skills/hallucination-and-eval/SKILL.md) | HHEM factual-consistency scoring, Vectara Hallucination Corrector, in-console query observability, Open RAG Eval offline framework |
| [python-sdk](skills/python-sdk/SKILL.md) | The official `vectara` Python SDK — install, client init, modules, error model, end-to-end examples |

## Install

### One-liner (all platforms)

```bash
npx skills add vectara/agent-skills
```

### Manual install

#### Claude Code

Copy skill files into `.claude/skills/` in your project:

```bash
mkdir -p .claude/skills
curl -o .claude/skills/vectara-agents.md \
  https://raw.githubusercontent.com/vectara/agent-skills/main/skills/agents/SKILL.md
```

#### Cursor

Copy skill files into `.cursor/rules/` in your project:

```bash
mkdir -p .cursor/rules
curl -o .cursor/rules/vectara-agents.md \
  https://raw.githubusercontent.com/vectara/agent-skills/main/skills/agents/SKILL.md
```

#### Codex

Copy skill files into `.agents/skills/` in your project:

```bash
mkdir -p .agents/skills
curl -o .agents/skills/vectara-agents.md \
  https://raw.githubusercontent.com/vectara/agent-skills/main/skills/agents/SKILL.md
```

## Documentation

- [docs.vectara.com](https://docs.vectara.com)
- [LLM-friendly docs](https://docs.vectara.com/llms.txt)

## License

MIT
