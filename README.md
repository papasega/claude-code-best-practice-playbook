# Claude Code — Best Practice Playbook

> **Audience :** Software engineers who want production-grade Claude Code setups, not toy demos.  
> **Scope :** Local workflows + GitHub CI/CD + token efficiency + git hygiene.  
> **Series :** Companion to [`claude-code-token-optimization.md`](./claude-code-token-optimization.md) — prompt customization & cost strategies.

> **Version :** Claude Code ≥ 2.1 · Last updated: 2026-03 · Verify with `claude --version`

---
![Claude Code Best Practice](./ndapli/ccbppb_psw.png)

## Table of Contents

1. [Mental Model](#1-mental-model)
2. [Project Configuration — settings.json](#2-project-configuration--settingsjson)
3. [User Configuration — global defaults](#3-user-configuration--global-defaults)
4. [CLAUDE.md — the right way](#4-claudemd--the-right-way)
5. [Context Management — the critical constraint](#5-context-management--the-critical-constraint)
6. [Hooks — deterministic guardrails](#6-hooks--deterministic-guardrails)
7. [Skills Architecture](#7-skills-architecture)
8. [Subagents — isolated context delegation](#8-subagents--isolated-context-delegation)
9. [GitHub Actions Integration](#9-github-actions-integration)
10. [Git History Policy](#10-git-history-policy)
11. [Code Quality Automation — REVIEW.md + hooks](#11-code-quality-automation--reviewmd--hooks)
12. [Command Reference](#12-command-reference)
13. [Usage Monitoring — Pro/Max Plans](#13-usage-monitoring--promax-plans)
14. [Token Optimization Table](#14-token-optimization-table)
15. [File Structure Reference](#15-file-structure-reference)
16. [Gotchas & Common Mistakes](#16-gotchas--common-mistakes)

---

> **TL;DR — 5 rules that cover 80% of the value:**
> 1. Keep `CLAUDE.md` under 150 lines — move everything else to skills
> 2. Set up hooks for git safety and large-file blocking — they fire deterministically, not "when Claude feels like it"
> 3. Use subagents (Haiku) for exploration, keep the main context clean for implementation
> 4. `/compact` every 30 min, `/clear` between unrelated tasks
> 5. Claude modifies files, you own `git log` — enforce with `permissions.deny` + `attribution: {"commit":"","pr":""}`

## 1. Mental Model

Claude Code is a **terminal-based agentic coding assistant**. It reads files, runs bash commands, edits code, and coordinates subagents — in a loop. Understanding how that loop works is the foundation of every optimization.

```
User prompt
    │
    ▼
Claude reasons (consumes tokens)
    │
    ▼
Claude calls a tool (Read / Bash / Edit / Write / ...)
    │
    ├─► PreToolUse hooks fire → can BLOCK or MODIFY the call
    │
    ▼
Tool executes
    │
    ├─► PostToolUse hooks fire → formatting, linting, logging
    │
    ▼
Result enters context window → Claude reasons again
    │
    └─► Loop until task complete or Stop event
```

**The context window is the shared resource everything competes for.** Every file Claude reads, every bash output, every conversation turn — all of it accumulates. Performance degrades as it fills. This is not a limitation to work around; it's the central design constraint to engineer for.

**What Claude Code CAN do :**
- Read, create, and modify any file in the working directory
- Execute bash commands (linting, testing, building, git operations)
- Spawn subagents with separate isolated context windows
- Post comments on GitHub issues and PRs via MCP tools
- Work autonomously across multi-file changes

**What you control :**
- Which tools are allowed or denied (via `permissions`)
- What runs before/after each tool (via `hooks`)
- What context Claude starts each session with (via `CLAUDE.md` + skills)
- Whether Claude appears in git history (via `attribution`)

---

## 2. Project Configuration — settings.json

**File : `.claude/settings.json`** — committed to git, shared with the team.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",

  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run lint:fix)",
      "Bash(npm run test *)",
      "Bash(npm run build)",
      "Bash(npm run type-check)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log --oneline *)",
      "Bash(git stash list)"
    ],
    "deny": [
      "Bash(git push *)",
      "Bash(git commit *)",
      "Bash(git checkout *)",
      "Bash(git merge *)",
      "Bash(git rebase *)",
      "Bash(git reset --hard *)",
      "Bash(git branch -d *)",
      "Bash(rm -rf *)",
      "Bash(sudo *)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Edit(./.env)",
      "Edit(./.env.*)",
      "Edit(./secrets/**)"
    ]
  },

  "model": "claude-sonnet-4-6",
  "effortLevel": "medium",

  "env": {
    "MAX_THINKING_TOKENS": "8000",
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1"
  },

  "attribution": {
    "commit": "",
    "pr": ""
  },

  "disabledMcpjsonServers": ["filesystem"],

  "respectGitignore": true,
  "includeGitInstructions": true,

  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/guard-git.sh"
          }
        ]
      },
      {
        "matcher": "Read",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/guard-large-read.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/auto-format.sh"
          }
        ]
      }
    ]
  }
}
```

**Key design decisions :**

| Decision | Why |
|---|---|
| `"model": "claude-sonnet-4-6"` | Team default — cheaper than Opus, covers 95% of tasks |
| `"effortLevel": "medium"` | Prevents runaway thinking tokens on routine tasks |
| `MAX_THINKING_TOKENS=8000` | Hard cap on thinking budget per turn |
| `attribution: {commit: "", pr: ""}` | Claude's `Co-Authored-By` line is suppressed from all commits/PRs |
| `disabledMcpjsonServers: ["filesystem"]` | Block raw filesystem MCP access — use Read/Write tools instead |
| `deny` on `git push/commit/merge` | Enforce human-only git history (see §10) |

**`.claude/settings.local.json`** — local overrides, never committed :

```json
{
  "effortLevel": "high",
  "model": "claude-opus-4-6"
}
```

Use this to temporarily upgrade model/effort on your machine without affecting team defaults. Claude Code auto-adds it to `.gitignore`.

---

## 3. User Configuration — global defaults

**File : `~/.claude/settings.json`** — applies to all your projects.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",

  "model": "claude-sonnet-4-6",
  "effortLevel": "medium",
  "autoUpdatesChannel": "stable",

  "permissions": {
    "allow": [
      "Bash(git diff *)",
      "Bash(git log --oneline *)",
      "Bash(git status)",
      "Read(~/.zshrc)",
      "Read(~/.bashrc)"
    ],
    "deny": [
      "Bash(sudo *)",
      "Bash(rm -rf *)"
    ]
  },

  "env": {
    "MAX_THINKING_TOKENS": "8000"
  }
}
```

**Scope precedence (highest → lowest) :**
```
Managed settings (org IT) > CLI flags > settings.local.json > settings.json (project) > settings.json (user)
```

When the same key appears in multiple scopes, the higher scope wins. **Exception : arrays (like `permissions.deny`) merge across all scopes** — they are concatenated and deduplicated, not overwritten.

---

## 4. CLAUDE.md — the right way

CLAUDE.md is loaded into context at the start of every session — every line costs tokens on every turn. Treat it like code: ruthless brevity, zero redundancy.

**Rules :**
- **Global** (`~/.claude/CLAUDE.md`) : ≤ 200 lines — personal style, universal conventions
- **Project** (`CLAUDE.md` or `.claude/CLAUDE.md`) : ≤ 150 lines — project-specific only
- **Subdirectory** (e.g. `src/api/CLAUDE.md`) : ≤ 50 lines — loaded only when Claude reads files in that directory
- **Never duplicate** rules that already exist at a higher level
- **Use `<!-- comment -->`** for maintainer notes — stripped from context automatically
- **Move workflows to skills** — they load on-demand, not at startup

**Example project CLAUDE.md (TypeScript/Express) :**

```markdown
<!-- Compact instructions: preserve modified files, test commands, active task. Discard exploration history. -->

# Project: [Name]

## Git policy
- ❌ NEVER run git commit, git push, git checkout, git merge, git rebase
- ✅ Run git diff / git status freely
- ✅ Modify files — developer reviews and commits

## Architecture
- `src/api/`  — Express routers (one file per resource)
- `src/auth/` — JWT middleware
- `src/db/`   — Prisma models + migrations
- `tests/`    — Jest, co-located in `__tests__/`

## Commands
- `npm run dev`        — start dev server
- `npm run test`       — run full test suite
- `npm run lint:fix`   — lint + autofix
- `npm run type-check` — TypeScript strict check
- `npm run build`      — compile to dist/

## Code standards
- `async/await` everywhere — no callbacks, no bare Promises
- `AppError` for all thrown errors — never `throw new Error()`
- No `any` without explicit `// eslint-disable-next-line` comment
- Tests required for all new public functions

## On-demand skills
- `/project-architecture` — full codebase map before large analysis
- `/code-review`          — PR review checklist
- `/git-safe`             — diff workflow after Claude modifies files
```

**What NOT to put in CLAUDE.md :**
- Full API documentation → put in a skill's reference files (loaded only when needed)
- Deployment runbooks → put in a skill
- Long lists of edge cases → put in a skill
- Code examples longer than 5 lines → put in a skill
- Content duplicated from `~/.claude/CLAUDE.md`

---

## 5. Context Management — the critical constraint

The context window holds everything: conversation history, file reads, bash output, CLAUDE.md, skills loaded, subagent responses. It fills fast. LLM performance degrades as it fills — Claude "forgets" earlier instructions, makes more mistakes.

**The auto-compact threshold is ~75% by default.** Do not wait for it. Manage context proactively.

### When to use each command

| Situation | Command | What it does |
|---|---|---|
| Switching to unrelated task | `/clear` (or `/reset` or `/new`) | Wipes all history. File edits persist. |
| Long session, still on same task | `/compact [instructions]` | Replaces history with a dense summary |
| Want to try a different approach | `/fork` | Branch conversation, experiment, resume the original |
| Quick lookup, don't want it in context | `/btw [question]` | Answer appears in overlay, never enters history |
| Session getting foggy | `Esc + Esc` | Open rewind menu — roll back conversation or code |
| Resume yesterday's session | `claude -c` | Continue the most recent session in current dir |

**Always pair `/rename` with `/clear` :**
```
/rename auth-refactor-session-2025-03
/clear
```
Named sessions appear in history and are findable weeks later.

**Custom compaction instructions in CLAUDE.md :**
```markdown
<!-- Compact instructions: preserve modified file list, test commands, active task. Discard: exploration history, verbose tool output. -->
```

**Compaction is triggered by :**
- You running `/compact`
- Auto-compact at ~75% fill
- You can override the threshold : `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=60`

### The subagent pattern for research

When Claude needs to explore a codebase before implementing, every file it reads bloats the main context. Instead:

```
Use a subagent to investigate how our authentication middleware handles token refresh.
Return a summary of: relevant file paths, current behavior, and any edge cases found.
Keep the summary under 200 words.
```

The subagent runs in a **separate, isolated context window**. It explores, then returns a compact report. Your main context stays clean for implementation.

---

## 6. Hooks — deterministic guardrails

Hooks are shell scripts (or HTTP endpoints) that run at specific points in Claude Code's lifecycle. Unlike CLAUDE.md instructions — which are requests Claude interprets — **hooks execute deterministically, every time**.

**Exit code contract :**
- `exit 0` → allow (optionally with `{"decision": "allow"}` on stdout)
- `exit 1` or `exit 2` → block (Claude sees the reason from stderr/stdout)
- `exit 0` + JSON with `decision: block` on stdout → block with explanation

**Hook input :** all hooks receive a JSON object on `stdin`. For tool events, `tool_name` and `tool_input` are the key fields.

**Environment variables available in hooks :**
- `$CLAUDE_PROJECT_DIR` — absolute path to project root
- `$CLAUDE_TOOL_INPUT_FILE_PATH` — file path for Edit/Write hooks (shortcut vs parsing stdin)
- `$CLAUDE_SESSION_ID` — current session ID

Two implementation styles are valid and can coexist:
- **Inline JSON command** (used in settings.json §1): best for simple one-liner logic, easier to version with the project config
- **External script** (used in hooks below): best for logic > 5 lines, easier to test independently with `echo '{...}' | bash hook.sh`

Both styles receive JSON on stdin and communicate via exit codes.
Choose inline for simplicity, external scripts for maintainability.

### Hook 1 : Block destructive git operations

**`.claude/hooks/guard-git.sh`** — make executable with `chmod +x`

```bash
#!/bin/bash
# Block git write operations — developer commits manually
set -euo pipefail

INPUT=$(cat)
CMD=$(echo "$INPUT" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('tool_input',{}).get('command',''))" 2>/dev/null || echo "")

if echo "$CMD" | grep -qE '^git\s+(push|commit|checkout|merge|rebase|reset\s+--hard|branch\s+-[dD])'; then
  python3 -c "
import json, sys
cmd = '''$CMD'''
print(json.dumps({
  'decision': 'block',
  'reason': f'Git write operation blocked by project policy: [{cmd}]. '
            f'Review changes with: git diff, then commit manually.'
}))
"
  exit 0
fi

exit 0
```

### Hook 2 : Block accidental large-file reads (token waste)

**`.claude/hooks/guard-large-read.sh`**

```bash
#!/bin/bash
# Block Read tool on files > 300 lines — enforce grep/sed extraction
set -euo pipefail

INPUT=$(cat)
FILE=$(echo "$INPUT" | python3 -c "
import json, sys
print(json.load(sys.stdin).get('tool_input', {}).get('file_path', ''))
" 2>/dev/null || echo "")

# Early exit: no file path or file doesn't exist
if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
  exit 0
fi

LINES=$(wc -l < "$FILE" 2>/dev/null || echo 0)
if [ "$LINES" -gt 300 ]; then
  python3 -c "
import json
print(json.dumps({
  'decision': 'block',
  'reason': (
    '$FILE has $LINES lines (~' + str($LINES * 5) + ' tokens). '
    'Use targeted extraction: '
    'grep -n \"pattern\" $FILE | head -20 '
    'or: sed -n \"/^def target/,/^def /p\" $FILE | head -50'
  )
}))
"
  exit 0
fi

exit 0
```

### Hook 3 : Auto-format on file write (PostToolUse)

**`.claude/hooks/auto-format.sh`**

```bash
#!/bin/bash
# Auto-run formatter after Claude edits a file
# Uses $CLAUDE_TOOL_INPUT_FILE_PATH env var (no stdin parsing needed for PostToolUse)
set -euo pipefail

FILE="${CLAUDE_TOOL_INPUT_FILE_PATH:-}"

if [ -z "$FILE" ] || [ ! -f "$FILE" ]; then
  exit 0
fi

# TypeScript / JavaScript
if echo "$FILE" | grep -qE '\.(ts|tsx|js|jsx)$'; then
  if command -v npx &>/dev/null; then
    npx prettier --write "$FILE" 2>/dev/null || true
  fi
fi

# Python
if echo "$FILE" | grep -qE '\.py$'; then
  if command -v ruff &>/dev/null; then
    ruff format "$FILE" 2>/dev/null || true
  fi
fi

# Go
if echo "$FILE" | grep -qE '\.go$'; then
  if command -v gofmt &>/dev/null; then
    gofmt -w "$FILE" 2>/dev/null || true
  fi
fi

exit 0
```

### Hook 4 : Gate Stop — run tests before Claude declares done

**In `.claude/settings.json` hooks section :**

```json
"Stop": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "INPUT=$(cat); [ \"$(echo $INPUT | python3 -c 'import json,sys; print(json.load(sys.stdin).get(\\\"stop_hook_active\\\", False))')\" = 'True' ] && exit 0; npm run test 2>&1 | tail -5 || exit 2"
      }
    ]
  }
]
```

> **Important :** Always check `stop_hook_active` in Stop hooks to prevent infinite loops. When Claude's Stop hook causes Claude to keep working, the field is set to `true` on subsequent invocations.

### Hook 5 : SessionStart — inject git context

```json
"SessionStart": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "echo \"{\\\"additionalContext\\\": \\\"Branch: $(git branch --show-current 2>/dev/null || echo 'no-git') | Last commit: $(git log --oneline -1 2>/dev/null || echo 'none')\\\"}\""
      }
    ]
  }
]
```

This injects the current git branch and last commit into every session start — Claude knows where it is without reading git manually.

---

## 7. Skills Architecture

Skills are on-demand context: **zero startup cost, full content loaded only when invoked**. Move everything workflow-specific from CLAUDE.md into skills.

**At startup, Claude loads only :** `name` + `description` from each skill (~30–50 tokens total per skill).  
**When a skill is triggered :** the full `SKILL.md` content loads into context.

### Skill: project-architecture

**`.claude/skills/project-architecture/SKILL.md`**

```markdown
---
name: project-architecture
description: >
  Load full codebase map before large analysis tasks.
  Triggers: "architecture", "project overview", "how is X organized",
  before any analysis spanning multiple modules.
---

# Project Architecture

## Module map
- `src/api/`    — Express routers (one file per HTTP resource)
- `src/auth/`   — JWT validation middleware + refresh logic
- `src/db/`     — Prisma schema, migrations, seed scripts
- `src/utils/`  — Pure utility functions (no side effects)
- `tests/`      — Jest test suite, co-located `__tests__/` dirs

## Key entry points (use grep, not full file reads)
```bash
# Find all route definitions
grep -rn "router\.\(get\|post\|put\|delete\)" src/api/ --include="*.ts"

# Find all exported functions in a module
grep -n "^export" src/auth/index.ts

# Find where an entity is used
grep -rn "UserRepository" src/ --include="*.ts" -l
```

## Dependency graph shortcut
```bash
# Direct dependencies of a file (TypeScript imports)
grep -n "^import" src/auth/middleware.ts

# Who imports a given module
grep -rn "from.*auth/middleware" src/ --include="*.ts"
```
```

### Skill: code-review

**`.claude/skills/code-review/SKILL.md`**

```markdown
---
name: code-review
description: >
  Structured PR review checklist. Triggers: "review this PR",
  "analyze this code", "what's wrong with", "check this diff".
---

# Code Review Process

## Step 1: Scope the diff (never read full files)
```bash
git diff HEAD~1 --stat              # What changed and how much
git diff HEAD~1 -- path/to/file.ts  # Targeted diff for one file
```

## Step 2: Review checklist

### Correctness
- [ ] Logic handles all edge cases (null, empty array, zero, negative)
- [ ] No off-by-one errors in loops or pagination
- [ ] Async errors are caught (unhandled rejections = silent prod bugs)

### Type safety (TypeScript)
- [ ] No implicit `any` — each should have `// eslint-disable-next-line` justification
- [ ] Return types explicit on all public functions
- [ ] Discriminated unions used for complex state

### Error handling
- [ ] HTTP endpoints return correct status codes (not always 200 + error body)
- [ ] No empty `catch` blocks
- [ ] Error messages don't leak internal stack traces to clients

### Security
- [ ] No secrets hardcoded (even in test fixtures)
- [ ] SQL queries use parameterized statements — no string interpolation
- [ ] User input validated with Zod before hitting business logic

### Tests
- [ ] New public functions have unit tests
- [ ] Edge cases covered: null input, empty array, error path
- [ ] No `it.only` or `xit` left in codebase

## Step 3: Output format for GitHub comment
```markdown
## Code Review

### ✅ Strengths
- [what was done well]

### ⚠️ Issues
| Severity | File:line | Issue |
|----------|-----------|-------|
| 🔴 critical | auth.ts:42 | SQL injection via string concat |
| 🟡 warning  | user.ts:87 | Missing error handling on DB call |
| 🔵 info     | types.ts:12 | Could use discriminated union |

### 💡 Suggestions (non-blocking)
- [optional improvements]
```
```

### Skill: test-writing

**`.claude/skills/test-writing/SKILL.md`**

```markdown
---
name: test-writing
description: >
  Write comprehensive Jest tests. Triggers: "write tests for", "add test coverage",
  "test this function", before any new public API is added.
---

# Test Writing — Jest + TypeScript

## Structure: Arrange / Act / Assert
```typescript
describe('ModuleName', () => {
  describe('functionName', () => {
    it('should [expected behavior] when [condition]', () => {
      // Arrange — set up test data and mocks
      const input = buildTestUser({ email: 'test@example.com' });
      
      // Act — call the function under test
      const result = validateEmail(input.email);
      
      // Assert — verify outcome
      expect(result).toBe(true);
    });
  });
});
```

## Test matrix — cover ALL of these
| Case | Description |
|------|-------------|
| Happy path | Normal input, expected output |
| Empty/null | `null`, `undefined`, `''`, `[]`, `{}` |
| Boundary | Max/min values, exact thresholds |
| Error path | Invalid input throws expected error |
| Async errors | Rejected promises are caught |

## Mock patterns
```typescript
// Mock a module
jest.mock('../db/userRepository');
const mockGetUser = jest.mocked(getUserById);
mockGetUser.mockResolvedValue(buildTestUser());

// Spy on a method
const consoleSpy = jest.spyOn(console, 'error').mockImplementation();

// Cleanup
afterEach(() => jest.clearAllMocks());
```

## Test factories (prefer over inline objects)
```typescript
// tests/factories/user.factory.ts
export const buildTestUser = (overrides: Partial<User> = {}): User => ({
  id: crypto.randomUUID(),
  email: 'default@test.com',
  createdAt: new Date('2025-01-01'),
  ...overrides,
});
```
```

### Skill: git-safe

**`.claude/skills/git-safe/SKILL.md`**

```markdown
---
name: git-safe
description: >
  Review Claude's file changes before committing. Triggers: "show changes",
  "what did you modify", "stage these", "ready to commit".
---

# Git-Safe Workflow

## After Claude modifies files
```bash
git status                  # What changed
git diff                    # Full diff of all changes
git diff path/to/file.ts    # Diff for one file
git add -p                  # Interactively stage hunks (recommended)
```

## Verify Claude didn't touch git history
```bash
git log --oneline -5        # Should show only your commits
git status                  # Modified files, NOT new commits
```

## Revert if needed
```bash
git checkout -- .           # Revert all unstaged changes
git stash                   # Stash changes temporarily
```

## Write the commit message yourself
```bash
git add -p                          # Review each hunk
git commit -m "feat(auth): ..."     # Conventional commits format
```
`attribution: {commit: "", pr: ""}` in settings.json ensures
no `Co-Authored-By: Claude` line appears — even if you forget.
```

---

## 8. Subagents — isolated context delegation

Subagents are separate Claude Code instances with their own context window. When a subagent finishes, it returns a summary to the parent — not the full exploration history.

**Use subagents for :**
- Codebase research/exploration before implementing
- Parallel independent tasks (test one module while refactoring another)
- Specialized review tasks that shouldn't pollute the main session

### Subagent: code-explorer (Haiku — 10× cheaper)

**`.claude/agents/code-explorer.md`**

```markdown
---
name: code-explorer
description: >
  Fast codebase exploration. Use for: finding where X is implemented,
  mapping dependencies, understanding module behavior before editing.
  Returns a compact summary — does NOT read full files when unnecessary.
model: haiku
effort: low
maxTurns: 10
tools: Read, Bash, Glob, Grep
---

You are a fast, efficient code navigator. Explore, then summarize compactly.

## Exploration rules
- Use `grep -n "pattern" file` instead of reading entire files
- Use `grep -rn "symbol" src/ -l` to find files before reading them
- Stop when you have enough to answer — do not explore exhaustively
- Never read files > 200 lines in full without justification

## Response format (ALWAYS)
Return a JSON block:
```json
{
  "relevant_files": ["src/auth/middleware.ts:15-45"],
  "key_findings": ["JWT validation happens at line 23", "No refresh token logic found"],
  "recommended_approach": "Add refresh endpoint in src/auth/ alongside existing middleware",
  "files_read": 3,
  "grep_calls": 5
}
```
```

### Subagent: pr-reviewer (Sonnet — thorough review)

**`.claude/agents/pr-reviewer.md`**

```markdown
---
name: pr-reviewer
description: >
  Thorough PR review. Use when a developer asks for a full review
  of their implementation. Runs the full code-review checklist,
  returns structured feedback for a GitHub comment.
model: sonnet
effort: medium
maxTurns: 15
tools: Read, Bash, Grep, Glob
---

You are a senior software engineer doing a thorough code review.

## Process
1. Run `git diff HEAD~1 --stat` — scope the change
2. For each changed file: run targeted diff, not full file read
3. Apply the code-review checklist (from project skills)
4. Produce a structured review comment

## Output format
Return a complete markdown block ready to paste as a GitHub PR comment.
Flag: 🔴 critical (must fix), 🟡 warning (should fix), 🔵 info (optional).
```

---

## 9. GitHub Actions Integration

The official action is `anthropics/claude-code-action@v1`. Control what Claude does through `--allowedTools` in `claude_args`. There are **no** `read_only`, `no_commits`, or `mode: "review-only"` parameters in the v1 API.

### 9.1 PR Code Review (comment-only)

**`.github/workflows/claude-review.yml`**

```yaml
name: Claude PR Review

on:
  pull_request:
    types: [opened, synchronize]
  issue_comment:
    types: [created]

# Minimal permissions — grant only what Claude actually needs
permissions:
  contents: read          # Read source code
  pull-requests: write    # Post PR comments
  issues: write           # Post issue comments
  id-token: write         # GitHub App OIDC auth

jobs:
  claude-review:
    if: |
      github.event_name == 'pull_request' ||
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude'))

    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for accurate diffs

      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Review this PR and post a structured comment with:
            1. Correctness issues (bugs, edge cases missed)
            2. Security concerns (injection, auth bypass, secret exposure)
            3. Test coverage gaps
            4. Performance red flags

            Be concise. Flag only real issues — not style preferences.
            DO NOT commit or push anything.

          # Only these tools are available — no Write, no git push/commit
          claude_args: |
            --max-turns 5
            --model claude-sonnet-4-6
            --allowedTools "Read,Grep,Glob,Bash(git diff *),Bash(git log --oneline *),Bash(npm run test *),Bash(npm run type-check),mcp__github__create_review_comment,mcp__github__create_issue_comment"
```

### 9.2 Automated security scan (headless, on push)

```yaml
name: Claude Security Scan

on:
  push:
    branches: [main, develop]

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  security-scan:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            Scan the diff from the last push for security issues only:
            - Hardcoded secrets or API keys
            - SQL/command injection vectors
            - Missing input validation on new endpoints
            - Auth/authz bypasses

            Report only confirmed issues, not hypothetical risks.
            Format: one bullet per issue with file:line reference.

          claude_args: |
            --max-turns 3
            --model claude-haiku-4-5-20251001
            --allowedTools "Read,Grep,Glob,Bash(git diff *),mcp__github__create_issue_comment"
```

### 9.3 Using Custom GitHub App (recommended for teams)

```yaml
- name: Generate token from custom app
  id: app-token
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ secrets.APP_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}

- uses: anthropics/claude-code-action@v1
  with:
    # Use app token instead of direct API key
    # Comments appear as your app's bot name, not claude[bot]
    github_token: ${{ steps.app-token.outputs.token }}
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    claude_args: |
      --allowedTools "Read,Grep,Glob,mcp__github__create_review_comment"
```

> **Quick setup :** Run `claude /install-github-app` in your terminal. It walks through GitHub App creation and injects the workflow YAML automatically.

---

## 10. Git History Policy

The goal: Claude's work appears in the codebase (files modified) but **never** in `git log`, commit author fields, or `Co-Authored-By` trailers.

### Two-layer enforcement

**Layer 1 — settings.json deny rules :** Prevent Claude from running git write commands (§2).

**Layer 2 — attribution suppression :** Even if a commit is made with Claude's help, the trailer is suppressed:

```json
"attribution": {
  "commit": "",
  "pr": ""
}
```

With empty strings, Claude Code writes no `Co-Authored-By: Claude` line in commits or PR descriptions. `git log --all` will show only human authors.

### Verify the policy is working

```bash
# After a session where Claude modified files
git log --oneline -5          # Should show only your commits
git log --format="%an %ae" -5 # Should show only your email
git show HEAD | grep -i claude # Should return nothing
```

### The developer workflow with Claude

```bash
# 1. Let Claude modify files
# "Implement the user registration endpoint"

# 2. Review everything Claude did
git diff
git diff --stat

# 3. Interactively stage what you want
git add -p

# 4. Write your own commit message
git commit -m "feat(auth): add user registration endpoint"

# 5. Push — Claude never appears
git push
```

---

## 11. Code Quality Automation — REVIEW.md + hooks

**`REVIEW.md`** is a project-level contract for what "good code" means. It's loaded into context via the `/code-review` skill, not at startup. Keep it as specific as possible — vague guidance wastes tokens without improving output.

**File : `REVIEW.md`**

```markdown
# Code Review Contract — [Project Name]

## Must-fix (🔴 critical)
- Missing `try/catch` on async DB operations in Express handlers
- Any `eval()` or `new Function()` call
- SQL string interpolation (use parameterized queries)
- `console.log` left in production code (use structured logger)
- `.env` values referenced without validation at startup

## Should-fix (🟡 warning)
- Functions > 50 lines (extract, don't just wrap)
- Tests missing for new public functions
- Middleware added without corresponding test for unauthorized access
- Return type missing on exported async functions

## Ignore (don't comment on)
- Import order (Prettier handles this)
- Whitespace and indentation (Prettier handles this)
- Variable naming style in test files
- Files under `src/gen/` (generated — never hand-edit)

## Security checklist
- All user input passes through Zod schema before use
- HTTP-only cookies for auth tokens
- Rate limiting on auth endpoints
- No stack traces in 4xx/5xx responses to clients

## Performance checklist
- Paginate all list endpoints (no `SELECT *` without LIMIT)
- Avoid N+1: use `include` in Prisma or join in raw SQL
- Cache expensive computations with Redis TTL
```

### Auto-format hook (PostToolUse) — already in §6

The `auto-format.sh` hook fires on every `Write|Edit|MultiEdit` event. For TypeScript projects, this means Prettier runs on every file Claude touches. No more "format on save" step — it's automatic.

### Type-check hook (Stop) — gate the session end

```json
"Stop": [
  {
    "hooks": [
      {
        "type": "command",
        "command": "INPUT=$(cat); ACTIVE=$(echo \"$INPUT\" | python3 -c 'import json,sys; print(json.load(sys.stdin).get(\"stop_hook_active\", False))'); [ \"$ACTIVE\" = 'True' ] && exit 0; npx tsc --noEmit 2>&1 | tail -10 || exit 2"
      }
    ]
  }
]
```

Claude cannot declare a task done if TypeScript type-check fails. `exit 2` causes Claude to see the error and keep working.

---

## 12. Command Reference

### Session management

| Command | What it does | When to use |
|---|---|---|
| `/clear` | Wipe all history. File edits persist. | Switching to unrelated task |
| `/compact [instructions]` | Summarize history into dense context | Context > 70%, still on same task |
| `/rename [name]` | Name the current session | Before `/clear` so you can resume it |
| `/fork` | Branch conversation for experimentation | Trying a risky approach |
| `/rewind` (or `Esc + Esc`) | Roll back conversation or code state | Wrong direction, need to backtrack |
| `/export` | Export session as plain text | Postmortems, sharing with teammates |
| `claude -c` | Resume most recent session | Coming back to yesterday's work |
| `claude -r [session-id]` | Resume specific session | Coming back to a named session |

### Context and cost visibility

| Command | What it shows |
|---|---|
| `/context` | Context window usage (%) |
| `/cost` | Token usage + cost this session (API users) |
| `/stats` | Usage breakdown (Pro/Max users) |
| `/status` | Active settings sources, MCP servers, model |
| `/doctor` | Installation health check |

### Model and effort

| Command | Effect |
|---|---|
| `/model` | Open model picker |
| `/effort low` | ~2k thinking tokens — fast, cheap, routine tasks |
| `/effort medium` | ~8k thinking tokens — default, balanced |
| `/effort high` | ~16k thinking tokens — complex debugging, architecture |
| `ultrathink` in prompt | Triggers high effort for that one turn only |

### Workflow

| Command | What it does |
|---|---|
| `/plan` (or `Shift+Tab`) | Toggle plan mode — Claude proposes, you approve each step |
| `/btw [question]` | Overlay answer — never enters conversation history |
| `/compact Focus on X` | Compact while explicitly preserving X |
| `/add-dir [path]` | Add additional working directory to session |
| `/install-github-app` | Set up Claude GitHub Actions integration |

### Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Shift+Tab` | Toggle plan mode |
| `Esc + Esc` | Open rewind menu |
| `Ctrl+C` | Cancel current operation |
| `Ctrl+R` | Search session history |
| `# [text]` | Quick memory note (saved to auto-memory) |
| `@path` | Reference a file in your message |
| `!command` | Run bash command inline in your prompt |

---

## 13. Usage Monitoring — Pro/Max Plans

Pro and Max subscribers pay a flat monthly fee — `/cost` reports token counts but **not billable dollars**. What matters is **weekly allocation**: a shared pool across claude.ai and Claude Code that resets every 7 days, with a 5-hour rolling window per session.

### Built-in commands (use these first)

| Command | What it shows | Plan |
|---|---|---|
| `/stats` | Usage patterns over time | Pro / Max |
| `/usage` | Reset timing + remaining allocation | Pro / Max |
| `/context` | Context window % used in current session | All |
| `/status` | Active model, settings, MCP servers | All |
| Settings → Usage (claude.ai) | Weekly progress bar with % consumed | Pro / Max |

> `/cost` is for **API pay-as-you-go users only** — it shows token counts but is irrelevant for billing on Pro/Max.

### Understanding your real token footprint

Claude Code logs every session as JSONL files in `~/.claude/projects/`. The raw token count is misleading because **cache reads are billed at 0.1×** while output tokens cost 5× more than input. Use a cost-weighted view to understand your actual budget consumption.

**Example output from the `usage-monitor` skill :**

```
========================================================
  CLAUDE CODE USAGE — Pro Plan (token breakdown)
  2026-03-28 11:06 UTC
========================================================
  TODAY  (2 sessions)
    Real input    :           67
    Cache create  :      139,614   (1.25x)
    Cache reads   :    1,265,080   (0.1x — cheap)
    Output        :       14,037   (5x)
    Cost-weighted :      371,278 units

  LAST 7 DAYS  (4 sessions)
    Real input    :       35,717
    Cache create  :    4,991,214   (1.25x)
    Cache reads   :   44,034,128   (0.1x — cheap)
    Output        :      355,421   (5x)
    Raw total     :   49,416,480   (inflated by cache reads)
    Cost-weighted :   12,455,252 units

  WEEKLY BUDGET (cost-weighted heuristic)
    [████████████░░░░░░░░] 62.3%
  >> >50% — monitor closely, prefer /compact
========================================================
```

The key insight from this output: **raw token total (49M) is dominated by cache reads** billed at 0.1×. The cost-weighted figure (12.4M units) is what actually counts toward your weekly limit. Never panic at the raw number.

### The usage-monitor skill

> **Location :** This skill lives in `~/.claude/skills/usage-monitor/SKILL.md` (your **global** home directory, not the project). It's available across all projects.

Create it once with this prompt in Claude Code, then invoke with `/usage-monitor` at session start or whenever you want a budget check:

```
Create a skill at ~/.claude/skills/usage-monitor/SKILL.md that tracks
my weekly Claude Code usage as a Pro subscriber.

Skill frontmatter:
  name: usage-monitor
  description: >
    Track weekly Claude Code usage limits for Pro plan.
    Triggers: "how much have I used", "check my limits", "weekly usage",
    "am I close to the limit", "usage reset", at the start of any session
    where budget awareness matters.

The skill must include an inline bash script (no external file) that:
1. Parses ~/.claude/projects/**/*.jsonl to extract usage events
2. Computes for TODAY and LAST 7 DAYS:
   - input_tokens, cache_creation_input_tokens (×1.25),
     cache_read_input_tokens (×0.1), output_tokens (×5)
   - cost_weighted = input + cache_create×1.25 + cache_read×0.1 + output×5
   - session count
3. Estimates weekly budget % using 20,000,000 cost-weighted units as
   the Pro plan heuristic ceiling
4. Renders the output in the exact format shown in the skill examples section
5. Prints a recommendation based on threshold:
   - <50%  → "on track — normal workflow"
   - 50–75% → "monitor closely, prefer /compact"
   - 75–90% → "budget tight — switch to Haiku for exploration"
   - >90%  → "critical — /clear aggressively, Haiku only"

Also include in the skill:
- The 5-hour rolling window explanation and how to pace sessions
- Model switching rule: drop to Haiku mid-week when >75%
- The ccusage commands (npx ccusage, npx ccusage daily)
- A weekly planning table template

After creating the skill, run the bash script immediately to show current state.
```

### Community tools (no install required)

```bash
# Full dashboard — reads local JSONL files, nothing sent externally
npx ccusage

# By day (last 7 days)
npx ccusage daily

# By session
npx ccusage --session

# Real-time monitor (separate terminal window while Claude Code runs)
pip install claude-monitor --quiet && claude-monitor --plan pro
# or: uvx claude-monitor --plan pro
```

### Weekly budget strategy

The weekly limit resets every 7 days. Plan intensive work accordingly:

| Weekly usage | Recommended action |
|---|---|
| < 50% | Normal workflow — all models available |
| 50–75% | Prefer `/compact` often, avoid long explorations without subagents |
| 75–90% | Switch to Haiku for all exploration subagents, Sonnet for core tasks only |
| > 90% | `/clear` aggressively between tasks, Haiku only, defer non-urgent work |
| Limit hit | Switch to API pay-as-you-go via `claude logout` → re-login with Console account |

**5-hour rolling window :** Each session window starts at your first prompt and resets 5 hours later. Heavy multi-file agentic work burns this faster. Plan intensive sessions (big refactors, architecture work) at the **start** of a fresh window, not the end.

**Model cost weight (relative to Sonnet) :**

| Model | Cost weight | Use for |
|---|---|---|
| Haiku | ~0.1× | All exploration subagents, simple grep/read tasks |
| Sonnet 4.6 | 1× | Default — 95% of tasks |
| Opus 4.6 | ~1.7× | Architecture decisions, complex multi-step reasoning only |

Switching from Sonnet to Haiku for exploration subagents when above 75% weekly usage extends your budget significantly without impacting output quality on the core tasks.

---

## 14. Token Optimization Table

| Technique | Mechanism | Estimated impact |
|---|---|---|
| CLAUDE.md ≤ 150 lines | Less context loaded at startup | −2k–8k tokens/session |
| Skills on-demand | Skill content loads only when invoked | −1k–5k tokens/session |
| Large-file hook (>300 lines) | Intercept before context pollution | −10k–50k tokens/session |
| `MAX_THINKING_TOKENS=8000` | Cap thinking budget per turn | −30–50% on thinking |
| `/compact` every 30 min | Eliminate accumulated noise | Variable, often largest gain |
| `/clear` between tasks | Zero carryover between unrelated work | ~100% context freed |
| `/btw` for quick lookups | Never enters conversation history | −500–2k per lookup |
| Haiku for subagents | 10× cheaper model for exploration | −80–90% on research calls |
| `--allowedTools` in CI | Narrow tool surface in GitHub Actions | −40–60% in CI runs |
| `effortLevel: medium` default | Prevents over-thinking on routine tasks | Baseline efficiency |
| SessionStart hook with git context | Avoid manual `git status` at start | −200–500 tokens/session |

**Monitoring :** Run `/context` every 20–30 min. At 70%+, compact before degradation begins. Run `/usage-monitor` at session start when on Pro/Max to check weekly budget.

---

## 15. File Structure Reference

```
project/
├── .claude/
│   ├── settings.json              # Team-shared config (committed)
│   ├── settings.local.json        # Per-machine overrides (gitignored)
│   ├── hooks/
│   │   ├── guard-git.sh           # Block git write ops
│   │   ├── guard-large-read.sh    # Block reads > 300 lines
│   │   └── auto-format.sh        # Auto-format on write/edit
│   ├── skills/
│   │   ├── project-architecture/
│   │   │   └── SKILL.md           # On-demand codebase map
│   │   ├── code-review/
│   │   │   └── SKILL.md           # PR review checklist
│   │   ├── test-writing/
│   │   │   └── SKILL.md           # Jest test patterns
│   │   ├── git-safe/
│   │   │   └── SKILL.md           # Diff + commit workflow
│   │   └── usage-monitor/         # ~/.claude/skills/ (global)
│   │       └── SKILL.md           # Weekly budget tracker (Pro/Max)
│   └── agents/
│       ├── code-explorer.md       # Haiku — cheap exploration
│       └── pr-reviewer.md         # Sonnet — thorough review
│
├── .github/
│   └── workflows/
│       ├── lint-test.yml          # Standard CI (no Claude)
│       ├── claude-review.yml      # PR review on open/sync
│       └── claude-security.yml    # Security scan on push
│
├── CLAUDE.md                      # Project context (≤150 lines)
├── REVIEW.md                      # Code quality contract
├── README.md
│
├── src/
│   ├── api/
│   ├── auth/
│   ├── db/
│   └── utils/
│
└── tests/
    ├── unit/
    ├── integration/
    └── e2e/
```

**File scope summary :**

| File | Loaded when | Token cost |
|---|---|---|
| `CLAUDE.md` | Every session start | Always — keep it short |
| `~/.claude/CLAUDE.md` | Every session start | Always — keep it short |
| `SKILL.md` (description only) | Every session start | ~30–50 tokens per skill |
| `SKILL.md` (full content) | When skill is invoked | On-demand only |
| `agents/*.md` (description only) | Every session start | ~30–50 tokens per agent |
| `agents/*.md` (full content) | When subagent spawns | On-demand only |

---

## 16. Gotchas & Common Mistakes

### Configuration mistakes

| Mistake | Correct approach |
|---|---|
| `"readOnlyMode": true` in settings.json | Does not exist. Use `permissions.deny` rules. |
| `"disabledMcpServers"` | Correct key is `"disabledMcpjsonServers"` |
| `mode: "review-only"` in claude-code-action | Does not exist. Use `--allowedTools` in `claude_args`. |
| `"allowFileModification": false` | Does not exist. Use `deny` rules on `Edit`/`Write`. |
| CLAUDE.md > 300 lines | Performance degrades. Split into skills. |
| Putting workflows in CLAUDE.md | Costs tokens every session. Move to skills. |

### Hooks mistakes

| Mistake | Correct approach |
|---|---|
| `post-edit.sh "$1"` (file path as arg) | Hooks don't receive args. Use `$CLAUDE_TOOL_INPUT_FILE_PATH` or parse stdin JSON. |
| Not making hooks executable | `chmod +x .claude/hooks/*.sh` is required. |
| Not checking `stop_hook_active` in Stop hooks | Causes infinite loops. Always gate on this field. |
| Hardcoding paths in hook commands | Use `$CLAUDE_PROJECT_DIR` prefix for portability. |
| Using `set -e` without testing exit codes | Unintended blocks. Test every exit path. |

### Context mistakes

| Mistake | Correct approach |
|---|---|
| `cat large-file.log` for debugging | `grep -i "error\|fail" logfile.log \| tail -50` |
| Reading entire directories to understand structure | `find . -name "*.ts" \| head -20` + grep for symbols |
| Not compacting between tasks | Context accumulates. `/compact` or `/clear` at task boundaries. |
| Asking Claude to read a PDF directly | Convert to text first: `pdftotext file.pdf - \| head -c 20000` |
| Ignoring the "context is getting large" warning | Act on it immediately — performance is already degrading. |

### GitHub Actions mistakes

| Mistake | Correct approach |
|---|---|
| `contents: write` on review-only jobs | `contents: read` is sufficient for reading code |
| No `fetch-depth: 0` in checkout | Claude can't see git diff accurately without full history |
| Using `GITHUB_TOKEN` when sticky comments are needed | Sticky comments only work with `claude[bot]` auth — remove `github_token` override |
| No `timeout-minutes` on Claude jobs | Long-running jobs burn quota. Set `timeout-minutes: 10` |

---

## References

| Resource | URL |
|---|---|
| Settings reference | https://code.claude.com/docs/en/settings |
| Hooks reference | https://code.claude.com/docs/en/hooks |
| GitHub Actions | https://code.claude.com/docs/en/github-actions |
| Best practices | https://code.claude.com/docs/en/best-practices |
| Manage costs | https://code.claude.com/docs/en/costs |
| Skill authoring | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices |
| Subagents | https://code.claude.com/docs/en/sub-agents |
| Memory & CLAUDE.md | https://code.claude.com/docs/en/memory |
| Model configuration | https://code.claude.com/docs/en/model-config |
| claude-code-action | https://github.com/anthropics/claude-code-action |
| ccusage (usage analytics) | https://github.com/ryoppippi/ccusage |
| Claude Code Usage Monitor | https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor |
| Pro/Max usage limits | https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan |

---

## Contributing

Contributions are welcome — open an issue or submit a PR.

**Guidelines :**
- Keep examples copy-paste ready (tested on macOS + Linux)
- One technique per section — no walls of text
- Update the Table of Contents if you add a section
- Test hook scripts before submitting: `echo '{"tool_name":"Read","tool_input":{"file_path":"/tmp/test"}}' | bash .claude/hooks/your-hook.sh`

---

## License

This playbook is released under the [MIT License](./LICENSE).
You are free to use, modify, and redistribute it — with attribution.

---

## Author

**Papa Sega WADE** — AI Research Engineer
Bridging software engineering and AI research — building tools, workflows, and systems that make LLM-assisted development practical at scale.

[papasegawade.com](https://papasegawade.com/) · [LinkedIn](https://www.linkedin.com/in/papa-s%C3%A9ga-wade-phd-a5727513a)

---
