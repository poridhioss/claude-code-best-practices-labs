# Session Persistence

Claude Code sessions are ephemeral by default. When you close a session, the conversation, the context, the decisions Claude made — all of it disappears. Open a new session tomorrow and Claude has no idea what you worked on yesterday. For a quick bug fix that is fine. For a multi day feature that spans dozens of files and architectural decisions, starting from zero every morning is expensive.

Session persistence is the practice of saving session state to files so Claude can pick up where it left off. Not through some built in memory database, but through hooks that fire at the right lifecycle moments and write structured context to disk. The hooks do the saving automatically. You just start working.


## The Session Lifecycle

Claude Code has three lifecycle moments that matter for persistence:

| Moment | Hook | What Happens |
|--------|------|-------------|
| Session starts | `SessionStart` | Claude begins a new conversation. This is where you load previous context. |
| Context compacts | `PreCompact` | Claude is about to summarize and compress the conversation. Save state before it gets summarized away. |
| Session ends | `Stop` | The conversation is over. Persist everything learned to disk. |

These three hooks form a cycle. The Stop hook from today's session writes a file. Tomorrow's SessionStart hook reads that file. Claude starts informed instead of blank.

```
SESSION 1                              SESSION 2
─────────                              ─────────

[Start]                                [Start]
   │                                      │
   ▼                                      ▼
┌──────────────┐                    ┌──────────────┐
│ SessionStart │                    │ SessionStart │◄── loads previous
│    Hook      │                    │    Hook      │    context
└──────┬───────┘                    └──────┬───────┘
       │                                   │
       ▼                                   ▼
   [Working]                           [Working]
       │                               (informed)
       ▼                                   │
┌──────────────┐                           ▼
│  PreCompact  │──► saves state       [Continue...]
│    Hook      │    before summary
└──────┬───────┘
       │
       ▼
   [Compacted]
       │
       ▼
┌──────────────┐
│  Stop Hook   │──► persists to ──────────►
│ (session end)│    ~/.claude/sessions/
└──────────────┘
```

The PreCompact hook is the safety net. If Claude compacts mid session (which happens when context fills up), you want to save state before the summary loses detail. The Stop hook is the final save when the session closes.


## Session File Structure

Session files live in a dedicated directory. One file per session, named by date and topic, so they are easy to find and prune.

```bash
mkdir -p ~/.claude/sessions
```

The naming convention: `YYYY-MM-DD-topic.tmp`

```
~/.claude/sessions/
├── 2026-03-10-auth-refactor.tmp
├── 2026-03-11-auth-refactor.tmp
├── 2026-03-12-billing-api.tmp
└── 2026-03-14-dashboard-redesign.tmp
```

Each file should contain:

- What approaches worked (verifiably, with evidence)
- Which approaches were attempted and did not work
- Which approaches have not been attempted and what is left to do
- Key decisions made and why
- Current blockers or open questions

This is not a conversation log. It is a structured summary that gives Claude the minimum context needed to resume intelligently. Think of it as a handoff document between two shifts of the same engineer.


## Building the Hooks

You need three shell scripts that fire at the right lifecycle moments.

```bash
mkdir -p ~/.claude/hooks/memory-persistence
```

### session-start.sh

This hook runs when a new session begins. It checks for recent session files and loads the most relevant one.

```bash
#!/bin/bash
# session-start.sh - Load previous session context

SESSION_DIR="$HOME/.claude/sessions"
DAYS_TO_LOOK_BACK=7

if [ ! -d "$SESSION_DIR" ]; then
  exit 0
fi

# Find recent session files
recent_files=$(find "$SESSION_DIR" -name "*.tmp" -mtime -${DAYS_TO_LOOK_BACK} | sort -r)

if [ -n "$recent_files" ]; then
  echo "[SessionStart] Found recent session files:" >&2
  echo "$recent_files" | head -5 | while read f; do
    echo "  - $(basename "$f")" >&2
  done
  echo "[SessionStart] Load with: @$(echo "$recent_files" | head -1)" >&2
fi
```

### pre-compact.sh

This hook fires before context compaction. It saves the current state so nothing critical gets lost in the summary.

```bash
#!/bin/bash
# pre-compact.sh - Save state before compaction

SESSION_DIR="$HOME/.claude/sessions"
TODAY=$(date +%Y-%m-%d)

mkdir -p "$SESSION_DIR"

# Log compaction event
ACTIVE_FILE=$(ls -t "$SESSION_DIR"/${TODAY}-*.tmp 2>/dev/null | head -1)

if [ -n "$ACTIVE_FILE" ]; then
  echo "" >> "$ACTIVE_FILE"
  echo "## Compaction at $(date +%H:%M)" >> "$ACTIVE_FILE"
  echo "Context was compacted. State above reflects pre-compaction context." >> "$ACTIVE_FILE"
  echo "[PreCompact] State saved to $(basename "$ACTIVE_FILE")" >&2
fi
```

### session-end.sh

This hook fires when the session ends. It creates or updates the daily session file with a structured template.

```bash
#!/bin/bash
# session-end.sh - Persist session state

SESSION_DIR="$HOME/.claude/sessions"
TODAY=$(date +%Y-%m-%d)

mkdir -p "$SESSION_DIR"

# Create session file if it does not exist
SESSION_FILE="$SESSION_DIR/${TODAY}-session.tmp"

if [ ! -f "$SESSION_FILE" ]; then
  cat > "$SESSION_FILE" << 'TEMPLATE'
# Session Log

## Started
- Time: TIMESTAMP
- Goal:

## Completed
-

## In Progress
-

## Blocked
-

## Key Decisions
-

## Next Steps
-
TEMPLATE
  sed -i "s/TIMESTAMP/$(date +%H:%M)/" "$SESSION_FILE"
fi

echo "[SessionEnd] Session state saved to $(basename "$SESSION_FILE")" >&2
```

Make them executable:

```bash
chmod +x ~/.claude/hooks/memory-persistence/*.sh
```


## Hook Configuration

Wire the scripts into Claude Code's hook system. Add this to your settings JSON:

```json
{
  "hooks": {
    "PreCompact": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/memory-persistence/pre-compact.sh"
      }]
    }],
    "SessionStart": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/memory-persistence/session-start.sh"
      }]
    }],
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/hooks/memory-persistence/session-end.sh"
      }]
    }]
  }
}
```

The `matcher: "*"` means these hooks fire on every session, regardless of what tools or commands are being used. Session persistence should always be active.


## Resuming a Session

The next day, open Claude and check what the SessionStart hook reports:

```bash
claude
```

You will see output like:

```
[SessionStart] Found recent session files:
  - 2026-03-14-dashboard-redesign.tmp
  - 2026-03-12-billing-api.tmp
```

<!-- screenshot: SessionStart hook showing available session files -->

Load the relevant session file by referencing it:

```
@~/.claude/sessions/2026-03-14-dashboard-redesign.tmp

Continue where we left off on the dashboard redesign.
```

Claude reads the file, sees the completed items, the blockers, the next steps, and picks up with full context of where things stand. No re exploration of the codebase. No re explaining the architecture. Just straight to work.


### Practice: Create and Resume a Session

Set up the hooks and start a session:

```bash
claude
```

Work on something for a few minutes. Then end the session. Check that a session file was created:

```bash
ls ~/.claude/sessions/
```

<!-- screenshot: session file created after ending a session -->

Now start a new session and verify the SessionStart hook reports the file:

```bash
claude
```

Load the file and ask Claude what it knows about the previous session:

```
@~/.claude/sessions/2026-03-15-session.tmp

What was I working on? What are the next steps?
```

<!-- screenshot: Claude resuming from a session file with context from previous work -->

> The session file approach is deliberate. It gives you full control over what persists and what gets pruned. Automated memory systems tend to accumulate noise. With session files, you can review, edit, or delete context before the next session loads it.


## Other Self Improving Memory Patterns

Session files are the foundation, but there are more sophisticated patterns built on the same idea.

### Session Reflection

After each session, a reflection agent extracts what went well, what failed, and what corrections you made. These learnings update a memory file that loads in subsequent sessions. Instead of raw session state, you get distilled preferences — a "diary" of what works for your specific workflow.

### Proactive Suggestions

Rather than waiting for you to notice patterns, the system proactively suggests improvements every 15 minutes. The agent reviews recent interactions, proposes memory updates, and you approve or reject. Over time it learns from your approval patterns and the suggestions get more relevant.

> Both patterns build on the hook infrastructure from this lab. The difference is what gets written to disk: raw state (session files), distilled patterns (reflection), or proposed improvements (proactive suggestions).


## Key Takeaways

- **Session persistence** uses three lifecycle hooks: `SessionStart` (load), `PreCompact` (save before compaction), `Stop` (save on exit). Together they form a cycle that carries context across sessions.

- **Session files** live in `~/.claude/sessions/` with one file per session day. They are structured summaries, not conversation logs. Include what worked, what failed, what is left, and key decisions.

- **PreCompact** is your safety net. Context compaction loses detail. Saving state before compaction ensures nothing critical disappears.

- **Resuming** is explicit. Reference the session file with `@` at the start of your next session. Claude reads it and picks up informed.

- **Prune your session files.** Back up what matters, delete what does not. Without pruning, the directory becomes noise that costs more context to load than it saves.

- **Session files are the foundation** for more sophisticated patterns like session reflection and proactive suggestions. Master the basics before adding complexity.
