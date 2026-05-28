---
description: Lightweight autonomous project driver. Audits, plans with subagents, implements, reviews, verifies, documents, and commits.
argument-hint: "[project-path] [task] [--dry-run] [--loop]"
---

# /drive-lite - Autonomous Project Driver (Lightweight)

Usage: `/drive-lite [project-path] [task] [--dry-run] [--loop]`

No flags = run one full autonomous cycle and commit. `--dry-run` = plan only. `--loop` = up to 3 cycles.

## Phase 1: Bootstrap

Read project context, failing gracefully when files are missing:

1. `git status --short` and `git branch --show-current`
2. `CLAUDE.md`, `AGENTS.md`, handoff docs (`docs/SESSION_START_HERE.md`, `docs/NEXT_DECISION.md`)
3. `git log --oneline -5`, `TODO.md`, failing tests
4. Top-level structure: `find . -maxdepth 2 -type f`, `package.json` / `pyproject.toml`

Output:
```
PROJECT: [path] | BRANCH: [branch] | STAGE: [Idea/MVP/Launch/Scale]
TECH: [stack] | TESTS: [commands]
LAST SESSION: [what's pending]
NEXT TASK: [selected task or "none"]
```

## Phase 2: Select Task

Priority: explicit task > handoff docs > broken things > TODO > tech debt.
Ambiguous product decisions -> ask user.

## Phase 3: Plan

Spawn 3 subagents in parallel (skip for trivial single-file edits):

| Role | Job |
|------|-----|
| Explore | Find relevant files, patterns, dependencies |
| QA | Design test cases and edge cases from the spec |
| Security | Check for injection, auth, secrets, unsafe IO |

Collect findings, then output:
```
TASK: [what]
INTENT: [outcome]
FILES: [likely changes]
STEPS: [1, 2, 3...]
VERIFY: [test commands]
```

If task is architecture-heavy or multi-file, push a plan snapshot and get GPT review via ChromeDevTools MCP before proceeding. Skip if `--no-gpt-review` or ChromeDevTools unavailable.

## Phase 4: Execute

Implement the task. Rules:
- Explain before doing
- Focused edits only, no unrelated refactors
- Preserve user changes in dirty files
- Stage specific files, never `git add -A`

**Safety tiers:**
- **Auto**: source files, tests, docs, linter, formatter, local commits
- **Report after**: dependencies, CI/CD, config, deletes, restructuring (>5 files)
- **Ask first**: production, force push, secrets, external services, destructive commands

## Phase 5: Cross-Review

Spawn 3 reviewers in parallel after implementation:

| Role | Job |
|------|-----|
| QA Reviewer | Check diff for logic errors, run tests, find edge cases |
| Security Reviewer | Scan for vulnerabilities (OWASP top 10, secrets, unsafe patterns) |
| Docs Reviewer | Identify stale or missing docs |

Collect findings by severity (CRITICAL > HIGH > MEDIUM > LOW).
Fix CRITICAL + HIGH before commit. Max 2 fix rounds, then ask user.

## Phase 6: Verify

**Never skip.** At minimum run the project's linter.

1. Targeted tests for changed behavior
2. Lint / typecheck
3. Full test suite (if high-risk area)
4. Build / smoke test (if frontend, CLI, or packaging)

Must pass before commit. Never use `--no-verify`.

## Phase 7: Document

Update project CLAUDE.md or handoff docs. **Read existing structure first** — insert into the right section, don't blindly append.

At minimum, add a session log entry:
```
### Session [date]
- Built: [what]
- Decisions: [key choices]
- Next: [what comes next]
```

If GPT blocked the plan earlier, include that feedback record in the same commit.

## Phase 8: Commit & Push

Follow project's commit convention (or `[type]: description`). Stage only task files.

Push feature branch for GPT review if ChromeDevTools MCP is available. If GPT blocks with CRITICAL/HIGH findings, fix and re-push (max 2 rounds).

Output:
```
=== Cycle [N] ===
TASK: [what]
AGENTS: [spawned roles]
FILES: [changed]
REVIEW: [findings]
VERIFY: [pass/fail]
DOCS: [updated or not]
COMMIT: [hash]
NEXT: [task or DONE]
```

## Rules

1. Use subagents for exploration and review. If unavailable, continue with fallback and note it.
2. Never skip verify or document phases.
3. Never self-review — the driver wrote the code, so subagents or GPT review it.
4. Never commit secrets or modify files outside the project.
