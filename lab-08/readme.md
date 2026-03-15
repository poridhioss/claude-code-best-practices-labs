# Continuous Learning and Token Optimization

Two problems that compound over time: Claude forgetting lessons from previous sessions, and burning tokens on work that could be cheaper. Continuous learning addresses the first — Claude automatically extracts patterns from sessions and saves them as reusable skills. Token optimization addresses the second — routing tasks to the right model, replacing expensive tools with cheaper alternatives, and keeping context lean.

Both problems share a root cause. If Claude keeps re discovering the same patterns and you keep paying full price for work that does not need the most expensive model, you are losing compounding gains. Fix both and the savings multiply across every session.


## The Continuous Learning Problem

You have had this experience: Claude makes the same mistake in session 5 that you corrected in session 2. You spend a prompt re steering, Claude apologizes and does it right, and next week it happens again. The knowledge from that correction existed for exactly one session and then vanished.

**The problem:** Wasted tokens, wasted context, wasted time. You fire a second prompt to re steer and calibrate Claude's compass. Those patterns must be appended to skills so they persist.

**The solution:** When Claude discovers something that is not trivial — a debugging technique, a workaround, a project specific pattern — it saves that knowledge as a new skill. Next time a similar problem comes up, the skill gets loaded automatically.

### How It Works

The continuous learning system uses a Stop hook. When your session ends, a script analyzes the session for patterns worth extracting: error resolutions, debugging techniques, workarounds, project specific patterns. It saves them as reusable skills in `~/.claude/skills/learned/`.

> Why a Stop hook instead of UserPromptSubmit? `UserPromptSubmit` runs on every single message you send — that is a lot of overhead, adds latency to every prompt, and is overkill for this purpose. `Stop` runs once at session end — lightweight, does not slow you down during the session, and evaluates the complete session rather than piecemeal.

### Installation

```bash
# Clone the full repo to skills folder
git clone https://github.com/affaan-m/everything-claude-code.git ~/.claude/skills/everything-claude-code

# Or grab just the continuous learning skill
mkdir -p ~/.claude/skills/continuous-learning
curl -sL https://raw.githubusercontent.com/affaan-m/everything-claude-code/main/skills/continuous-learning/evaluate-session.sh \
  > ~/.claude/skills/continuous-learning/evaluate-session.sh
chmod +x ~/.claude/skills/continuous-learning/evaluate-session.sh
```

### Hook Configuration

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
          }
        ]
      }
    ]
  }
}
```

The Stop hook triggers when your session ends. The script analyzes the session for patterns worth extracting and saves them as reusable skills in `~/.claude/skills/learned/`.

### Manual Extraction with /learn

You do not have to wait for session end. The `/learn` command lets you extract a pattern right when you solve something non trivial. It prompts you to describe what you learned, drafts a skill file, and asks for confirmation before saving.

```
/learn
```

Use this when you just solved a tricky problem and the solution is fresh. Do not rely solely on the Stop hook — some patterns are obvious in the moment but subtle when reviewed later.

<!-- screenshot: /learn command extracting a pattern mid session -->


### Practice: Install and Verify Continuous Learning

Install the continuous learning skill and configure the Stop hook. Then start a session and deliberately solve a problem that requires a non obvious approach:

```
Fix the timezone handling in the date formatting utility. The tests expect UTC but the formatter uses local time.
```

After solving it, end the session. Check if the Stop hook created a learned skill:

```bash
ls ~/.claude/skills/learned/
```

<!-- screenshot: learned skill file created after session end -->

Start a new session and verify the skill loads:

```
/memory
```

The learned skill should appear in the loaded configuration files.


## Token Optimization

Tokens cost money and context is finite. Every inefficiency compounds across sessions. The strategies here target the biggest sources of waste.

### Model Routing

Not every task needs Opus. Most do not even need Sonnet. The cost differences are significant:

| Model | Input / Output Price | Cost vs Opus |
|-------|---------------------|-------------|
| **Haiku 4.5** | $1 / $5 per 1M tokens | 5x cheaper |
| **Sonnet 4.5** | $3 / $15 per 1M tokens | 1.67x cheaper |
| **Opus 4.6** | $5 / $25 per 1M tokens | Baseline |

The practical routing:

| Guideline | When |
|-----------|------|
| **Default to Sonnet** | 90% of coding tasks |
| **Upgrade to Opus** | First attempt failed, task spans 5+ files, architectural decisions, or security critical code |
| **Downgrade to Haiku** | Task is repetitive, instructions are very clear, or using as a "worker" in multi agent setup |

Sonnet covers daily work. Opus is for when Claude is making wrong architectural decisions or missing edge cases. Haiku is for repetitive tasks where the instructions are unambiguous.

The cost math: Haiku vs Opus is a 5x cost difference, compared to a 1.67x difference between Sonnet and Opus. **Haiku + Opus combo makes the most sense** — use Haiku for the cheap work and Opus for the hard work, skipping Sonnet entirely in cost sensitive setups.

In your agent definitions, specify the model:

```yaml
---
name: quick-search
description: Fast file search and simple lookups
tools: Glob, Grep
model: haiku
---
```

```yaml
---
name: architect
description: System design and architectural decisions
tools: Read, Glob, Grep
model: opus
---
```

### Practice: Compare Model Costs

Switch to Haiku and ask Claude to do a simple task:

```
/model
```

Select Haiku, then:

```
Find all files that import from the database module.
```

<!-- screenshot: Haiku handling a simple search task -->

Now switch to Opus and ask something that requires deep reasoning:

```
/model
```

Select Opus, then:

```
Review the authentication flow across all middleware, route handlers, and database queries. Identify any security gaps.
```

> The rule: if the task has a clear, unambiguous answer, use Haiku. If it requires judgment, architectural understanding, or cross file reasoning, use Opus. Sonnet is the middle ground when you are not sure.


### mgrep vs grep Token Comparison

The default grep tool returns raw matches — often far more output than Claude needs. `mgrep` is a semantic search tool that returns more relevant results with less noise. On various tasks, mgrep uses about half the tokens compared to traditional grep.

```bash
# Traditional grep - returns every match, lots of noise
grep -r "handleSubmit" src/

# mgrep - semantic search, relevant results only
mgrep "function handleSubmit"
```

Install mgrep via the plugin marketplace:

```bash
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep
```

Then install from `/plugins` inside Claude.

<!-- screenshot: mgrep returning focused results vs grep flooding with matches -->

The token reduction matters because grep output goes into Claude's context. If grep returns 200 lines and Claude only needs 10, you just wasted 190 lines of context tokens. mgrep returns the 10 lines directly.


### MCP to CLI Conversion

MCPs connect Claude to external services, but every MCP adds tool descriptions to the context window. For services that already have robust CLIs — GitHub, Supabase, Vercel, Railway — the MCP is essentially wrapping the CLI in a context expensive way.

The alternative: strip out the tools the MCP exposes and turn them into commands or skills.

**Instead of the GitHub MCP loaded at all times:**

```bash
# Create a /gh-pr command
mkdir -p .claude/commands
```

```markdown
# .claude/commands/gh-pr.md
Create a pull request using the GitHub CLI.

Run: gh pr create --title "$TITLE" --body "$BODY"
```

**Instead of the Supabase MCP:**

Create skills that use the Supabase CLI directly. The functionality is the same, the convenience is similar, but your context window is freed up for actual work.

> With lazy loading of MCPs (a recent improvement), the context window issue is mostly solved. But token usage and cost is not solved the same way. The CLI + skills approach is still a token optimization method because you can run MCP operations via CLI instead of in context, which reduces token usage significantly for heavy operations like database queries or deployments.


### Background Processes

When applicable, run background processes outside Claude if you do not need Claude to process the entire output. This ties directly into the tmux patterns from Lab 05.

```bash
# Instead of running in Claude's context:
# "Run the test suite"

# Run in tmux and bring back only what matters:
tmux new -s tests -d "npm test 2>&1 | tail -20 > /tmp/test-results.txt"
```

Take the terminal output and either summarize it or copy only the part you need. This saves on input tokens, which is where the majority of cost comes from ($5/M tokens for Opus input, $25/M for output).


### Modular Codebase Benefits

Having a more modular codebase with reusable utilities and main files in the hundreds of lines instead of thousands helps both in token optimization and getting a task done right on the first try. If you have to prompt Claude multiple times because it loses information while reading a 2000 line file, you are burning through tokens.

The leaner your codebase is, the cheaper your token cost will be. Identify dead code by using skills to continuously clean the codebase by refactoring. At certain points, go through and skim the whole codebase looking for things that stand out or look repetitive, manually piece together that context, and then feed that into Claude alongside refactor and dead code skills.


## Key Takeaways

- **Continuous learning** uses a Stop hook to extract patterns at session end and save them as reusable skills in `~/.claude/skills/learned/`. Use `/learn` mid session for immediate extraction.

- **Stop hook over UserPromptSubmit** — Stop runs once at session end (lightweight), UserPromptSubmit runs on every message (overhead). Session level evaluation beats message level evaluation.

- **Model routing** is the biggest cost lever. Haiku for simple tasks (5x cheaper than Opus), Sonnet for daily work, Opus for architecture and security. Haiku + Opus combo skips the middle tier entirely.

- **mgrep replaces grep** with ~50% token reduction on average. Less noise in context means more room for actual work.

- **Convert MCPs to CLI commands** for services that have robust CLIs (GitHub, Supabase, Vercel). Same functionality, freed up context window, lower token usage.

- **Background processes in tmux** keep streaming output out of Claude's context. Bring back only the relevant portion.

- **Modular codebases** cost less to work with. Files in the hundreds of lines instead of thousands mean Claude reads less, re reads less, and gets it right more often.
