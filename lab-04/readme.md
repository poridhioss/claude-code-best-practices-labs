# Subagents and MCPs

In the previous labs we configured how Claude behaves (rules), what workflows it can run (skills and commands), and what it enforces automatically (hooks). All of that happens inside a single Claude session, a single conversation with a single context window.

This lab introduces two ways to extend beyond that single session. Subagents let you delegate work to separate Claude instances with their own scoped context and tools. MCPs (Model Context Protocol) connect Claude to external services like databases, GitHub, and deployment platforms. Together they turn Claude Code from a solo operator into an orchestrator that delegates specialized work and reaches into your infrastructure.


## Subagents

When Claude is working on a complex task, it accumulates context. Every file it reads, every grep result, every tool output fills the 200k token window. At some point, the context gets noisy and Claude starts losing track of earlier findings. This is context rot.

Subagents solve this by delegation. Instead of doing everything in the main session, Claude spawns a separate agent with its own clean context, scoped tools, and a focused task. The subagent does the work, returns a summary, and the main session stays lean.

Think of it like a senior engineer delegating to specialists. You don't have the architect also doing the QA testing. Each role gets the tools it needs and nothing more.

### Agent Definitions

Agents are markdown files in `.claude/agents/` (project scope) or `~/.claude/agents/` (user scope). Each file defines a specialist with a description, instructions, and tool restrictions.

```bash
mkdir -p .claude/agents
```

### Planner Agent

Create `.claude/agents/planner.md`:

```markdown
---
name: planner
description: Break down feature requests into implementation plans. Use when a task involves multiple files, unclear scope, or architectural decisions.
tools: Read, Glob, Grep
model: sonnet
---

# Planner Agent

You are a planning specialist. Your job is to analyze a feature request and produce a clear implementation plan.

## Process
1. Explore the codebase to understand current architecture
2. Identify which files need to change
3. List the changes in order of dependency
4. Flag any risks or decisions that need human input

## Output Format
Return a structured plan:
- **Goal**: one sentence summary
- **Files to change**: list with what changes in each
- **New files needed**: list with purpose of each
- **Dependencies**: what must happen before what
- **Risks**: anything unclear or potentially breaking

## Rules
- Do NOT write any code
- Do NOT edit any files
- Only read, search, and analyze
- Keep your plan under 50 lines
```

Notice the frontmatter. **`tools: Read, Glob, Grep`** means this agent can only read and search. It cannot edit files, run bash commands, or write anything. A planner that accidentally starts implementing defeats the purpose.

**`model: sonnet`** pins this agent to Sonnet. You could use `haiku` for simpler agents (file search, formatting) or `opus` for agents that need deep reasoning (architecture decisions, security review).

### Code Reviewer Agent

Create `.claude/agents/code-reviewer.md`:

```markdown
---
name: code-reviewer
description: Review code changes for quality, security, and convention compliance. Use after implementation to catch issues before commit.
tools: Read, Glob, Grep
model: sonnet
---

# Code Reviewer Agent

You review code changes for quality and security issues.

## Review Checklist

**Security (CRITICAL)**
- Hardcoded secrets or credentials
- SQL injection (string concatenation in queries)
- Unvalidated user input
- Missing authentication or authorization checks

**Quality (HIGH)**
- Functions over 50 lines
- Missing error handling in async code
- Unused imports or dead code
- Inconsistent naming patterns

**Convention (MEDIUM)**
- Matches project CLAUDE.md standards
- Test coverage for new code
- Proper typing (no `any`)

## Output
For each finding: severity, file, line, issue, suggested fix.
Sort by severity (CRITICAL first).
```

### Build Error Resolver Agent

Create `.claude/agents/build-error-resolver.md`:

```markdown
---
name: build-error-resolver
description: Fix build errors, type errors, and test failures. Use when CI is red or local builds fail.
tools: Read, Edit, Bash, Grep, Glob
model: sonnet
---

# Build Error Resolver

You fix build and type errors.

## Process
1. Run the build or test command to see the errors
2. Read the failing files
3. Fix the errors one at a time
4. Re-run to verify the fix
5. Repeat until clean

## Rules
- Fix only what's broken, don't refactor surrounding code
- If a fix would change behavior (not just fix types), flag it and stop
- Run the build/test after every fix to confirm
```

This agent has `Edit` and `Bash` in its tools because it actually needs to fix code and run builds. The planner and reviewer don't.

### How Subagents Get Invoked

In Claude Code, subagents are invoked through the Task tool. When the main Claude session decides a task is better handled by a specialist, it spawns a subagent. The key behavior to understand:

- **Isolated context**: subagents do NOT inherit the main session's conversation history. They start fresh with only what you pass in the prompt.
- **Scoped tools**: they can only use the tools listed in their `tools` frontmatter.
- **Summary return**: the subagent works, then returns a summary to the main session. The verbose output stays in the subagent's context, not yours.

This is the same pattern as `context: fork` in skills, but applied to agents. The difference is that agents have persistent definitions with model selection and tool scoping, while skills are workflow templates.


### Practice: Use a Subagent

Ask Claude to plan a feature using the planner agent:

```
Use the planner agent to create an implementation plan for adding
a password reset flow to the user management API
```

Watch Claude spawn the planner as a subagent. The planner will read files, grep for auth patterns, and return a structured plan. Your main context gets the plan summary, not all the file reads.

<!-- screenshot: Claude delegating to planner subagent and receiving plan summary -->

Now ask for a review:

```
Use the code-reviewer agent to review the changes in packages/api/src/users.ts
```

<!-- screenshot: code-reviewer subagent returning findings -->

> When to use subagents vs forked skills: if you need a reusable specialist with specific model and tool config, make it an agent. If you need a one off workflow that runs isolated, use a skill with `context: fork`.


## MCPs (Model Context Protocol)

MCPs connect Claude to external services. Instead of copy pasting database results or manually fetching API responses, Claude talks to the service directly through the MCP protocol.

A Supabase MCP lets Claude query your database. A GitHub MCP lets Claude read issues, create PRs, review code. A Vercel MCP lets Claude check deployment status. Claude uses these as tools just like it uses Read or Grep, but they reach outside your local machine.

### Project Level MCP Configuration

MCP servers shared with the team go in `.mcp.json` at the project root. This file is committed to git so everyone gets the same integrations.

Create `.mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    }
  }
}
```

The `${GITHUB_TOKEN}` syntax is environment variable expansion. The actual token lives in each developer's environment, not in the committed file. This is how you share MCP configs without leaking credentials.

### User Level MCP Configuration

Personal or experimental MCPs go in `~/.claude.json`. These aren't shared with the team.

Add to your `~/.claude.json`:

```json
{
  "mcpServers": {
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    }
  }
}
```

Both project level and user level MCPs are discovered at startup and available simultaneously. Claude sees all the tools from all configured servers.


### The Context Window Problem

This is the part most people learn the hard way. Every MCP server exposes tools. Each tool has a description that gets loaded into Claude's context. The GitHub MCP alone exposes dozens of tools. Add Supabase, Vercel, Cloudflare, and a few others and you've consumed a significant chunk of your 200k context window before you've even started working.

The math is brutal. Your 200k context window before compacting might be only 70k with too many tools enabled. Claude's performance degrades because it's spending attention on tool descriptions instead of your actual task.

**Rule of thumb from the shorthand guide:** have 20 to 30 MCPs configured, but keep under 10 enabled and under 80 tools active at any given time.

### Disabling MCPs Per Project

You manage this by disabling MCPs you don't need for the current project. In your project settings or `~/.claude.json`:

```json
{
  "projects": {
    "/path/to/project": {
      "disabledMcpServers": [
        "cloudflare-docs",
        "cloudflare-workers-builds",
        "clickhouse",
        "AbletonMCP"
      ]
    }
  }
}
```

Or use the CLI:

```
/mcp
```

This shows all configured MCP servers and their status. Disable what you don't need. Enable what you do.

<!-- screenshot: /mcp output showing enabled and disabled servers -->


### Practice: Measure Context Window Impact

Before enabling any MCPs, check your available context:

```
How much context window do you have available right now?
```

<!-- screenshot: context window before MCPs -->

Now enable a few MCPs and ask again. You'll see the difference. This is why being selective matters.


## Plugins

Plugins bundle tools, skills, hooks, and MCPs into packages you can install from marketplaces. Instead of manually configuring everything, you install a plugin and it sets up what it needs.

### Installing mgrep

`mgrep` is a meaningful upgrade over the default grep/ripgrep. It's a semantic search tool that understands code better. Install it from the marketplace:

```bash
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep
```

Then open Claude and run `/plugins` to find the new marketplace and install from there.

<!-- screenshot: /plugins showing mgrep marketplace -->

Once installed, `mgrep` replaces grep for code search:

```bash
mgrep "function handleSubmit"           # Local search
mgrep --web "Next.js 15 app router"    # Web search
```

The token reduction is significant. On various tasks, mgrep uses about half the tokens compared to traditional grep because it returns more relevant results with less noise.

### LSP Plugins

If you run Claude Code from the terminal without an IDE, LSP (Language Server Protocol) plugins give Claude the same intelligence your editor has: type checking, go to definition, intelligent completions.

```bash
# Useful LSP plugins
typescript-lsp@claude-plugins-official  # TypeScript intelligence
pyright-lsp@claude-plugins-official     # Python type checking
```

> Same warning as MCPs: every plugin adds tools to the context window. Only enable what you're actively using. Run `/plugins` to manage what's active.


## Putting It Together

At this point your Claude Code setup has four layers:

| Layer | What It Does | Configured In |
|-------|-------------|---------------|
| Rules and CLAUDE.md | Passive conventions Claude follows | `CLAUDE.md`, `.claude/rules/` |
| Skills and Commands | Workflows you trigger on demand | `.claude/skills/`, `.claude/commands/` |
| Hooks | Automations that fire on events | Settings JSON |
| Subagents and MCPs | Delegation and external service access | `.claude/agents/`, `.mcp.json` |

Each layer serves a different purpose. Rules define what. Commands define how. Hooks enforce always. Agents delegate who. MCPs connect where. The rest of this lab series builds on all four layers.


## Key Takeaways

- **Subagents** are specialists with isolated context, scoped tools, and a specific model. They do focused work and return summaries, keeping your main session clean.

- **Agent definitions** live in `.claude/agents/` with frontmatter for `tools`, `model`, and `description`. Only give agents the tools they need for their role.

- **Subagents don't inherit context.** You must pass everything they need in the prompt. This is by design, not a limitation.

- **MCPs** connect Claude to external services (databases, GitHub, deployment platforms). Project level configs go in `.mcp.json` with env var expansion. Personal MCPs go in `~/.claude.json`.

- **Context window is precious.** Every MCP and plugin adds tool descriptions that eat your 200k window. Keep under 10 MCPs enabled and under 80 tools active. Disable what you don't need per project.

- **Plugins** bundle tools for easy install. `mgrep` is a direct upgrade over grep with ~50% token reduction. LSP plugins add editor intelligence to terminal Claude.

- **Use `/mcp`** and **`/plugins`** to see what's active and manage your context budget.
