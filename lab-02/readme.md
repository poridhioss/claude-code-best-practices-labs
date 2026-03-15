# Config and Rules

Claude Code reads markdown files at startup to understand how it should behave on your project. No JSON config, no YAML settings for behavior. Just markdown. These files are the single biggest lever you have for consistent output. Get them right and Claude follows your conventions from the first prompt. Get them wrong (or skip them entirely) and you'll spend half your session correcting code style, commit formats, and architectural decisions that should have been automatic.

Most people either dump everything into one massive CLAUDE.md or skip configuration entirely. Both are wrong. The configuration system has a deliberate hierarchy, and understanding how it layers is what makes the difference between Claude that "sort of knows" your project and Claude that actually follows your team's standards.


## The Configuration Hierarchy

Claude Code loads instructions from three levels. They all merge at runtime, they don't override each other.

| Level | Location | Scope | Shared via Git? |
|-------|----------|-------|-----------------|
| User | `~/.claude/CLAUDE.md` | Your personal preferences. Follows you across every project on your machine. | No |
| Project | `CLAUDE.md` or `.claude/CLAUDE.md` (repo root) | Team standards. Every developer who clones the repo gets these. | Yes |
| Directory | `packages/api/CLAUDE.md` | Package specific rules. Only load when you're working inside that directory. | Yes |

When you open Claude Code inside `packages/api/`, it loads all three. User, project root, and the directory level file. They stack.

> The critical mistake most teams make: putting team rules in `~/.claude/CLAUDE.md`. A new developer joins, clones the repo, and gets zero project context. They wonder why Claude generates completely different code than everyone else. Team standards go in the project root, always. Personal preferences go in user level.


## Setting Up the Practice Project

We'll build a Node.js monorepo with an API package and a React frontend package. Different conventions for each, which is exactly where the config hierarchy shines.

```bash
mkdir -p claude-config-lab/{packages/{api/src,web/src/components},tests}
cd claude-config-lab
git init
```

Create minimal source files so Claude has something to work with:

```bash
# API file
cat > packages/api/src/users.ts << 'EOF'
import { Router } from "express";
import { db } from "../db";

const router = Router();

router.get("/users", async (req, res) => {
  const users = await db.query("SELECT * FROM users");
  res.json({ data: users, meta: { total: users.length } });
});

router.post("/users", async (req, res) => {
  const { name, email } = req.body;
  const user = await db.query(
    "INSERT INTO users (name, email) VALUES ($1, $2)",
    [name, email]
  );
  res.status(201).json({ data: user });
});

export default router;
EOF
```

```bash
# React component
cat > packages/web/src/components/UserList.tsx << 'EOF'
import { useState, useEffect } from "react";

interface User {
  id: string;
  name: string;
  email: string;
}

export function UserList() {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    fetch("/api/users")
      .then(res => res.json())
      .then(data => setUsers(data.data));
  }, []);

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
EOF
```

```bash
# Test file
cat > tests/users.test.ts << 'EOF'
import { describe, it, expect } from "vitest";

describe("Users API", () => {
  it("should return a list of users", async () => {
    const res = await fetch("/api/users");
    const body = await res.json();
    expect(body.data).toBeDefined();
    expect(Array.isArray(body.data)).toBe(true);
  });
});
EOF
```

Your structure now looks like this:

```
claude-config-lab/
├── packages/
│   ├── api/src/users.ts
│   └── web/src/components/UserList.tsx
├── tests/users.test.ts
└── (config files we'll create next)
```


## Project Level CLAUDE.md

This is the file every team member gets when they clone the repo. It defines the baseline. Think of it as the onboarding document for Claude, the same way you'd onboard a new engineer on the team's conventions.

Create `CLAUDE.md` at the project root:

```markdown
# Project: User Management Platform

## Stack
- Backend: Express + TypeScript (packages/api/)
- Frontend: React + TypeScript (packages/web/)
- Database: PostgreSQL with parameterized queries
- Testing: Vitest

## Standards
- All code in TypeScript, no .js files
- Strict mode enabled
- No `any` types, use `unknown` and narrow
- No `console.log` in committed code, use the structured logger

## Git
- Conventional commits: `feat:`, `fix:`, `refactor:`, `test:`, `docs:`
- No force pushes to main
- PRs require passing CI before merge

## Testing
- Write tests before implementation (TDD)
- Tests live in `tests/` at the root
- Use `describe`/`it` blocks, no `test()` calls
- Minimum 80% coverage on new code
```

Notice it's concise. No prose explaining why, just what to do. Claude doesn't need motivation, it needs clear instructions. Every line here is something you'd otherwise repeat manually in prompts session after session.


### Practice: Verify Project Config

Open Claude Code inside the project:

```bash
claude
```

> Trust the folder when prompted.

<!-- screenshot: Claude Code startup with trust prompt -->

Ask Claude:

```
What are the project standards you're aware of?
```

Claude should reference conventional commits, TypeScript strict mode, no `any` types, Vitest. All from the CLAUDE.md you just wrote. If it doesn't mention these, check your file placement.

<!-- screenshot: Claude listing project standards from CLAUDE.md -->


## User Level CLAUDE.md

Your personal preferences that don't belong in the shared repo. These follow you across every project on your machine. A teammate won't have these, they'll have their own. That's the point.

```bash
mkdir -p ~/.claude
```

Create `~/.claude/CLAUDE.md`:

```markdown
# Personal Preferences

- Prefer explicit return types on functions
- Use single quotes for strings in TypeScript
- When I say "clean up" I mean: remove dead code, unused imports, and empty blocks
- Don't add comments unless the logic is genuinely non obvious
- When showing terminal commands, don't add explanatory comments after each line
```

Your preference for single quotes shouldn't be forced on the whole team through the project config. That's what this file is for.


## Directory Level CLAUDE.md

Different packages have different conventions. An Express API has rules that make no sense in a React frontend. Directory level files scope conventions to the package where they apply.

Create `packages/api/CLAUDE.md`:

```markdown
# API Package Standards

## Routing
- All routes return JSON:API format: `{ data: ..., meta: ... }`
- Use async/await, no .then() chains
- All database queries use parameterized inputs, never string concatenation

## Error Handling
- Wrap route handlers in asyncHandler()
- Return structured errors: `{ error: { code, message, status } }`
- Never expose stack traces in responses

## Validation
- Validate request bodies with Zod schemas before processing
- Schema files live next to their route: `users.schema.ts` beside `users.ts`
```

Create `packages/web/CLAUDE.md`:

```markdown
# Web Package Standards

## Components
- Functional components only, no class components
- Use named exports, not default exports
- Props interface goes directly above the component

## Styling
- Tailwind CSS utility classes
- No inline style objects
- Component specific styles via Tailwind @apply only when 5+ utilities repeat

## State
- React Query for server state
- useState/useReducer for local UI state
- No global state library unless explicitly discussed
```

When Claude is working inside `packages/api/`, it sees the API rules plus the root standards. When it's inside `packages/web/`, it sees the web rules plus the root standards. It never gets confused about which conventions apply where.


### Practice: Verify Directory Scoping

Open Claude inside the API package:

```bash
cd packages/api
claude
```

Ask:

```
What conventions should I follow when writing a new route in this package?
```

Claude should mention JSON:API format, asyncHandler, Zod schemas, parameterized queries from the directory level file. It should **also** know about conventional commits and TypeScript strict mode from the root CLAUDE.md. Both layers loaded.

<!-- screenshot: Claude listing API specific conventions alongside root standards -->

Now try the same from the web package:

```bash
cd packages/web
claude
```

```
What conventions should I follow for a new component?
```

Claude should reference functional components, named exports, Tailwind, React Query. And still know the root level standards. It should **not** mention JSON:API or Zod schemas. Those are API specific.

<!-- screenshot: Claude listing web conventions, no API rules visible -->


## Path Scoped Rules

Directory level CLAUDE.md files work, but they have a limitation. Some conventions apply to a **file type** spread across the whole codebase. Test files are the classic example. They might live in `tests/` at the root, or right next to the source files as `Button.test.tsx` beside `Button.tsx`. A directory scoped file can't capture "all test files everywhere."

This is what `.claude/rules/` with glob patterns solves. You create markdown files with YAML frontmatter that specifies which file paths the rules apply to. The rule only loads when Claude is editing or reading a file that matches the pattern. This saves context since Claude doesn't get testing rules when you're editing a component.

```bash
mkdir -p .claude/rules
```

Create `.claude/rules/testing.md`:

```markdown
---
paths:
  - "**/*.test.*"
  - "**/*.spec.*"
  - "tests/**/*"
---

# Testing Conventions

- Use `describe` blocks grouped by function or component under test
- Use `it` with behavior descriptions: `it("should return 404 when user not found")`
- One assertion per `it` block
- Use `beforeEach` for setup, not inline construction in every test
- Mock external services, never hit real endpoints in unit tests
- Name test files matching source: `users.ts` then `users.test.ts`
```

Create `.claude/rules/api-routes.md`:

```markdown
---
paths:
  - "packages/api/src/**/*.ts"
---

# API Route Conventions

- Every route file exports a single Router
- Route handlers are async and wrapped in asyncHandler()
- Input validation runs before any business logic
- Database access goes through repository functions, not raw queries in handlers
```

Create `.claude/rules/react-components.md`:

```markdown
---
paths:
  - "packages/web/src/**/*.tsx"
---

# React Component Conventions

- Props interface named {ComponentName}Props
- Destructure props in the function signature
- Hooks at the top of the function, before any conditionals
- Early return for loading/error states before the main render
```

The `paths` field uses glob patterns. `**/*.test.*` matches any test file at any depth in the project tree. This is more powerful than directory scoping because test conventions apply regardless of where the file sits.

> The certification exam specifically tests this: when conventions must apply to files spread across the codebase, path specific rules in `.claude/rules/` are the correct answer over subdirectory CLAUDE.md files.


### Practice: Verify Path Scoping

Open Claude at the project root:

```bash
cd claude-config-lab
claude
```

Ask:

```
I'm going to edit packages/api/src/users.ts. What conventions apply?
```

Claude should mention the API route conventions from `api-routes.md`. It should **not** mention React component conventions.

<!-- screenshot: Claude showing API route conventions for a .ts file -->

Now ask:

```
I'm going to edit tests/users.test.ts. What testing conventions apply?
```

Claude should reference the testing conventions from `testing.md`. `describe`/`it` blocks, one assertion per test, mocking external services. All loaded because the file matches `**/*.test.*`.

<!-- screenshot: Claude showing testing conventions for a test file -->


## Modular Config with @import

As your project grows, the root CLAUDE.md can become unwieldy. The `@import` syntax lets you reference external files, keeping the main file lean while pulling in detailed standards from separate owned files.

Update your root `CLAUDE.md` to use imports:

```markdown
# Project: User Management Platform

## Stack
- Backend: Express + TypeScript (packages/api/)
- Frontend: React + TypeScript (packages/web/)
- Database: PostgreSQL with parameterized queries
- Testing: Vitest

@import docs/standards/typescript.md
@import docs/standards/git-workflow.md
@import docs/standards/security.md
```

Now create those imported files:

```bash
mkdir -p docs/standards
```

`docs/standards/typescript.md`:

```markdown
# TypeScript Standards

- Strict mode enabled, no implicit any
- Use `unknown` over `any`, then narrow with type guards
- Prefer `interface` for object shapes, `type` for unions and intersections
- No non null assertions unless you add a comment explaining why
- Use const enum or string union types, not numeric enums
```

`docs/standards/git-workflow.md`:

```markdown
# Git Workflow

- Branch from `main` for features: `feat/short-description`
- Branch from `main` for fixes: `fix/short-description`
- Squash merge PRs, one commit per feature on main
- Delete branches after merge
- Tag releases: `v{major}.{minor}.{patch}`
```

`docs/standards/security.md`:

```markdown
# Security Standards

- No hardcoded secrets, use environment variables
- Validate all user input at the boundary
- Parameterized SQL queries only, never concatenate user input
- Sanitize HTML output to prevent XSS
- Rate limit public endpoints
- Log security events to structured logger
```

The benefit here is ownership. The security team updates `security.md`, the platform team updates `typescript.md`, and the root CLAUDE.md stays lean. The imports resolve at load time so Claude sees the full merged result.


### Practice: Verify Imports

```bash
claude
```

```
What security standards should I follow in this project?
```

Claude should reference parameterized queries, no hardcoded secrets, rate limiting. All from the imported `security.md`, even though those rules aren't written directly in the root CLAUDE.md.

<!-- screenshot: Claude referencing imported security standards -->


## Verifying What Loaded

The `/memory` command shows exactly which configuration files Claude has loaded in the current session.

```
/memory
```

You should see your user level CLAUDE.md, the project root CLAUDE.md (with imports resolved), and any matching `.claude/rules/` files based on the files you're currently working with.

<!-- screenshot: /memory output showing loaded config files -->

If Claude is ignoring a convention you know you wrote down, check `/memory` first. Nine times out of ten, the file isn't loading because of a path issue or a typo in the glob pattern.


## The Complete Structure

Here's what a well configured project looks like when everything is in place:

```
project/
├── CLAUDE.md                           # Team standards + @imports
├── .claude/
│   └── rules/
│       ├── testing.md                  # paths: ["**/*.test.*"]
│       ├── api-routes.md              # paths: ["packages/api/src/**/*.ts"]
│       └── react-components.md        # paths: ["packages/web/src/**/*.tsx"]
├── docs/
│   └── standards/
│       ├── typescript.md               # @imported
│       ├── git-workflow.md            # @imported
│       └── security.md                # @imported
├── packages/
│   ├── api/
│   │   ├── CLAUDE.md                   # API specific conventions
│   │   └── src/users.ts
│   └── web/
│       ├── CLAUDE.md                   # Frontend specific conventions
│       └── src/components/UserList.tsx
└── tests/
    └── users.test.ts

~/.claude/
└── CLAUDE.md                           # Personal prefs, not in repo
```

The hierarchy makes a specific tradeoff: more files to maintain, but each file is focused and loads only when relevant. You're not wasting context on React rules when writing SQL queries, and you're not forcing personal style preferences on the whole team.


## Key Takeaways

- **Project level CLAUDE.md** is for team standards. Committed to git, everyone gets them. This is your first line of defense against inconsistent output.

- **User level `~/.claude/CLAUDE.md`** is for personal preferences. Not shared. Never put team rules here, that's the number one configuration mistake.

- **Directory level CLAUDE.md** works for package specific conventions that only apply inside that directory tree.

- **`.claude/rules/` with paths frontmatter** is the more flexible option. Use it when conventions apply to a file type (tests, routes, components) spread across the codebase regardless of directory.

- **`@import`** keeps your root CLAUDE.md lean. Split standards into separate owned files as the project grows.

- **`/memory`** is your debugging tool. If Claude isn't following a convention, check what actually loaded before blaming the model.

- Rules **stack**. User plus project plus directory plus path matched rules all merge together. Understanding this hierarchy is what separates a well configured setup from one that fights you every session.
