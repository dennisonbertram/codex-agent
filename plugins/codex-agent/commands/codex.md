---
name: codex
description: |
  Delegate a task to OpenAI Codex CLI for autonomous execution.

  Codex runs in full-auto ephemeral mode, executes the task, and reports
  back with results and any file changes.

  Supports optional flags:
    --reasoning=low|medium|high   Set reasoning effort (default: auto-detected)
    --read-only                   Run in read-only mode (no file writes)
    --json                        Request JSON-formatted output from Codex

  Examples:
    /codex add JSDoc to all functions in lib/utils.ts
    /codex --reasoning=high analyze the auth flow in middleware.ts
    /codex --read-only count API routes under app/api/
    /codex --json list all exported types in src/types/
    /codex refactor the database connection pool to use singleton pattern
argument-hint: "<task description> [--reasoning=low|medium|high] [--read-only] [--json]"
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - Write
---

# Codex Agent Command

You are executing a task delegation to OpenAI Codex CLI. Follow these steps precisely.

## Step 1: Validate Environment

Run these checks before proceeding:

```bash
which codex 2>/dev/null || echo "CODEX_NOT_FOUND"
```

If `codex` is not found, tell the user:
- Codex CLI is not installed or not on PATH.
- They can install it with: `npm install -g @openai/codex`
- Then retry the command.
- **Stop here.** Do not proceed further.

Check for the OpenAI API key:

```bash
echo "${OPENAI_API_KEY:+SET}" || echo "NOT_SET"
```

If the key is not set, tell the user:
- The `OPENAI_API_KEY` environment variable is not set.
- They need to export it or add it to their shell profile.
- **Stop here.** Do not proceed further.

## Step 2: Parse Arguments

The raw arguments string is: `$ARGUMENTS`

Parse the following from the arguments:

1. **Flags** - Extract any flags present:
   - `--reasoning=low|medium|high` - Explicit reasoning level override.
   - `--read-only` - If present, run Codex in read-only mode (add `--read-only` flag).
   - `--json` - If present, request JSON output (add `--json` flag).

2. **Task description** - Everything remaining after flags are removed. This is the actual task to pass to Codex.

If the task description is empty after parsing, ask the user to provide a task description and **stop here.**

## Step 3: Determine Reasoning Level

If `--reasoning` was explicitly provided, use that value.

Otherwise, auto-detect based on the task description:

- **low** - Simple, mechanical tasks: adding comments, renaming variables, formatting, counting files, listing items, simple find-and-replace operations.
- **medium** - Moderate tasks: refactoring a single function, adding error handling, writing tests for existing code, implementing a straightforward feature.
- **high** - Complex tasks: architectural analysis, multi-file refactoring, security audits, designing new systems, debugging subtle issues, performance optimization.

Use your judgment. When in doubt, use **medium**.

## Step 4: Record Pre-Execution State

Before running Codex, capture the current git state so you can show what changed:

```bash
git diff --stat HEAD 2>/dev/null
git status --porcelain 2>/dev/null
```

Save this output mentally as the "before" state.

## Step 5: Execute Codex

Build and run the Codex command. The base command is:

```
codex exec --full-auto --ephemeral
```

Add flags based on parsing:

- Always add: `--reasoning <level>` using the determined reasoning level.
- If `--read-only` was specified: add `--read-only`.
- If `--json` was specified: add `--json`.
- The task description goes at the end as a quoted string argument.

The full command pattern:

```bash
codex exec --full-auto --ephemeral --reasoning <LEVEL> [--read-only] [--json] "<TASK_DESCRIPTION>"
```

Run this with a **5-minute timeout** (300000ms). Codex tasks can take time.

**Important:** Capture both stdout and stderr. If the command fails, capture the exit code.

## Step 6: Handle Errors

If Codex returns a non-zero exit code or produces an error:

- Show the user the error output.
- Provide guidance based on common errors:
  - "command not found" -> Codex is not installed.
  - "API key" or "authentication" -> API key issue.
  - "rate limit" -> Suggest waiting and retrying.
  - "timeout" -> Task may be too large; suggest breaking it down.
- **Stop here** unless partial output was produced, in which case show it.

## Step 7: Present Results

Show the user the Codex output clearly. Format it as follows:

### Codex Output
Present the stdout from Codex. If it is very long (more than 100 lines), summarize the key points and offer to show the full output.

### Files Changed
Run `git diff --stat HEAD` and `git status --porcelain` again. Compare with the "before" state from Step 4 to identify what Codex actually changed.

If files were changed:
- Show the `git diff --stat` summary.
- For each changed file, run `git diff <file>` and show a concise summary of what changed.
- If `--read-only` was used, confirm that no files were modified.

If no files were changed:
- State that no files were modified (expected for read-only or analysis tasks).

## Step 8: Offer Verification

If files were changed (and the task was not read-only), offer to run verification:

> Files were changed by Codex. Would you like me to run lint and build to verify everything is clean?

Do **not** run lint/build automatically. Wait for the user to confirm before running any verification commands.

If the user confirms, run:

```bash
npm run lint 2>&1
npm run build 2>&1
```

Report the results. If either fails, show the errors and offer to help fix them.

## Important Notes

- Always use `--ephemeral` to avoid Codex persisting state between runs.
- Always use `--full-auto` since the user has explicitly requested delegation.
- Never modify files yourself during this command -- Codex does the work.
- If Codex output contains sensitive information (API keys, secrets), warn the user and do not display them in full.
- The working directory for Codex should be the current working directory.
