# Tmux and Parallel Basics

The previous labs built out your Claude Code configuration: rules, skills, hooks, agents, MCPs. All of that defines how Claude behaves and what it can do. This lab is about how *you* work with Claude effectively. The physical setup: your editor, your terminal, your session management, and the commands that let you navigate and control Claude's behavior during a session.

This is less about configuration and more about muscle memory. The shortcuts, commands, and patterns here are what separate someone who uses Claude Code from someone who is productive with Claude Code.


<!-- ## Terminal and Editor Setup

Claude Code runs in your terminal. It doesn't need an editor. But pairing it with one makes a significant difference because you get real time file tracking as Claude edits, quick navigation to files Claude references, and the ability to review changes before committing.

The setup is straightforward: split your screen. Terminal with Claude Code on one side, editor on the other.

### Zed

Zed is a Rust based editor that's lightweight and fast. It has native Claude integration through its Agent Panel, which tracks file changes in real time as Claude edits.

Key features for Claude Code workflow:
- **Agent Panel**: tracks file changes as Claude edits them
- **`Ctrl+G`**: quickly open the file Claude is currently working on
- **`CMD+Shift+R`**: command palette for all your custom commands and tools
- **Auto save**: enable it so Claude's file reads are always current
- **Vim mode**: full vim keybindings if that's your preference

### VSCode / Cursor

Also works well. You can use it in terminal format with automatic sync using `\ide` for LSP functionality, or use the Claude Code extension which is more integrated with the editor UI.

The important settings regardless of which editor you use:

- **Auto save** enabled so Claude always reads the latest version
- **File watchers** enabled so the editor reloads files Claude changes
- **Git integration** visible so you can review Claude's changes before committing -->

<!-- screenshot: split screen with terminal (Claude Code) on left, editor on right -->


## tmux for Long Running Commands

When Claude runs a dev server, a build process, or a test suite, the output streams directly in your Claude session. This has two problems. First, the output consumes context tokens. Second, if your connection drops or you need to restart Claude, the process dies.

tmux solves both. It's a terminal multiplexer that keeps processes running in detachable sessions. Claude can start a dev server in a tmux session, and you can detach, reattach, or monitor it independently.

### Basic tmux Usage

```bash
# Create a new session named "dev"
tmux new -s dev

# Inside tmux, start your dev server or whatever long process
npm run dev

# Detach from the session (process keeps running)
# Press: Ctrl+B, then D

# Reattach later
tmux attach -t dev

# List all sessions
tmux ls

# Kill a session
tmux kill-session -t dev
```

### tmux with Claude Code

The practical pattern: let Claude spin up servers and background processes in tmux instead of in its own session. This keeps the streaming output out of Claude's context window and lets you monitor logs separately.

```
Start the API server in a tmux session called "api" and the frontend
in a tmux session called "web"
```

Claude will create the tmux sessions and run the servers there. You can attach to either session to watch logs without burning context tokens.

```bash
# Watch API logs
tmux attach -t api

# In another terminal, watch frontend logs
tmux attach -t web
```

> This ties directly into the token optimization from the longform guide. Running background processes outside Claude's context saves input tokens, which is where the majority of cost comes from. For Opus that's $5 per million input tokens.

<!-- screenshot: tmux session with dev server running, separate from Claude Code terminal -->


## Session Management Commands

Claude Code has built in commands for managing your session. These are the ones you'll use daily.

| Command | What It Does |
|---------|-------------|
| `/compact` | Manually trigger context compaction. Summarizes conversation history to free up context. |
| `/clear` | Clears the entire conversation. Fresh start, no history. |
| `/rewind` | Go back to a previous state in the conversation. Undo recent exchanges. |
| `/checkpoints` | File level undo points. Restore files to a previous state. |
| `/fork` | Fork the current conversation into a new branch. |
| `/rename` | Rename the current session for easy identification. |
| `/model` | Switch the model (Haiku, Sonnet, Opus) mid session. |
| `/memory` | Show which config files are loaded. |
| `/mcp` | Show MCP server status. |
| `/plugins` | Manage installed plugins. |
| `/statusline` | Customize the status bar (branch, context %, model, todos). |

### /compact

Context compaction summarizes your conversation history to free up tokens. Claude keeps the important information and discards the verbose details.

When to use it:
- You've finished exploring and are ready to start implementing
- Context is getting full (check the `%` in your statusline)
- You're transitioning between distinct phases of work

```
/compact
```

The longform guide recommends disabling auto compact and compacting manually at logical phase transitions. Auto compact happens at arbitrary points, often mid task. Manual compact lets you control when context gets summarized.

### /fork

Fork creates a branch of your current conversation. Both branches share the same history up to the fork point, then diverge independently.

The practical use: keep your main session for code changes and fork for research questions, documentation lookups, or exploratory tasks. The fork gets its own context, so verbose research doesn't pollute your implementation session.

```
/fork
```

After forking, you'll have two sessions. Use `/rename` to label them:

```
/rename implementation
```

In the forked session:

```
/rename research
```

Now you can switch between them and know which is which.

> From the longform guide: "I prefer the main chat to be working on code changes and the forks for questions about the codebase and its current state, or to do research on external services such as pulling in documentation or searching GitHub."

<!-- screenshot: two Claude sessions, one named "implementation", one named "research" -->


### /rewind and /checkpoints

`/rewind` undoes recent conversation exchanges. If Claude went down a wrong path, rewind instead of trying to correct course with more prompts. Going backward is cheaper than arguing forward.

`/checkpoints` is file level. Claude creates checkpoints of files it edits. If an edit broke something, you can restore the file to a previous checkpoint without undoing the entire conversation.

```
/rewind
```

```
/checkpoints
```

<!-- screenshot: /checkpoints showing available restore points -->


## Keyboard Shortcuts

These save seconds each time, which adds up across a full day of usage.

| Shortcut | Action |
|----------|--------|
| `Ctrl+U` | Delete entire line (faster than holding backspace) |
| `!` | Quick bash command prefix. `! git status` runs immediately |
| `@` | Search for files to reference |
| `/` | Initiate slash commands |
| `Shift+Enter` | Multi line input. Write longer prompts without sending |
| `Tab` | Toggle thinking display (show/hide Claude's reasoning) |
| `Esc Esc` | Interrupt Claude mid response or restore code |

The ones that matter most:

**`Shift+Enter`** for multi line prompts. Instead of sending three separate messages (three turns of context), compose one detailed prompt. This saves context and gives Claude complete instructions upfront.

**`Esc Esc`** to interrupt. If Claude is going down the wrong path, don't wait for it to finish. Double escape, then redirect. Every token Claude generates on the wrong path is wasted.

**`!`** for quick shell commands. `! git status`, `! npm test`, `! ls src/`. Faster than asking Claude to run them.

**`@`** to reference files. `@src/routes/users.ts` pulls the file into context directly. More precise than asking Claude to find it.

### Practice: Build the Muscle Memory

Open Claude and practice the flow:

```
! git status
```

<!-- screenshot: quick bash with ! prefix -->

Now try a multi line prompt with `Shift+Enter`:

```
Review these two files for consistency:        [Shift+Enter]
@packages/api/src/users.ts                     [Shift+Enter]
@packages/web/src/components/UserList.tsx       [Enter to send]
```

Claude gets both files referenced directly. No searching needed. Parallel reads.

<!-- screenshot: multi line prompt with @ file references -->


## Parallel Workflows

Running multiple Claude instances is powerful but easy to misuse. The longform guide puts it bluntly: "Your goal should be how much you can get done with the minimum viable amount of parallelization."

For most work, a single Claude session with `/fork` for research is enough. When you genuinely need parallel code changes that touch different parts of the codebase, use git worktrees.

### Git Worktrees

A worktree is an independent checkout of your repo. Each worktree has its own working directory, so two Claude instances can edit different files without conflicts.

```bash
# Create worktrees for parallel work
git worktree add ../project-feature-auth feature-auth
git worktree add ../project-feature-billing feature-billing

# Run Claude in each worktree
cd ../project-feature-auth && claude
cd ../project-feature-billing && claude
```

Each Claude instance sees its own clean working directory. No merge conflicts during the session. You merge the branches afterward through normal git workflow.

### The Cascade Method

When you do run multiple sessions, organize them:

1. Open new tasks in new tabs to the right
2. Sweep left to right, oldest to newest
3. Check on specific tasks as needed
4. **Maximum 3 to 4 parallel tasks.** Beyond that, mental overhead increases faster than productivity.

Use `/rename` on every session so you know what's what:

```
/rename auth-feature
/rename billing-refactor
/rename research
```

> Start with a single instance. Add a second for research or a non overlapping task. Only go to 3 or 4 if the tasks are truly independent and you have clear plans for each. Most of the time 2 is plenty.

<!-- screenshot: multiple terminal tabs with renamed Claude sessions -->


## Key Takeaways

- **Split screen**: terminal with Claude on one side, editor on the other. Enable auto save and file watchers in your editor.

- **tmux** for long running processes. Dev servers, builds, and test suites should run in tmux sessions, not in Claude's context. This saves tokens and survives connection drops.

- **`/compact` manually** at phase transitions instead of relying on auto compact. You control when context gets summarized.

- **`/fork` for research.** Main session for code changes, forked session for questions and exploration. Label both with `/rename`.

- **`/rewind`** when Claude goes wrong. Going backward is cheaper than arguing forward. **`/checkpoints`** for file level undo.

- **`Shift+Enter`** for multi line prompts. **`Esc Esc`** to interrupt. **`!`** for quick bash. **`@`** for direct file references. These save seconds that compound across a full day.

- **Git worktrees** for genuinely parallel code changes. `/fork` for parallel conversations within one codebase. Don't exceed 3 to 4 parallel sessions.

- **The minimum viable parallelization.** Start with one instance. Add more only when you have a clear, non overlapping task for each.
