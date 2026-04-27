# Vectara Agent Skills

Agent skills that teach coding agents how to build with the [Vectara](https://vectara.com) platform correctly.

## What are Agent Skills?

Agent skills are instruction files that help AI coding agents (Claude Code, Cursor, Codex, etc.) understand how to use the Vectara platform. Instead of relying on potentially outdated training data, the agent gets current, accurate patterns.

## Available Skills

| Skill | Description |
|-------|-------------|
| [agents](skills/agents/SKILL.md) | Build AI agents with tools, sessions, and instructions using the v2 API |

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
