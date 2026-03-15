# Skills, Commands and Hooks

In previous lab we set up the configuration hierarchy: CLAUDE.md files and rules that tell Claude what conventions to follow. That's passive context. It loads at startup and sits there.

This lab is about the active side. Skills and commands are workflows you trigger on demand. Hooks are automations that fire on events without you asking. Together they turn Claude Code from a chatbot that knows your conventions into a system that executes repeatable workflows and enforces standards automatically.

The mental model: CLAUDE.md is what Claude *knows*. Skills and commands are what Claude *does* when asked. Hooks are what Claude *does* without being asked.


## Skills vs Commands

People mix these up constantly because they overlap. Both are markdown files. Both influence Claude's behavior. The difference is how they're stored and when they activate.

| | Commands | Skills |
|--|---------|--------|
| Location | `.claude/commands/` | `.claude/skills/` |
| Invoked by | Slash command (`/review`) | Claude loads when relevant, or on demand |
| Format | Plain markdown, optional frontmatter | `SKILL.md` with structured frontmatter |
| Scope | Project (`.claude/commands/`) or user (`~/.claude/commands/`) | Project (`.claude/skills/`) or user (`~/.claude/skills/`) |
| Think of it as | A prompt you saved and can rerun | A knowledge module with workflow logic |

**Commands** are shorthand for prompts you'd otherwise type repeatedly. "Run my team's code review checklist." "Generate tests following our TDD pattern." "Clean up dead code." You turn that prompt into a file, drop it in `.claude/commands/`, and now it's a slash command anyone on the team can run.

**Skills** are deeper. They carry domain knowledge, activation conditions, and configuration through frontmatter. A skill might define your entire TDD workflow: when to activate, which tools it's allowed to use, whether it runs in an isolated context. Skills can also be loaded automatically when Claude recognizes a relevant situation instead of waiting for you to invoke them.

In practice, you'll use commands for quick repeatable actions and skills for complex workflows that need isolation or tool restrictions.


## Building Commands

Commands live in `.claude/commands/` for project scope (shared via git) or `~/.claude/commands/` for personal commands. Each file becomes a slash command. The filename minus the `.md` extension is the command name.

Let's continue with the monorepo from Lab 01 and add some commands to it.

```bash
mkdir -p .claude/commands
```

### A Simple Code Review Command

Create `.claude/commands/review.md`:

```markdown
Review all uncommitted changes in this project:

1. Run `git diff --name-only HEAD` to get changed files
2. For each changed file, check for:

**Security (block commit if found):**
- Hardcoded credentials, API keys, tokens
- SQL injection vulnerabilities (string concatenation in queries)
- Missing input validation on route handlers

**Quality (warn):**
- Functions longer than 50 lines
- Files longer than 800 lines
- Nesting deeper than 4 levels
- Missing error handling in async functions

3. Generate a report with severity, file location, description, and suggested fix
4. If any CRITICAL issues found, say so clearly at the top
```

That's it. Now any developer on the team can run `/review` and get the same consistent code review. No one has to remember the checklist or type it out.

### A TDD Command

Create `.claude/commands/tdd.md`:

```markdown
Run a TDD cycle on the task I describe:

1. **RED**: Write failing tests first
   - Use `describe`/`it` blocks (not `test()`)
   - Cover the happy path, edge cases, and error cases
   - Run the tests, confirm they fail

2. **GREEN**: Write the minimal code to make tests pass
   - Don't over engineer, just make the tests green
   - Run the tests, confirm they pass

3. **IMPROVE**: Refactor
   - Clean up duplication
   - Improve naming
   - Run the tests again, confirm they still pass

4. Report what was created: which test file, which source file, coverage if available

$ARGUMENTS
```

The `$ARGUMENTS` placeholder at the bottom means whatever the user types after the command gets injected there. So `/tdd add email validation to the user creation endpoint` passes that description into the prompt.

### A Personal Command

Commands you don't want to share with the team go in your user directory:

```bash
mkdir -p ~/.claude/commands
```

Create `~/.claude/commands/cleanup.md`:

```markdown
Clean up the current file or the files I specify:

- Remove unused imports
- Remove unused variables and functions
- Remove empty blocks and dead code branches
- Remove console.log statements
- Don't add comments explaining what you removed
- Don't refactor working logic, just clean
```

This is personal. It reflects how *you* define "clean up." A teammate might want a different version. That's fine because it's in `~/.claude/commands/`, not in the project repo.


### Practice: Run Your Commands

Open Claude in the project root:

```bash
claude
```

Type `/` and you should see your commands in the autocomplete:

<!-- screenshot: slash command autocomplete showing /review, /tdd -->

Run the review command:

```
/review
```

Claude will execute the code review workflow you defined. Watch it run `git diff`, read the changed files, and apply your checklist. Same result every time, any team member, no prompt engineering on the fly.

<!-- screenshot: Claude executing /review command, showing the review process -->

Now try the TDD command:

```
/tdd add a PATCH endpoint for updating user email
```

Claude should write failing tests first, then implement, then refactor. The `$ARGUMENTS` placeholder received your task description.

<!-- screenshot: Claude running /tdd with RED GREEN IMPROVE phases -->


## Building Skills

Skills are more powerful than commands. They use `SKILL.md` files with YAML frontmatter that controls how the skill behaves: whether it runs in isolation, which tools it can access, and what arguments it expects.

```bash
mkdir -p .claude/skills/security-review
```

### A Skill with Frontmatter

Create `.claude/skills/security-review/SKILL.md`:

```markdown
---
name: security-review
description: Run a security focused review of the codebase. Checks for OWASP top 10 vulnerabilities, hardcoded secrets, and insecure patterns.
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "Specify files or directories to review, or leave blank for full scan"
---

# Security Review

Perform a thorough security audit:

## Check for Secrets
- Grep for patterns: API keys, tokens, passwords, connection strings
- Check .env files are in .gitignore
- Look for hardcoded URLs with credentials

## Check for Injection
- SQL: find string concatenation in database queries
- XSS: find unescaped user input in HTML output
- Command injection: find user input passed to exec/spawn

## Check for Auth Issues
- Routes without authentication middleware
- Missing authorization checks (user accessing other user's data)
- Sensitive endpoints without rate limiting

## Report Format
For each finding:
- **Severity**: CRITICAL / HIGH / MEDIUM / LOW
- **File**: path and line number
- **Issue**: what's wrong
- **Fix**: how to fix it
```

Three frontmatter fields make this different from a plain command:

**`context: fork`** runs the skill in an isolated sub agent context. The security review will read lots of files, grep for patterns, and generate verbose output. Without fork, all of that floods your main conversation and eats context. With fork, the skill runs separately and returns a summary. Your main session stays clean.

**`allowed-tools`** restricts what the skill can do. This security review only needs to read and search. It can't edit files, run bash commands, or write anything. This is a safety constraint. A review skill that can also modify code is a bad idea.

**`argument-hint`** tells the developer what to provide when invoking the skill. If they run it without arguments, Claude shows this hint.

> `context: fork` is the most important frontmatter option to understand. Any skill that produces verbose output (codebase analysis, exploration, brainstorming) should run forked. Skills that make targeted edits to specific files can run in the main context.

### A Codemap Updater Skill

This is a pattern from the shorthand guide. A skill that maintains a map of your codebase so Claude can navigate quickly without burning context on exploration every session.

```bash
mkdir -p .claude/skills/codemap-updater
```

Create `.claude/skills/codemap-updater/SKILL.md`:

```markdown
---
name: codemap-updater
description: Update the project codemap file that tracks key files, their purpose, and relationships. Run at session start or after major structural changes.
context: fork
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
---

# Codemap Updater

Scan the project structure and update `.claude/codemap.md` with:

## For Each Package/Module
- Entry point file
- Key source files and their purpose (one line each)
- Test file locations
- Config files

## Relationships
- Which modules import from which
- Shared utilities and where they're used
- Database models and which routes use them

## Format
Keep it under 100 lines. This file is read at session start for quick orientation.
Don't include node_modules, build artifacts, or generated files.
Write the result to `.claude/codemap.md`.
```

This skill has `Write` in its allowed tools because it needs to create the codemap file. But it still runs forked so the exploration doesn't pollute your main context.


### Practice: Test Skill Isolation

Run the security review skill:

```
/security-review packages/api/
```

Watch the output. Because of `context: fork`, Claude spawns a sub agent that does all the reading and grepping. Your main conversation gets the summary, not the hundreds of lines of grep output.

<!-- screenshot: forked skill returning a summary to main context -->

Now compare. If you had run the same review workflow as a regular prompt without fork, your main context would be filled with every file read and grep result. That's the difference fork makes.


## Hooks

Skills and commands are things you choose to run. Hooks are things that run automatically when specific events happen. You don't invoke a hook. It fires on its own when its trigger condition is met.

| Hook | When It Fires | Common Use |
|------|--------------|------------|
| `PreToolUse` | Before a tool executes | Block dangerous operations, show reminders |
| `PostToolUse` | After a tool finishes | Auto format code, run type checks, warn about issues |
| `UserPromptSubmit` | When you send a message | Validate input, add context |
| `Stop` | When Claude finishes responding | Audit changes, extract patterns, cleanup |
| `PreCompact` | Before context compaction | Save important state |
| `Notification` | On permission requests | Custom approval flows |

Hooks are configured in your Claude Code settings as JSON. Each hook has a **matcher** (which tool or event it applies to) and a **command** (what to run).

### Hook Configuration

The hook config lives in your Claude Code settings. Here's the structure:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "your-script-here.sh"
          }
        ]
      }
    ]
  }
}
```

Let's build the hooks from the shorthand guide, the ones that actually matter day to day.


### Hook 1: Block Unnecessary .md File Creation

Claude loves creating markdown files. Documentation files, notes, summaries, plans. Most of the time you don't want them cluttering your repo. This hook blocks `.md` file creation except for files you actually want (README, CLAUDE.md, CHANGELOG).

```json
{
  "matcher": "Write",
  "hooks": [
    {
      "type": "command",
      "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const f=i.tool_input?.file_path||'';if(f.endsWith('.md')&&!/(README|CLAUDE|CHANGELOG|codemap)/i.test(f)){console.error('[Hook] BLOCKED: Creating non-standard .md file: '+f);console.error('[Hook] Only README.md, CLAUDE.md, CHANGELOG.md, and codemap.md are allowed');process.exit(2)}console.log(d)})\""
    }
  ],
  "description": "Block creation of random .md files"
}
```

The key here is `process.exit(2)`. Exit code 2 tells Claude Code to **block the tool call**. Exit code 0 means continue. Any other non zero code logs a warning but doesn't block.

> This is the critical concept: hooks provide **deterministic enforcement**. Telling Claude "don't create .md files" in your CLAUDE.md is probabilistic. Claude will usually follow it. A hook with exit code 2 will always block it. When something must never happen, use a hook. When something should usually happen, use a rule.


### Hook 2: Auto Format After Edits

Every time Claude edits a `.ts` or `.tsx` file, run the formatter automatically:

```json
{
  "matcher": "Edit",
  "hooks": [
    {
      "type": "command",
      "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const f=i.tool_input?.file_path||'';if(/\\.(ts|tsx|js|jsx)$/.test(f)){const{execSync}=require('child_process');try{execSync('npx prettier --write '+f,{stdio:'pipe'});console.error('[Hook] Formatted: '+f)}catch(e){console.error('[Hook] Format failed: '+e.message)}}console.log(d)})\""
    }
  ],
  "description": "Auto-format JS/TS files after edits"
}
```

This is a `PostToolUse` hook on the `Edit` tool. After Claude edits a file, the hook runs Prettier on it. Claude never has to think about formatting. The hook handles it.


### Hook 3: Console.log Audit on Session End

A `Stop` hook that checks all modified files for `console.log` statements before the session ends:

```json
{
  "matcher": "*",
  "hooks": [
    {
      "type": "command",
      "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const{execSync}=require('child_process');try{const files=execSync('git diff --name-only HEAD',{encoding:'utf8'}).trim().split('\\n').filter(f=>/\\.(ts|tsx|js|jsx)$/.test(f));let found=false;files.forEach(f=>{try{const c=require('fs').readFileSync(f,'utf8');const lines=c.split('\\n');lines.forEach((l,i)=>{if(/console\\.log/.test(l)){console.error('[Hook] console.log found in '+f+':'+(i+1));found=true}})}catch(e){}});if(found){console.error('[Hook] Remove console.log statements before committing')}}catch(e){}console.log(d)})\""
    }
  ],
  "description": "Audit modified files for console.log on session end"
}
```

The `matcher: "*"` means this fires on every `Stop` event regardless of what tool was last used. It checks git modified files for `console.log` and warns you. It doesn't block because it's a Stop hook (blocking doesn't make sense at session end), it just alerts.


### Hook 4: tmux Reminder for Long Commands

Before Claude runs package manager commands (npm, yarn, pytest, cargo), remind about tmux if you're not already in a tmux session:

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const cmd=i.tool_input?.command||'';if(/(npm|pnpm|yarn|cargo|pytest|vitest)/.test(cmd)&&!process.env.TMUX){console.error('[Hook] Consider running this in tmux for session persistence')}console.log(d)})\""
    }
  ],
  "description": "Remind about tmux for long-running commands"
}
```

This doesn't block. It's a gentle nudge. The command still runs, but you get a reminder in stderr.


### Putting the Hooks Together

Your complete hook configuration:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "..." }],
        "description": "Block random .md file creation"
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "..." }],
        "description": "tmux reminder for long commands"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "..." }],
        "description": "Auto-format JS/TS after edits"
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [{ "type": "command", "command": "..." }],
        "description": "Audit for console.log in modified files"
      }
    ]
  }
}
```

Four hooks covering: file creation control, developer reminders, automatic formatting, and session end audits. That's a solid baseline for any TypeScript project.


### Practice: See Hooks in Action

With your hooks configured, try creating a random markdown file:

```
Create a file called notes.md with some project notes
```

The PreToolUse hook should block it and you'll see the `[Hook] BLOCKED` message.

<!-- screenshot: hook blocking .md file creation with error message -->

Now edit a TypeScript file:

```
Add a console.log("debug") to packages/api/src/users.ts
```

After the edit, the PostToolUse hook should auto format the file. When the session ends or Claude finishes responding, the Stop hook will warn about the console.log.

<!-- screenshot: PostToolUse hook auto-formatting, Stop hook warning about console.log -->


## Deterministic vs Probabilistic

This is worth calling out explicitly because the certification exam tests it.

When you write "never create .md files" in your CLAUDE.md, Claude will follow that instruction most of the time. Maybe 95% of the time. That's **probabilistic** enforcement. For style preferences and soft guidelines, that's fine.

When you create a PreToolUse hook that exits with code 2, the file creation is blocked 100% of the time. That's **deterministic** enforcement. For business rules, security constraints, and anything where "most of the time" isn't good enough, hooks are the right tool.

| Approach | Enforcement | Use When |
|----------|-------------|----------|
| CLAUDE.md / rules | Probabilistic (LLM follows instructions) | Style preferences, coding conventions, soft guidelines |
| Hooks with exit code 2 | Deterministic (tool call blocked) | Security rules, file restrictions, business logic that must never be violated |
| Hooks with exit code 0 | Advisory (warning only) | Reminders, formatting, logging, non blocking checks |

The question is never "should I use hooks or rules." It's "does this rule need a 100% guarantee or is 95% good enough?" Use both layers together. Rules for conventions. Hooks for enforcement.

> Use the `hookify` plugin to create hooks conversationally instead of writing JSON by hand. Run `/hookify` and describe what you want. It generates the hook config for you.


## Key Takeaways

- **Commands** are saved prompts. Drop a markdown file in `.claude/commands/` and it becomes a slash command. Project scoped commands are shared via git, user scoped commands are personal.

- **Skills** are deeper workflow definitions. `SKILL.md` with frontmatter controls isolation (`context: fork`), tool access (`allowed-tools`), and argument hints.

- **`context: fork`** is the most important skill option. Any skill that produces verbose output should run forked to keep your main context clean.

- **Hooks** fire automatically on events. `PreToolUse` runs before a tool, `PostToolUse` runs after, `Stop` runs when Claude finishes responding.

- **Exit code 2** in a PreToolUse hook blocks the tool call. This is deterministic enforcement, unlike prompt instructions which are probabilistic.

- **Use rules for conventions, hooks for enforcement.** If "usually follows" is fine, put it in CLAUDE.md. If "always follows" is required, make it a hook.

- **Chain commands.** You can run `/tdd add email validation` and `/review` in the same session. Commands compose naturally because they're just prompts.
