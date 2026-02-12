---
name: codex-agent
description: |
  Delegates scoped coding tasks to OpenAI Codex CLI running non-interactively as a fast build agent. Codex uses gpt-5.3-codex-spark at ~1k tokens/sec, making it ideal for mechanical edits, boilerplate generation, read-only analysis, and parallelizable work. Trigger phrases: "use codex", "codex agent", "run with codex", "delegate to codex", "codex build", "send to codex".
---

# Codex Build Agent

Delegate scoped, mechanical coding tasks to OpenAI's Codex CLI (`codex exec`) running non-interactively. Codex uses `gpt-5.3-codex-spark` at ~1,000 tokens/sec output, making it the fastest option for well-constrained work.

## Prerequisites

- **Codex CLI**: Install globally via `npm install -g @openai/codex`
- **API key**: `OPENAI_API_KEY` must be set in the shell environment

Verify availability before first use:

```bash
command -v codex >/dev/null && echo "codex available" || echo "codex not found — run: npm install -g @openai/codex"
```

## When to Use

**Good fits:**
- Single-file or scoped multi-file edits with clear constraints
- Mechanical refactoring (rename, add types, update imports, toggle directives)
- Adding comments, JSDoc, or inline documentation
- Generating boilerplate, test stubs, or repetitive code
- Read-only code analysis, audits, and exploration
- Tasks that decompose into independent parallel invocations

**Poor fits:**
- Complex architectural changes requiring deep cross-file context
- Tasks needing interactive clarification or approval gates
- Large multi-file refactors with subtle dependencies
- Anything requiring human review mid-execution

## Core Commands

All commands use `codex exec` (non-interactive subcommand). Always include `--full-auto` (no approval prompts, workspace-write sandbox) and `--ephemeral` (no persisted session files).

### Write Task

```bash
codex exec --full-auto --ephemeral -C "$PROJECT_DIR" <<'PROMPT'
Your task description here. Be specific about:
- Which file(s) to modify
- What changes to make
- What NOT to change
PROMPT
```

### Read-Only Analysis

```bash
codex exec --full-auto --ephemeral -s read-only -C "$PROJECT_DIR" <<'PROMPT'
Your analysis question here.
PROMPT
```

### JSON Output (for programmatic parsing)

```bash
codex exec --full-auto --ephemeral --json -C "$PROJECT_DIR" <<'PROMPT'
Your task here.
PROMPT
```

### File Output (save final message)

```bash
codex exec --full-auto --ephemeral -o output.md -C "$PROJECT_DIR" <<'PROMPT'
Your task here.
PROMPT
```

## Flag Reference

| Flag | Purpose |
|------|---------|
| `exec` | Non-interactive subcommand (required for all invocations) |
| `--full-auto` | No approval prompts; enables workspace-write sandbox |
| `--ephemeral` | Do not persist session files |
| `-s read-only` | Read-only sandbox (prevents all file writes) |
| `-s workspace-write` | Write to workspace only (default with `--full-auto`) |
| `-C <dir>` | Set working directory for the task |
| `--json` | Emit JSONL events to stdout for programmatic parsing |
| `-o <file>` | Write the agent's final message to a file |
| `-m <model>` | Override model (default: `gpt-5.3-codex-spark`) |
| `-c model_reasoning_effort=<level>` | Set reasoning effort: `low`, `medium`, or `high` |
| `--add-dir <dir>` | Grant write access to additional directories outside workspace |

## Reasoning Effort Levels

Control how much the model reasons before acting. Pass via `-c model_reasoning_effort="<level>"`.

| Level | Flag | Use Case |
|-------|------|----------|
| `low` | `-c model_reasoning_effort="low"` | Mechanical edits, boilerplate, simple refactors. Fastest. This is the default. |
| `medium` | `-c model_reasoning_effort="medium"` | Multi-step edits, moderate logic changes, small refactors with dependencies |
| `high` | `-c model_reasoning_effort="high"` | Complex analysis, architectural review, bug investigation, security audits |

### Examples by Level

```bash
# Low (default) — fast mechanical work
codex exec --full-auto --ephemeral -C "$DIR" <<'PROMPT'
Add "use client" directive to the top of components/dashboard.tsx
PROMPT

# Medium — needs some reasoning
codex exec --full-auto --ephemeral -c model_reasoning_effort="medium" -C "$DIR" <<'PROMPT'
Refactor lib/api-client.ts to use a generic request() method that all CRUD
methods delegate to. Keep the same public API surface.
PROMPT

# High — deep analysis
codex exec --full-auto --ephemeral -c model_reasoning_effort="high" -s read-only -C "$DIR" <<'PROMPT'
Analyze the authentication flow across middleware.ts, lib/auth.ts, and
app/api/protected/route.ts. Identify any paths where unauthenticated
requests could reach protected data.
PROMPT
```

## Prompt Engineering Guidelines

Codex performs best with constrained, specific prompts. Follow these rules:

1. **Name exact files** to read or modify. Never say "find the relevant files."
2. **State constraints explicitly**: "Only modify X", "Do not change Y", "Preserve Z."
3. **One logical task per invocation**. Do not bundle unrelated changes.
4. **Include project context** when the task depends on conventions (naming, patterns, framework).
5. **Specify output format** for analysis tasks ("List as bullet points", "Return as JSON").

### Good prompt:

```
Update `lib/workflow/executor.ts` only:
- Add a case for "video-gen" in the parallel dispatch switch (~line 2787)
- Add a matching case in the sequential dispatch switch (~line 3732)
- Follow the same pattern as "image-gen" for both cases
Do NOT modify any other switch statements. Keep the `never` exhaustiveness check.
```

### Bad prompt:

```
Add video generation support to the executor.
```

## Parallel Execution

Run multiple independent codex agents simultaneously using background processes:

```bash
codex exec --full-auto --ephemeral -C "$DIR" -o /tmp/task1.md <<'P1' &
Add JSDoc comments to all exported functions in lib/utils.ts
P1

codex exec --full-auto --ephemeral -C "$DIR" -o /tmp/task2.md <<'P2' &
Add credentials: "include" to all fetch calls in lib/api-client.ts
P2

codex exec --full-auto --ephemeral -C "$DIR" -o /tmp/task3.md <<'P3' &
Remove all console.log statements from components/dashboard.tsx
P3

wait
echo "All tasks complete"
cat /tmp/task1.md /tmp/task2.md /tmp/task3.md
```

**Critical rule:** Only parallelize tasks that touch **different files**. Concurrent writes to the same file cause merge conflicts and data loss.

## Parsing JSON Output

When `--json` is used, codex emits newline-delimited JSON events to stdout. Parse them to extract results programmatically:

```bash
codex exec --full-auto --ephemeral --json -s read-only -C "$DIR" <<'PROMPT' 2>/dev/null | \
  node -e "
    const lines = require('fs').readFileSync('/dev/stdin','utf8').trim().split('\n');
    for (const line of lines) {
      const obj = JSON.parse(line);
      if (obj.type === 'turn.completed') {
        console.log('Tokens:', obj.usage.input_tokens + obj.usage.output_tokens);
      }
      if (obj.item?.text) console.log(obj.item.text);
    }
  "
Your analysis question here.
PROMPT
```

### JSON Event Types

| Event | Meaning |
|-------|---------|
| `thread.started` | Session initialized |
| `turn.started` | Agent turn began |
| `item.completed` (agent_message) | Agent text response available |
| `item.started` (command_execution) | Shell command about to execute |
| `item.completed` (command_execution) | Shell command finished |
| `item.completed` (file_update) | File was written or modified |
| `turn.completed` | Turn finished; includes `usage` object with token counts |

## Integration with Claude Code

### Capture output for further processing

```bash
RESULT=$(codex exec --full-auto --ephemeral -o /dev/stdout -C "$PROJECT_DIR" <<'PROMPT' 2>/dev/null
Your task here.
PROMPT
)
echo "$RESULT"
```

### Verify codex changes after execution

Always inspect and validate what codex produced:

```bash
# Review the diff
git diff

# Run lint and build to catch errors
pnpm run lint && pnpm run build
```

### Pattern: delegate then verify

1. Invoke codex with a scoped task via the Bash tool.
2. Read the output or diff to confirm correctness.
3. Fix any issues directly or re-invoke codex with refined constraints.
4. Run the project's lint and build commands before considering the task done.

## Stderr Noise

Codex emits debug logs, rollout path warnings, and MCP startup messages to stderr. Suppress with `2>/dev/null` when only stdout content matters.

## Performance Profile

| Property | Value |
|----------|-------|
| Model | `gpt-5.3-codex-spark` |
| Output speed | ~1,000 tokens/sec |
| Default reasoning | Low (fastest, mechanical) |
| Typical task cost | 5k-25k tokens |
| Parallel scaling | Linear with independent tasks |
