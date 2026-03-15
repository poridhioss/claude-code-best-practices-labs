# Claude Code Labs

Setup → Daily Workflow → Architecture → Production

## Module 1 — Setup and Daily Workflow

| Lab | Title | Description |
|-----|-------|-------------|
| 01 | Introduction | Coding assistants, tool use, agentic loop, parallel reads, model tiers (Haiku/Sonnet/Opus), plan mode |
| 02 | Config and Rules | User + project CLAUDE.md, `.claude/rules/` with path scoped YAML frontmatter, `@import`, verify with `/memory` |
| 03 | Skills, Commands and Hooks | Project/user scoped slash commands, skill with `context: fork` + `allowed-tools`, hooks: `PreToolUse`, `PostToolUse`, `Stop`, `hookify` |
| 04 | Subagents and MCPs | `~/.claude/agents/` (planner, reviewer, resolver), MCP in `.mcp.json` with `${ENV_VAR}`, disable per project, install `mgrep`, measure context window |
| 05 | Editor, tmux and Parallel Basics | Split screen setup, tmux for long commands, `/fork`, `/rename`, `/compact`, `/rewind`, keyboard shortcuts |


## Module 2 — Context, Memory and Optimization

| Lab | Title | Description |
|-----|-------|-------------|
| 06 | Session Persistence | `SessionStart`/`PreCompact`/`Stop` memory hooks, session files at `~/.claude/sessions/`, resume from saved state next day |
| 07 | Strategic Compacting and System Prompt Injection | Disable auto compact, build compact suggester hook, manual `/compact` at phase transitions. Context aliases via `--system-prompt` (`dev.md`, `review.md`, `research.md`) |
| 08 | Continuous Learning and Token Optimization | Install continuous learning skill + Stop hook, `/learn` mid session, `/instinct-status`. Model routing (Haiku/Sonnet/Opus), `mgrep` vs grep token comparison, MCP to CLI conversion |


## Module 3 — Verification and Orchestration

| Lab | Title | Description |
|-----|-------|-------------|
| 09 | TDD and Verification Loops | RED → GREEN → IMPROVE cycle, checkpoint based + continuous evals, pass@k vs pass^k, benchmark with git worktree diff |
| 10 | Parallelization and Two Instance Kickoff | Git worktrees for parallel work, cascade method (3 to 4 max), two instance start: scaffold agent + research agent, merge outputs |
| 11 | Multi Agent Orchestration | Sequential phases (RESEARCH → PLAN → IMPLEMENT → REVIEW → VERIFY), iterative retrieval (query + objective, max 3 cycles), scoped tools per agent (4 to 5 max), `/clear` between phases |


## Module 4 — Agent SDK and Exam Domains

| Lab | Title | Description |
|-----|-------|-------------|
| 12 | Agentic Loop and Tool Design | Python agent with `stop_reason` loop (`tool_use`/`end_turn`), tool descriptions with input formats + edge cases + boundaries, split generic tools into purpose specific |
| 13 | Structured Output and Error Handling | `tool_use` with JSON schema (required/optional/nullable, enums), `tool_choice` (`auto`/`any`/forced), structured errors (`errorCategory`, `isRetryable`), retry with feedback loops |
| 14 | Prompt Engineering | Explicit criteria vs vague instructions (measure false positives), 2 to 4 few shot examples for ambiguous cases, multi pass review (independent instance vs self review) |
| 15 | Context Management and Reliability | "Case facts" block, trim verbose outputs, lost in the middle test, claim source mappings with provenance, escalation criteria + few shot, confidence calibration |
| 16 | CI/CD Integration | `-p` flag, `--output-format json` + `--json-schema`, CLAUDE.md as CI context, independent review instance, Message Batches API (`custom_id`, SLA calc) |


## Module 5 — Capstone

| Lab | Title | Description |
|-----|-------|-------------|
| 17 | Customer Support Agent | MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`), programmatic prerequisite gates, hook enforcement, structured errors, escalation criteria, multi concern decomposition, 80%+ FCR target |
| 18 | Multi Agent Research Pipeline | Coordinator + 3 subagents, scoped tools, parallel execution, iterative refinement (3 cycles), provenance tracking, error propagation, conflict annotation, coverage gap reporting |
| 19 | Real Project | Full stack on your own codebase: two instance kickoff → sequential phases → TDD → verification → session persistence → CI/CD |


## References

- [Shorthand Guide](https://x.com/affaanmustafa/status/2012378465664745795)
- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352)
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code)
- [Claude Certified Architect Exam Guide](https://everpath-course-content.s3-accelerate.amazonaws.com/instructor%2F8lsy243ftffjjy1cx9lm3o2bw%2Fpublic%2F1773274827%2FClaude+Certified+Architect+%E2%80%93+Foundations+Certification+Exam+Guide.pdf)
- [claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)