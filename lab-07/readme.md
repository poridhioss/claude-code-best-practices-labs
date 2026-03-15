# Strategic Compacting and System Prompt Injection

Context compaction is Claude summarizing its own conversation to free up tokens. By default, Claude Code auto compacts when the context window fills up. The problem is that auto compact happens at arbitrary points — often mid task, mid thought, mid implementation. Claude loses track of details it was just working with because the summary compressed them away.

Strategic compacting means you control when compaction happens. You compact at logical phase transitions — after exploration is done and before implementation starts, after a milestone is complete and before the next begins. The summary captures a clean state instead of a messy mid task snapshot.

This lab also covers system prompt injection: loading different context profiles depending on what kind of work you are doing. Development mode, review mode, research mode — each with its own set of priorities and instructions loaded at the system prompt level.


## Why Auto Compact Hurts

Auto compact triggers based on token count, not task state. Here is what typically happens:

1. You explore the codebase — Claude reads 15 files, greps for patterns, builds a mental model
2. Context fills up mid exploration
3. Auto compact fires — Claude summarizes everything
4. The summary is generic: "explored codebase, found patterns in auth module"
5. You start implementing — Claude has lost the specific file contents, line numbers, and edge cases it just discovered

The summary preserved the gist but lost the details. Now Claude needs to re read files it already read. You burn tokens twice for the same information.

### Manual Compact at Phase Transitions

The better pattern:

1. **Explore** — read files, grep, understand the codebase
2. **Compact** — summarize exploration findings at a clean breakpoint
3. **Plan** — design the implementation from the summary
4. **Compact** — summarize the plan
5. **Implement** — code from the plan summary
6. **Compact** — summarize implementation for review phase

Each compact captures the output of a complete phase. The summary is meaningful because it summarizes a finished thought, not an interrupted one.

```
/compact
```

> Disable auto compact in your settings and compact manually. You will use more context per phase, but each phase will be higher quality because Claude retains full detail while actively working.


## Building a Compact Suggester Hook

You should not have to remember to compact. A hook can watch your tool call count and suggest compaction at logical intervals. It does not force compaction — it nudges you when context has accumulated enough that compacting would help.

```bash
#!/bin/bash
# Strategic Compact Suggester
# Runs on PreToolUse to suggest manual compaction at logical intervals
#
# Why manual over auto compact:
# - Auto compact happens at arbitrary points, often mid task
# - Strategic compacting preserves context through logical phases
# - Compact after exploration, before execution
# - Compact after completing a milestone, before starting next

COUNTER_FILE="/tmp/claude-tool-count-$$"
THRESHOLD=${COMPACT_THRESHOLD:-50}

# Initialize or increment counter
if [ -f "$COUNTER_FILE" ]; then
  count=$(cat "$COUNTER_FILE")
  count=$((count + 1))
  echo "$count" > "$COUNTER_FILE"
else
  echo "1" > "$COUNTER_FILE"
  count=1
fi

# Suggest compact after threshold tool calls
if [ "$count" -eq "$THRESHOLD" ]; then
  echo "[StrategicCompact] $THRESHOLD tool calls reached - consider /compact if transitioning phases" >&2
fi
```

Hook it to `PreToolUse` on Edit and Write operations:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "tool == \"Edit\" || tool == \"Write\"",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/strategic-compact.sh"
      }]
    }]
  }
}
```

The threshold is configurable via the `COMPACT_THRESHOLD` environment variable. Start with 50 tool calls and adjust based on your workflow. Some sessions are read heavy (high tool count but low context pressure), others are edit heavy (fewer tools but more context per call).

<!-- screenshot: compact suggester hook firing after 50 tool calls -->


### Practice: Disable Auto Compact and Work Manually

Disable auto compact in your Claude Code settings. Start a session and explore a codebase without compacting:

```
Explore the project structure and identify all API endpoints, their handlers, and the database queries they use.
```

Let Claude read extensively. When it finishes exploration, run:

```
/compact
```

<!-- screenshot: manual /compact after exploration phase -->

Now ask Claude to plan the implementation:

```
Based on what you found, plan how to add rate limiting to all public API endpoints.
```

Notice how the compact preserved the exploration summary cleanly. Claude knows which endpoints exist and where, even though the individual file reads were compressed away.


## Dynamic System Prompt Injection

Claude Code loads `CLAUDE.md` and `.claude/rules/` at startup. These are static — the same rules load every session regardless of what you are doing. Sometimes you want different context depending on the task. A development session needs implementation patterns. A review session needs security checklists. A research session needs exploration guidelines.

System prompt injection lets you load context dynamically through CLI flags.

```bash
claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"
```

### Why System Prompt vs @ File References

When you use `@memory.md` or put something in `.claude/rules/`, Claude reads it via the Read tool during the conversation — it comes in as tool output. When you use `--system-prompt`, the content gets injected into the actual system prompt before the conversation starts.

The difference is instruction hierarchy:

| Level | Authority | Example |
|-------|-----------|---------|
| System prompt | Highest | `--system-prompt` content |
| User messages | Medium | Your typed prompts |
| Tool results | Lower | `@file` references, Read tool output |

For most day to day work this distinction is marginal. But for things like strict behavioral rules, project specific constraints, or context you absolutely need Claude to prioritize, system prompt injection ensures it is weighted appropriately.

### Setting Up Context Aliases

Create context files for different modes of work:

```bash
mkdir -p ~/.claude/contexts
```

**`~/.claude/contexts/dev.md`** — focused on implementation:

```markdown
# Development Mode

- Focus on implementation, not exploration
- Write tests before code (TDD)
- Keep changes minimal and focused
- Run tests after every edit
- Do not refactor surrounding code unless asked
```

**`~/.claude/contexts/review.md`** — focused on quality and security:

```markdown
# Review Mode

- Check for security vulnerabilities first
- Look for hardcoded secrets or credentials
- Verify input validation at boundaries
- Check error handling in async code
- Look for missing edge cases in tests
- Do not make changes, only report findings
```

**`~/.claude/contexts/research.md`** — focused on exploration before acting:

```markdown
# Research Mode

- Explore thoroughly before drawing conclusions
- Read related files, not just the target
- Look for patterns across the codebase
- Document findings in structured format
- Do not write any code or make any edits
- Summarize what you learned at the end
```

### Creating Shell Aliases

Turn these into one command aliases:

```bash
# Add to ~/.bashrc or ~/.zshrc
alias claude-dev='claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"'
alias claude-review='claude --system-prompt "$(cat ~/.claude/contexts/review.md)"'
alias claude-research='claude --system-prompt "$(cat ~/.claude/contexts/research.md)"'
```

Now starting a review session is:

```bash
claude-review
```

Claude loads with review priorities baked into the system prompt. It will naturally check for security issues before anything else because the system prompt told it to.

<!-- screenshot: claude-review session with review mode system prompt loaded -->


### Practice: Compare System Prompt Modes

Start Claude in research mode:

```bash
claude --system-prompt "$(cat ~/.claude/contexts/research.md)"
```

Ask:

```
Look at the authentication module and tell me how it works.
```

Claude will explore thoroughly, read multiple files, and give you a structured summary without touching any code.

<!-- screenshot: research mode — Claude exploring and summarizing without editing -->

Now start a new session in dev mode:

```bash
claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"
```

Ask the same kind of question:

```
Add input validation to the user registration endpoint.
```

Claude will jump straight to implementation — write tests first, then code, then verify. Different mode, different behavior, same Claude.

<!-- screenshot: dev mode — Claude implementing with TDD approach -->

> For most workflows the difference between `.claude/rules/` and `--system-prompt` is marginal. The CLI approach is faster (no tool call to read), has higher authority (system level), and is slightly more token efficient. But it is a minor optimization. Use it when the behavioral difference matters — like enforcing strict review mode — and stick with rules files for everything else.


## Combining Both Patterns

The real power is combining strategic compacting with system prompt injection. A typical multi phase workflow:

1. Start in research mode: `claude-research`
2. Explore the codebase, understand the problem
3. `/compact` — summarize research findings
4. End session, start in dev mode: `claude-dev`
5. Load the previous session file: `@~/.claude/sessions/2026-03-15-session.tmp`
6. Implement from the research summary
7. `/compact` — summarize implementation
8. End session, start in review mode: `claude-review`
9. Review the changes with security focus

Each phase gets the right context profile and starts with a clean, relevant summary from the previous phase.


## Key Takeaways

- **Disable auto compact.** Auto compact fires at arbitrary points and loses detail mid task. Manual `/compact` at phase transitions preserves clean, meaningful summaries.

- **Compact at phase transitions**: after exploration, after planning, after implementation. Each compact captures a finished thought, not an interrupted one.

- **The compact suggester hook** nudges you when context has accumulated enough that compacting would help. It does not force compaction — you decide when the phase is actually done.

- **System prompt injection** via `--system-prompt` loads context at the highest authority level. Use it for behavioral modes (dev, review, research) that change how Claude approaches a task.

- **Context aliases** (`claude-dev`, `claude-review`, `claude-research`) make mode switching a single command. Different modes, different priorities, same Claude.

- **Combine both patterns** for multi phase workflows: research mode with compact, then dev mode with compact, then review mode. Each phase gets the right context and starts from a clean summary.
