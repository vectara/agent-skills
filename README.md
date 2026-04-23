# Vectara Agent Skills

Agent skills that teach coding agents how to build with the [Vectara](https://vectara.com) platform correctly.

## What are Agent Skills?

Agent skills are instruction files that help AI coding agents (Claude Code, Cursor, Copilot, etc.) understand how to use the Vectara platform. Instead of relying on potentially outdated training data, the agent gets current, accurate patterns for building Vectara agents.

## Quick Start

### Claude Code

```bash
# Copy the skill to your project
curl -o .claude/skills/vectara.md https://raw.githubusercontent.com/vectara/agent-skills/main/SKILL.md
```

### Cursor

```bash
curl -o .cursor/rules/vectara.md https://raw.githubusercontent.com/vectara/agent-skills/main/SKILL.md
```

### Manual

Copy the contents of `SKILL.md` into your AI coding agent's instructions or rules directory.

## What's Included

The skill teaches agents to:

- Use the **v2 API** (never the deprecated v1)
- Create agents with proper `tool_configurations`, `first_step`, and `model` settings
- Configure corpus search, web search, and other built-in tools
- Manage sessions and send messages correctly
- Avoid common mistakes (wrong auth, missing fields, deprecated endpoints)

## Documentation

For full Vectara documentation:
- [docs.vectara.com](https://docs.vectara.com)
- [LLM-friendly docs](https://docs.vectara.com/llms.txt)

## License

MIT
