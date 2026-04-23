# Vectara Agent Skills

Agent skills that teach coding agents how to build with the [Vectara](https://vectara.com) platform correctly.

## What are Agent Skills?

Agent skills are instruction files that help AI coding agents (Claude Code, Cursor, Copilot, etc.) understand how to use the Vectara platform. Instead of relying on potentially outdated training data, the agent gets current, accurate patterns.

## Available Skills

| Skill | Description |
|-------|-------------|
| [agents](agents/SKILL.md) | Build AI agents with tools, sessions, and instructions using the v2 API |

## Quick Start

### Claude Code

```bash
curl -o .claude/skills/vectara-agents.md https://raw.githubusercontent.com/vectara/agent-skills/main/agents/SKILL.md
```

### Cursor

```bash
curl -o .cursor/rules/vectara-agents.md https://raw.githubusercontent.com/vectara/agent-skills/main/agents/SKILL.md
```

### Manual

Copy the contents of the skill's `SKILL.md` into your AI coding agent's instructions or rules directory.

## Documentation

For full Vectara documentation:
- [docs.vectara.com](https://docs.vectara.com)
- [LLM-friendly docs](https://docs.vectara.com/llms.txt)

## License

MIT
