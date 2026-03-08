# Hooks & SDK

## Introduction to Hooks

Hooks let you run custom code at specific moments in Claude Code's workflow. Every time Claude reads a file, runs a command, starts a session, or finishes a response — a hook can fire.

### What Hooks Solve

Claude Code works in a loop: decide, act, observe, repeat. Hooks let you tap into that loop at every stage:

- **Block dangerous commands** before they run (PreToolUse)
- **Validate code** after Claude edits it (PostToolUse)
- **Log every tool call** for auditing (PostToolUse)
- **Inject context** into Claude's conversation (UserPromptSubmit)
- **Send notifications** when Claude finishes (Stop)

### Hook Events

Claude Code fires hooks at specific points. Here are the ones you'll use most:

| Event | When It Fires | Use Case |
|-------|--------------|----------|
| `PreToolUse` | Before a tool runs | Block dangerous commands, validate inputs |
| `PostToolUse` | After a tool succeeds | Run linters, verify output, log activity |
| `UserPromptSubmit` | You submit a prompt | Modify or validate the prompt |
| `Stop` | Claude finishes responding | Notify, trigger follow-up actions |
| `SessionStart` | New session begins | Load environment, setup |
| `Notification` | Claude needs attention | Play sounds, send alerts |

Other events: `PostToolUseFailure`, `PermissionRequest`, `SubagentStart`, `SubagentStop`, `PreCompact`, `SessionEnd`, `Setup`, `TeammateIdle`, `TaskCompleted`, `ConfigChange`.

### How Hooks Receive Data

When a hook fires, Claude Code passes event data as JSON on stdin:

```json
{
  "hook_event_name": "PreToolUse",
  "session_id": "abc-123",
  "cwd": "/home/user/claude-code-best-practices-starter",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test",
    "description": "Run tests",
    "timeout": 120000
  }
}
```

Your script reads this JSON, decides what to do, and exits.

---

## Defining a Hook

Hooks are defined in `.claude/settings.json` under the `hooks` key. Each entry specifies the event, an optional matcher, and the command to run.

### Anatomy of a Hook Entry

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PROJECT_DIR}/scripts/validate-bash.py",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

| Field | What It Does |
|-------|-------------|
| `type` | `"command"` (shell), `"prompt"` (AI judgment), or `"agent"` (sub-agent) |
| `command` | The script to run. `${CLAUDE_PROJECT_DIR}` resolves to project root |
| `timeout` | Kill the script after this many milliseconds |
| `async` | `true` = run in background, don't block Claude |
| `once` | `true` = fire only once per session |
| `statusMessage` | Text shown in spinner while hook runs |

### Matchers

Matchers filter which events trigger the hook:

- `"Bash"` — exact match, only fires for Bash tool
- `"Edit|Write"` — fires for either Edit or Write
- `"mcp__memory__.*"` — regex, all tools from memory MCP server

Not all events support matchers. `UserPromptSubmit`, `Stop`, and `Setup` always fire.

### Three Hook Types

| Type | What It Does | When To Use |
|------|-------------|-------------|
| `command` | Runs a shell command | Most hooks — logging, validation, sounds |
| `prompt` | Sends prompt to Claude for yes/no judgment | Decisions needing AI reasoning |
| `agent` | Spawns a sub-agent with tool access | Complex verification needing file inspection |

### Practice: Define Your First Hook

Create `.claude/settings.json` in the starter project (if it doesn't exist) and add a simple session logging hook:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Session started at $(date)\" >> /tmp/claude-sessions.log",
            "timeout": 5000,
            "async": true,
            "once": true
          }
        ]
      }
    ]
  }
}
```

Restart Claude and check `/tmp/claude-sessions.log` — you should see the session timestamp.

<!-- Poridhi screenshot: settings.json with hook defined, then checking the log file -->

---

## Implementing a Hook

Let's build a real hook script that inspects what Claude is about to do and logs it.

### The Script

Create `scripts/hook-logger.py` in your starter project:

```python
import json
import sys

def main():
    try:
        # Read JSON from stdin
        stdin_content = sys.stdin.read().strip()
        if not stdin_content:
            sys.exit(0)

        data = json.loads(stdin_content)

        event = data.get("hook_event_name", "")
        tool = data.get("tool_name", "")
        tool_input = data.get("tool_input", {})

        # Log what Claude is doing
        with open("/tmp/claude-hook-log.txt", "a") as f:
            if tool:
                f.write(f"[{event}] {tool}: {json.dumps(tool_input)}\n")
            else:
                f.write(f"[{event}]\n")

        sys.exit(0)

    except Exception:
        sys.exit(0)  # Never block Claude on hook errors
```

The crucial pattern: **always exit 0**. If your hook crashes with a non-zero exit code, it can block Claude. The try/except wrapping everything ensures even unexpected errors don't disrupt your session.

### Wire It Up

Add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PROJECT_DIR}/scripts/hook-logger.py",
            "timeout": 5000,
            "async": true
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PROJECT_DIR}/scripts/hook-logger.py",
            "timeout": 5000,
            "async": true
          }
        ]
      }
    ]
  }
}
```

### Practice: See Your Hook in Action

1. Create the `scripts/` directory and `hook-logger.py` file
2. Update `.claude/settings.json` with the hook entries above
3. Restart Claude and ask a question about the project
4. Check the log: `cat /tmp/claude-hook-log.txt`
5. You'll see every tool Claude used — Read, Glob, Grep — logged with their inputs

<!-- Poridhi screenshot: Hook log showing tool calls -->

---

## Common Gotchas

Hooks have sharp edges. These are the things that bite you.

### Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| `0` | Success — Claude continues |
| `2` | Block — for PreToolUse, prevents the tool from running |
| Non-zero | Error — may block Claude depending on the event |

Always wrap your hook in try/except with `sys.exit(0)`. A crash in your hook script should never stop Claude from working.

### Sync vs Async

By default, hooks are **synchronous** — Claude waits for the hook to finish. If your hook takes 3 seconds, Claude pauses for 3 seconds on every tool call.

Use `"async": true` for anything that doesn't need to block (logging, sounds, notifications). Only keep sync for hooks where you need the result before Claude continues (like pre-commit validation).

### Timeout

The `timeout` field (milliseconds) kills hung processes. Rules of thumb:
- Quick tasks (logging, sounds): `5000`
- Validation scripts: `10000`
- Complex checks: `30000`

### Path Quoting

Always quote `${CLAUDE_PROJECT_DIR}` in shell scripts — paths with spaces will break:

```bash
python3 "$CLAUDE_PROJECT_DIR/scripts/hook-logger.py"
```

### Disabling Hooks

Emergency switch in `.claude/settings.local.json`:

```json
{
  "disableAllHooks": true
}
```

Or use the interactive manager:

```
/hooks
```

---

## Useful Hook Patterns

### Pre-Commit Validation

Block `git commit` unless tests pass:

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "python3 ${CLAUDE_PROJECT_DIR}/scripts/pre-commit-check.py",
          "timeout": 10000
        }
      ]
    }
  ]
}
```

The script checks if `tool_input.command` contains `git commit`, runs tests, and returns exit code 2 to block or 0 to allow. No `async: true` — you want this to block until validation finishes.

```python
import json, sys, subprocess

data = json.loads(sys.stdin.read())
command = data.get("tool_input", {}).get("command", "")

if "git commit" in command:
    result = subprocess.run(["npm", "test"], capture_output=True, cwd=data["cwd"])
    if result.returncode != 0:
        print(json.dumps({
            "hookSpecificOutput": {
                "permissionDecision": "deny",
                "permissionDecisionReason": "Tests are failing. Fix them before committing."
            }
        }))
        sys.exit(0)

sys.exit(0)
```

### Post-Edit Linting

Run your linter after Claude edits a file:

```json
{
  "PostToolUse": [
    {
      "matcher": "Edit|Write",
      "hooks": [
        {
          "type": "command",
          "command": "python3 ${CLAUDE_PROJECT_DIR}/scripts/post-edit-lint.py",
          "timeout": 10000
        }
      ]
    }
  ]
}
```

The script reads the file path from `tool_input`, runs the linter, and returns warnings as `additionalContext` — Claude sees them and can fix issues immediately.

### Notification on Finish

Send a desktop notification when Claude finishes:

```json
{
  "Stop": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "notify-send 'Claude finished'",
          "timeout": 5000,
          "async": true
        }
      ]
    }
  ]
}
```

### Practice: Build a Pre-Commit Hook

1. Create `scripts/pre-commit-check.py` using the pattern above
2. Add the PreToolUse hook to `.claude/settings.json` with `matcher: "Bash"`
3. Ask Claude to commit something — watch it get blocked if tests fail
4. Fix the test, ask again — this time it goes through

<!-- Poridhi screenshot: Claude blocked by pre-commit hook, then succeeding -->

---

## Decision Control

Hooks can do more than observe — they can control Claude's behavior.

### Blocking a Tool Call

A PreToolUse hook can prevent a tool from running:

```json
{
  "hookSpecificOutput": {
    "permissionDecision": "deny",
    "permissionDecisionReason": "Production database access is blocked"
  }
}
```

Claude sees the denial and the reason, and adjusts its approach.

### Auto-Approving a Tool

Automatically approve without prompting:

```json
{
  "hookSpecificOutput": {
    "permissionDecision": "allow",
    "autoAllow": true
  }
}
```

`autoAllow: true` means future uses of this tool won't prompt either.

### Injecting Context

Any hook can add information to Claude's conversation:

```json
{
  "additionalContext": "CI status: build failing on main. Last failure: auth.test.js line 45."
}
```

### Modifying the User's Prompt

`UserPromptSubmit` can rewrite your prompt before Claude sees it:

```json
{
  "hookSpecificOutput": {
    "prompt": "The user asked: 'fix the bug'. CI context: failing test is auth.test.js line 45, AttributeError."
  }
}
```

### Stopping Claude Entirely

```json
{
  "continue": false,
  "stopReason": "Daily usage limit reached"
}
```

### Decision Control Reference

| Hook | Output Field | Values |
|------|-------------|--------|
| PreToolUse | `permissionDecision` | `allow`, `deny`, `ask` |
| PreToolUse | `autoAllow` | `true` to auto-approve future uses |
| UserPromptSubmit | `prompt` | Rewritten prompt text |
| Any hook | `continue` | `false` to stop Claude |
| Any hook | `additionalContext` | Text injected into conversation |
| Any hook | `systemMessage` | Warning shown to user only |

---

## Claude Code SDK

Claude Code can run outside the interactive terminal. Headless mode and the SDK enable CI/CD pipelines, automated workflows, and custom integrations.

### Headless Mode with --print

The `--print` flag (or `-p`) runs Claude non-interactively:

```bash
claude -p "Summarize the changes in the last commit"
```

One prompt in, one response out. No interactive session.

### Output Formats

```bash
# Plain text (default)
claude -p "Explain this function" --output-format text

# JSON — structured, includes metadata
claude -p "List the files" --output-format json

# Streaming JSON — events as they happen
claude -p "Refactor this" --output-format stream-json
```

### Permission Handling for Automation

In headless mode, permission prompts would block forever. Pre-approve tools:

```bash
claude -p "Fix linting errors" \
  --allowedTools "Read,Edit,Bash(npm run lint)"
```

Or use a broader permission mode:

```bash
claude -p "Run the test suite" \
  --permission-mode acceptEdits
```

### CI/CD Patterns

**Automated code review:**
```bash
gh pr diff $PR_NUMBER | claude -p "Review this diff for bugs" \
  --output-format json --model haiku
```

**Changelog generation:**
```bash
git log --oneline v1.0..HEAD | claude -p "Generate a changelog" \
  --output-format text > CHANGELOG.md
```

**Pre-merge validation:**
```bash
claude -p "Check if tests cover the new auth module" \
  --allowedTools "Read,Glob,Grep" --output-format json
```

### Piping Input

```bash
cat error.log | claude -p "Analyze this error and identify the root cause"
git diff HEAD~1 | claude -p "Summarize these changes"
```

### The Node.js SDK

For deeper integration:

```javascript
import { query } from "@anthropic-ai/claude-code";

const messages = await query({
  prompt: "What files are in this project?",
  options: {
    model: "sonnet",
    maxTurns: 3,
    allowedTools: ["Read", "Glob", "Grep"]
  }
});
```

**When to use `--print` vs the SDK:**
- `--print` for CI/CD scripts, one-shot tasks, shell piping
- SDK for web interfaces, multi-turn workflows, custom agent systems

### Key Startup Flags

| Flag | What It Does |
|------|-------------|
| `--print` / `-p` | Non-interactive mode |
| `--output-format` | text, json, stream-json |
| `--model` | Set model tier |
| `--allowedTools` | Pre-approve specific tools |
| `--permission-mode` | acceptEdits, bypassPermissions |
| `--max-turns` | Limit agentic loop iterations |

### Practice: Try Headless Mode

1. Run: `claude -p "What is this project about?"`
2. Run: `claude -p "List all JavaScript files" --output-format json`
3. Try piping: `git log --oneline -5 | claude -p "Summarize these commits"`

<!-- Poridhi screenshot: Headless mode output -->

---

## Key Takeaways

- Hooks are shell commands triggered at specific points in Claude's workflow, receiving event data as JSON on stdin
- Always `sys.exit(0)` in catch blocks — unhandled errors can block Claude
- Use `async: true` for side-effects; sync only for blocking validation
- PreToolUse hooks can deny tools, auto-approve tools, or inject context
- `--print` mode enables CI/CD automation; the Node.js SDK enables custom tooling
- Match `--model` to task: haiku for quick CI checks, sonnet for deeper analysis
