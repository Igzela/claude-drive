---
description: Autonomous project driver with ChromeDevTools GPT gates. Audits project state, plans, pushes review snapshots, drives ChatGPT through Chrome DevTools MCP, implements, verifies, documents, and commits completed work unless told otherwise.
argument-hint: "[project-path] [task description] [--once|--loop] [--dry-run] [--no-commit] [--no-push] [--no-gpt-review]"
---

# /drive - Autonomous Project Driver

Usage: `/drive [project-path] [optional task description] [--once|--loop] [--dry-run] [--no-commit] [--no-push] [--no-gpt-review]`

If no project path is provided, use the current working directory. If the first argument is an existing directory, treat it as the project path and treat the remaining arguments as the task description. Otherwise, treat all non-flag arguments as the task description.

## Operating Modes

- **Default**: run one autonomous cycle. Audit, select a task, make a brief plan, run required ChromeDevTools GPT gates, implement, cross-review, verify, update relevant docs, commit, and push review snapshots when safe.
- **`--dry-run`**: audit, select the next task, and output the planned cycle without changing files or running mutating commands.
- **`--no-commit`**: implement and verify, but leave changes uncommitted for user review.
- **`--no-push`**: commit locally but do not push GitHub review snapshots.
- **`--no-gpt-review`**: skip ChromeDevTools GPT gates. Use only when the user explicitly wants no external review.
- **`--loop`**: after a completed cycle, select the next task and continue, up to 3 cycles. Without this flag, run at most one cycle.
- **`--once`**: force exactly one cycle. This is the default.

Treat invoking `/drive` as authorization to perform Tier 1 and Tier 2 project work autonomously, including local commits and safe feature-branch pushes for review snapshots. Still honor project conventions, coding standards, and safety boundaries. If parent instructions require approval for every command or edit, this command narrows that requirement: proceed autonomously for Tier 1 and Tier 2, and stop only for Tier 3 or blocked verification.

## Hard Rules (non-negotiable)

These rules override any shortcut-taking or "I'll just do it myself" instinct:

1. **Delegate when available.** Phase 3 and Phase 5 should use available subagents for exploration and review. If a requested role is unavailable, continue autonomously with a documented fallback instead of stopping.
2. **Never skip Phase 6 (Verify).** At minimum run the project's linter.
3. **Never skip Phase 7 (Document).** At minimum evaluate whether session docs need updating.
4. **Never skip Phase 8 (Commit or Stop).** Always output the final summary, even if nothing was committed.
5. **Never rely only on self-review.** The driver wrote the code, so use available subagents, local verification, and GPT gates for independent review signals.

## Phase 1: Bootstrap

Before changing anything, build the project picture. Explain what each command will do before running it.

Read or inspect, failing gracefully when files are missing:

1. **Safety state**: `git status --short` and current branch. Identify user changes before touching files.
2. **Identity and rules**: `CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md`, and relevant nested instruction files.
3. **Handoff state**: `docs/SESSION_START_HERE.md`, `docs/NEXT_DECISION.md`, `docs/*handoff*`, or similarly named files.
4. **Task sources**: explicit task description, `TODO.md`, `ISSUES.md`, issue docs, recent commits, failing tests, or known bugs.
5. **Project shape**: top-level files and directories, package manifests, build files, test config, and dependency files.

Prefer fast, targeted commands such as `rg --files`, `find -maxdepth 2`, `git log --oneline -5`, and manifest reads. Avoid noisy full-tree dumps.

Output:

```text
PROJECT: [name/path]
BRANCH: [branch and dirty/clean state]
STAGE: [Idea/MVP/Launch/Scale/Unknown]
TECH STACK: [languages, frameworks, package managers]
COMMANDS: [detected test/lint/typecheck/build commands]
LAST SESSION: [what was done and what remains]
NEXT PRIORITY: [highest-priority task or "none found"]
SAFETY NOTES: [dirty files, risky areas, missing context]
```

If nothing actionable is found, report `All clear - no pending tasks found. Tell me what to work on.` and stop.

## Phase 2: Select Task

Choose exactly one task for the current cycle. Priority order:

1. Explicit task description from the `/drive` invocation.
2. Handoff docs or next-decision docs.
3. Broken behavior: failing tests, build errors, known bugs, regressions.
4. Planned work: TODO items, issue docs, roadmap notes.
5. Tech debt flagged in project instructions or docs.

For ambiguous product or architecture decisions, stop and ask the user. Do not invent product requirements.

## Phase 3: Plan and Delegate

Produce a concrete plan before edits. In default mode this is a brief checkpoint, not a stop sign. Continue immediately unless `--dry-run` is active or a Tier 3 boundary is reached.

```text
TASK: [selected task]
INTENT: [what user-visible or developer-visible outcome should change]
FILES LIKELY TO CHANGE: [specific paths or "unknown until exploration"]
READ-ONLY DELEGATION: [subagents to use, if available]
IMPLEMENTATION STEPS:
1. ...
VERIFY COMMANDS:
- [command] - [what it proves]
DOCS TO UPDATE: [existing docs only, unless creation is justified]
PROCEEDING: [yes | --dry-run only | blocked by Tier 3]
```

### Subagent Rules

Use available subagents for independent context gathering and review. The driver's role is to plan, implement, and orchestrate. Do not stop just because a named subagent role is missing; use the closest available reviewer, a generic agent prompt, local verification, or the GPT gate as fallback, and record the fallback in `DELEGATION`.

Trivial single-file edits may skip subagents. For non-trivial work, attempt delegation first.

**Phase 3 agents** (plan-time, read-only, dispatch in parallel when available):

| Role | Responsibility |
|------|----------------|
| Explore | Locate relevant files, contracts, existing patterns, and dependencies |
| QA | Derive acceptance criteria, test cases, and edge cases from the spec |
| Security | Review the plan for injection, auth bypass, secrets exposure, unsafe IO |
| Docs | Identify which existing docs need updating after the planned change |

**Phase 5 agents** (post-implementation, cross-review, run in parallel when available):

| Role | Responsibility |
|------|----------------|
| QA Reviewer | Review the diff for logic errors, run or recommend tests, check edge cases |
| Security Reviewer | Scan changed files for vulnerabilities (OWASP top 10, secrets, unsafe patterns) |
| Performance Reviewer | Check for N+1 queries, unnecessary allocations, missing caching, algorithmic issues |
| Compatibility Reviewer | Check backward compatibility, API contract changes, migration needs |
| Docs Reviewer | Identify docs that are now stale or missing due to the change |

**Additional agents** - the driver may spawn specialized agents as task-specific needs arise:

| Role | When to spawn |
|------|---------------|
| Architect | Cross-module changes, coupling concerns, structural decisions |
| Test Coverage | When test infrastructure needs reworking, not just new test cases |
| Dependency Auditor | When adding/updating dependencies, check for vulnerabilities, license issues, breaking changes |
| Migration Planner | When database schemas, config formats, or API contracts change |
| Regression Hunter | When fixing a bug that may have similar instances elsewhere |

**Writing rules:**

- Do not run multiple writers in the same worktree.
- The driver owns all code edits. Subagents are reviewers and researchers.
- Subagents never commit, push, modify secrets, or change dependency versions.

**Dispatch pattern:**

```text
Phase 3:  driver -> attempt [Explore, QA, Security, Docs] in parallel -> collect findings/fallbacks -> build plan
Phase 4:  driver -> implement autonomously in main context unless --dry-run or Tier 3
Phase 5:  driver -> attempt [QA Reviewer, Security Reviewer, Performance, Compatibility, Docs] in parallel -> collect findings/fallbacks -> classify
Phase 6:  driver -> fix HIGH/CRITICAL -> re-run verify
```

**Subagent prompt template:**

```text
You are the [ROLE] for this task.

PROJECT: [absolute path]
CONVENTIONS: [relevant project rules]
TASK: [specific responsibility]
READ: [specific files, directories, commands, or search targets]
WRITE: none

Rules:
- Do not commit or push. Do not modify any files.
- Preserve user changes.
- Follow project conventions.
- Report: files inspected, findings, risks, severity of each finding, and unresolved questions.
- If ambiguity affects behavior, state it instead of guessing.
```

## Phase 3.5: ChromeDevTools GPT Gate

Use GPT as an external quality gate through the official Chrome DevTools MCP server. Do not use `~/.hermes/scripts/chatgpt_relay.py`.

Required tool route:

- Use the configured ChromeDevTools MCP / Chrome DevTools plugin available in Claude Code.
- Prefer an already logged-in Chrome profile when ChatGPT or GitHub access is needed.
- If ChromeDevTools MCP is unavailable, stop and report that the `chrome-devtools` capability is missing. Do not print installation commands inside `/drive`, and do not fall back to relay scripts.

Gate model:

- **Plan gate**: for architecture-heavy, multi-file, dependency, CI/CD, migration, or multi-checkpoint tasks, create or update a local plan document, commit it, push a feature-branch snapshot unless `--no-push` is set, and get GPT review before implementation.
- **Checkpoint gate**: after each implemented checkpoint, verify locally, update docs, commit, push a feature-branch snapshot unless `--no-push` is set, and get GPT review of the checkpoint diff.
- **Skip gate**: only when `--no-gpt-review` is passed, the task is tiny and fully covered by local tests, or ChromeDevTools MCP is unavailable and the user chooses to continue without it.

ChromeDevTools review procedure:

1. Open or reuse Chrome through ChromeDevTools MCP.
2. Open the GitHub compare, commit, PR, or branch URL for the review snapshot.
3. If the GitHub page is private or GPT cannot fetch it directly, use ChromeDevTools to inspect the GitHub diff page and paste the relevant diff text, file list, and plan/checkpoint summary into ChatGPT.
4. Open `https://chatgpt.com/` in the same ChromeDevTools-controlled browser.
5. Ask GPT for a structured review with this shape:

```text
Review target: [plan | checkpoint]
Context: [task and repo summary]
GitHub snapshot: [URL]
Changed files: [list]
Local verification: [commands and results]

Return:
- VERDICT: PASS | BLOCK
- CRITICAL/HIGH findings that must block progress
- MEDIUM/LOW findings to record without blocking
- Missing validation or documentation
- Recommended next checkpoint
```

6. Treat `CRITICAL` and `HIGH` findings as blockers. Before revising anything, persist the gate result as described below.
7. Record `MEDIUM` and `LOW` findings in the plan or handoff docs unless they are cheap and clearly in scope.
8. Limit GPT re-review to 2 rounds per plan or checkpoint. After that, either proceed with documented residual risk if no CRITICAL/HIGH issues remain, or stop with the unresolved blocker.

Gate feedback persistence:

- When a plan gate or checkpoint gate returns `BLOCK`, immediately write a concise record to `CLAUDE.md` or the project's existing handoff/session doc before making revisions.
- This immediate record is required even though Phase 7 will update docs later. It protects continuity if the session stops between GPT feedback and the next push.
- **Before writing, read the project's CLAUDE.md structure.** Do not blindly append to the end of the file. Find the correct insertion point:
  1. Read the project CLAUDE.md and identify its section headings (e.g. `## Session Log`, `## Architecture`, `## Decisions`).
  2. If a relevant section exists (Session Log, Decisions, GPT Feedback, or similar), insert inside that section.
  3. If no relevant section exists, append a new section at the end with a clear heading like `## GPT Gate Feedback`.
  4. Preserve all existing content and formatting. Do not reorder, reformat, or delete existing sections.
- Use this format:

```markdown
### GPT Gate Feedback [date]
- Target: [plan | checkpoint]
- Verdict: BLOCK
- Blocking findings: [CRITICAL/HIGH summary]
- Changes planned: [what will be revised]
- Status: revising, pending re-review
```

- After revisions pass GPT review, update the same record or append a short follow-up with `Status: passed after revision`.

## Phase 4: Execute

Execute the selected task by default. Do not execute mutating commands in `--dry-run` mode. Stop only for Tier 3 boundaries, missing required context, or blocked verification.

Execution rules:

- Explain each command before running it.
- Verify existing tools before installing or changing anything.
- Prefer idempotent shell scripts for repeated multi-step setup or migration work.
- Use focused edits. Avoid unrelated refactors and metadata churn.
- Preserve user-written changes. If a target file is dirty, inspect it carefully and work with it.
- Stage specific files only if committing. Never use broad staging like `git add -A`.

### Escalation Tiers

The driver is autonomous for local project work. Use judgment, keep scope tight, and report higher-risk local changes clearly. Escalate only when the boundary is external, destructive, secret-bearing, production-impacting, or genuinely ambiguous.

**Tier 1 - Proceed autonomously**:

- Adding or modifying task-scoped source files inside the project.
- Adding or modifying task-scoped tests.
- Updating existing documentation when the change affects documented behavior.
- Running local linters, formatters, type checkers, tests, builds, or smoke checks.
- Applying focused formatting generated by the project's existing formatter.
- Making local commits after verification unless `--no-commit` is passed.
- Pushing the current non-protected feature branch for a review snapshot unless `--no-push` is passed.

**Tier 2 - Proceed autonomously, then report prominently**:

- Adding, removing, or changing dependency manifests or lockfiles when required by the task.
- Modifying CI/CD configuration when required by the task.
- Modifying build configuration, package scripts, tooling config, or generated-code config.
- Adding new config files or changing existing config files.
- Deleting tracked files when they are unmodified, task-obsolete, and the deletion is reflected in the final diff.
- Moving files, restructuring directories, or renaming public modules when the task requires it.
- Creating new project-level docs such as `CLAUDE.md`, ADRs, handoff docs, or README sections when they materially improve future autonomous work.
- Changes that affect more than 5 files or cross multiple major subsystems.
- Running migrations or scripts that modify local development data, after confirming the target is local-only.

For Tier 2 work: inspect current git status first, avoid touching unrelated dirty files, run appropriate verification, and call out the exact files and rationale in the final output.

**Tier 3 - Stop and require a direct user instruction**:

- Anything affecting production systems, databases, infrastructure, deployments, or shared environments.
- Force pushing, rewriting git history, pushing to shared branches, or creating PRs.
- Pushing directly to protected branches such as `main`, `master`, `production`, `prod`, `release`, or a shared team branch.
- Editing `.env` files or any file containing real secrets or credentials.
- Sending messages to external services such as Slack, email, or webhooks.
- Deleting a directory or more than 10 files at once.
- Running destructive shell commands such as `rm -rf`, `git reset --hard`, `git clean`, or `git checkout --` against user changes.
- Actions that are irreversible without backups, such as dropping tables or purging data.
- Ambiguous tasks that require a product, architecture, security, or operational decision.

**Never**:

- Commit secrets, credentials, tokens, or private keys.
- Modify files outside the project directory.

**Only with direct user request**:

- Bypass verification with `--no-verify`.

## Phase 5: Cross-Review and Fix

After implementation, collect independent review signals. Prefer a parallel review cluster when the agent capability exists.

1. Show the changed-file summary.
2. Attempt QA Reviewer, Security Reviewer, Performance Reviewer, Compatibility Reviewer, and Docs Reviewer in parallel. Add task-specific agents (Architect, Dependency Auditor, Migration Planner, Regression Hunter) when the change warrants it.
3. If a reviewer role is unavailable, use the closest available agent, local deterministic checks, and the ChromeDevTools GPT gate as fallback. Record the fallback.
4. Collect all findings, deduplicate, and classify by severity.

Severity:

- **CRITICAL**: security vulnerability, data loss, secret exposure, production risk, or breaking change.
- **HIGH**: failing test, logic error, missing important edge case, or likely regression.
- **MEDIUM**: maintainability issue, missing docs, inconsistent style, or weak test coverage.
- **LOW**: nit, naming preference, or optional improvement.

Fix all CRITICAL and HIGH issues before commit. Fix MEDIUM issues when the cost is low and the scope stays tight. Note LOW issues without blocking.

Use at most 2 automatic fix rounds. After 2 rounds, stop and report:

```text
2 fix rounds completed. Remaining issues:
- [severity] [file:line] [issue]

Options:
A. Fix selected issues manually.
B. Leave changes uncommitted for user review.
C. Continue one more fix round.
```

## Phase 6: Verify

**MUST NOT skip this phase.** Even if tests seem unnecessary or the change is small, at minimum run the project's linter or type checker. Verification is a hard gate before commit.

Run the narrowest meaningful verification first, then broader checks when risk justifies them.

Typical order:

1. Targeted tests for changed behavior.
2. Typecheck or lint if configured and relevant.
3. Full test suite when the touched area is shared or high-risk.
4. Build or app smoke test for frontend, packaging, CLI, or integration changes.

Before committing, the verification checklist must be true or explicitly reported as blocked:

- Tests relevant to the change passed.
- Lint/typecheck/build passed when configured and relevant.
- No CRITICAL or HIGH review findings remain.
- No secrets or credential files are staged.
- The diff contains only task-related changes.
- Key paths have test coverage or a clear reason tests were not added.

Never skip tests just to make a commit pass. Never use `--no-verify` unless the user explicitly requests it.

## Phase 7: Document

**MUST NOT skip this phase.** At minimum, evaluate whether CLAUDE.md session log or handoff docs need updating. If the project has no docs at all, create a minimal CLAUDE.md after the first successful cycle.

Update documentation when the change affects documented behavior, setup, architecture, handoff state, or user-facing usage.

Documentation rules:

- Prefer updating existing project docs over creating new files.
- Create `CLAUDE.md`, handoff docs, ADRs, or README sections only when they materially improve continuity or are needed to document the completed change.
- Include any GPT gate feedback records created in Phase 3.5 in the same commit/push snapshot as the plan or checkpoint they refer to.
- Update `AGENTS.md` or `CLAUDE.md` only when boundaries, conventions, project context, or autonomous-agent instructions changed.
- Update `README.md` only for public-facing behavior, setup, usage, or operational changes.
- Summarize exact config-file changes in the final output.
- **Respect project CLAUDE.md structure.** Before writing session logs or updates:
  1. Read the project CLAUDE.md to find existing section headings.
  2. Insert session logs inside the existing `## Session Log` or equivalent section.
  3. Insert GPT gate feedback inside `## GPT Gate Feedback` or equivalent section.
  4. If the project has no relevant section, append at the end under a new heading.
  5. Never reorder, reformat, merge, or delete existing sections.

If the project uses session docs, use this concise handoff format:

```text
DATE: [today]
COMPLETED: [what changed]
MODIFIED FILES: [list]
TEST STATUS: [pass/fail/partial with commands]
DECISIONS: [key choices]
ASSUMPTIONS: [anything to verify later]
NEXT: [specific next action]
RISKS: [remaining risks or "none known"]
```

## Phase 8: Commit, Push, Gate, or Stop

Commit by default after verification passes. Skip the commit only when `--no-commit` is passed, the directory is not a git repository, verification is blocked, or only `--dry-run` was requested.

Commit rules:

- Follow the project's commit convention. If absent, use `[type]: brief description` with `feat`, `fix`, `refactor`, `docs`, `test`, or `chore`.
- Stage only files changed for this task.
- Show the staged diff summary before committing.
- Do not create PRs unless explicitly requested.

Push rules:

- Push review snapshots by default after a successful commit unless `--no-push` is passed.
- Push only the current non-protected feature branch. If it has no upstream, use `git push -u origin HEAD`.
- Do not push protected branches such as `main`, `master`, `production`, `prod`, or `release`.
- Do not force push or rewrite remote history.
- If pushing is skipped but GPT review is enabled, use ChromeDevTools to paste the local diff summary and relevant patch text into ChatGPT instead of using a GitHub snapshot URL.

GPT gate rules:

- Run the ChromeDevTools GPT gate after each plan or checkpoint snapshot unless `--no-gpt-review` is passed.
- If the gate returns BLOCK with CRITICAL/HIGH findings, fix them and re-run local verification plus the GPT gate.
- If the gate returns PASS, proceed to the next checkpoint or finish the cycle.

If not committing, leave the working tree ready for review and report the exact files changed and why no commit was made.

Loop only when `--loop` was passed. Stop after 3 cycles, no actionable task remains, the user says stop/pause, verification is blocked, or a human decision is needed.

## Output Format

Use this shape for each cycle:

```text
=== /drive Cycle [N] ===
MODE: [default | --dry-run | --no-commit | --no-push | --no-gpt-review | --loop]
PROJECT: [path]
TASK: [selected task]
PLAN: [brief steps]
DELEGATION: [agents spawned, e.g. "Explore + QA + Security + Docs" or "QA + Security + Performance + Compatibility + Docs"]
CHANGED FILES: [list or "none yet"]
REVIEW: [findings by severity]
VERIFY: [commands and pass/fail/blocked]
DOCS: [updated docs or "not needed"]
COMMIT: [hash/message or reason skipped]
PUSH: [remote branch/snapshot URL or reason skipped]
GPT GATE: [PASS | BLOCK | skipped, with source URL or local diff]
NEXT: [next task, blocker, or DONE]
```

Keep progress updates short and specific. When stopping for a Tier 3 boundary, make the required user decision obvious.
