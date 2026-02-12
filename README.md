# codex-agent

A Claude Code plugin that delegates coding tasks to OpenAI's Codex CLI as a fast build agent. Codex runs non-interactively using `gpt-5.3-codex-spark` at ~1,000 tokens/sec with configurable reasoning levels.

## Features

- **Skill** — Full documentation for using Codex as a build agent, loaded automatically when relevant
- **`/codex` command** — Quick delegation: `/codex add JSDoc to lib/utils.ts`
- **Autonomous agent** — Detects mechanical tasks and suggests Codex delegation

## Prerequisites

- [OpenAI Codex CLI](https://github.com/openai/codex) installed: `npm install -g @openai/codex`
- `OPENAI_API_KEY` set in your environment

## Installation

### Via Claude Code CLI

```bash
claude plugin add /path/to/codex-agent
```

### Manual

Clone this repo and point Claude Code at it:

```bash
git clone https://github.com/DennisonBertworram/codex-agent-plugin.git
claude --plugin-dir ./codex-agent-plugin
```

Or symlink into your plugins directory:

```bash
ln -s /path/to/codex-agent ~/.claude/plugins/codex-agent
```

## Usage

### Slash Command

```
/codex add "use client" directive to components/dashboard.tsx
/codex --reasoning=high analyze the auth flow in middleware.ts
/codex --read-only count how many API routes exist
```

### Skill (automatic)

The skill loads automatically when you mention "codex", "delegate to codex", "use codex agent", or discuss parallel task execution. It provides Claude with full knowledge of Codex CLI flags, reasoning levels, and prompt patterns.

### Agent (autonomous)

The codex-delegator agent triggers when you ask to delegate mechanical tasks:
- "use codex to add types to these files"
- "have codex handle the boilerplate generation"
- "delegate the JSDoc additions to codex"

## Reasoning Levels

Control how much Codex "thinks" before acting:

| Level | Best for | Speed |
|-------|----------|-------|
| `low` (default) | Mechanical edits, boilerplate, simple refactors | Fastest |
| `medium` | Multi-step edits, moderate logic changes | Balanced |
| `high` | Complex analysis, architectural review, bug investigation | Thorough |

## Components

```
codex-agent/
├── plugin.json              # Plugin manifest
├── README.md                # This file
├── skills/
│   └── codex-agent/
│       └── skill.md         # Codex build agent knowledge
├── commands/
│   └── codex.md             # /codex slash command
└── agents/
    └── codex-delegator.md   # Autonomous delegation agent
```

## License

MIT
