# claude-drive

Autonomous project driver slash command for Claude Code.

## What it does

`/drive` runs an autonomous development cycle: audit project state, select the next task, plan with subagent delegation, implement, cross-review with a parallel agent cluster, verify, document, commit, and push review snapshots with GPT quality gates.

## Installation

```bash
# Clone into your Claude commands directory
git clone https://github.com/Igzela/claude-drive.git ~/.claude/commands-drive
cp ~/.claude/commands-drive/commands/drive.md ~/.claude/commands/drive.md
```

Or manually copy `commands/drive.md` to `~/.claude/commands/drive.md`.

## Usage

```
/drive                                          # audit current directory, auto-select task
/drive /path/to/project                         # audit specific project
/drive /path/to/project "add search feature"    # specific task
/drive --dry-run                                # plan only, no changes
/drive --loop                                   # run up to 3 cycles
/drive --no-gpt-review                          # skip GPT quality gates
```

## Flags

| Flag | Effect |
|------|--------|
| `--dry-run` | Plan only, no file changes |
| `--no-commit` | Implement but leave uncommitted |
| `--no-push` | Commit locally but don't push review snapshots |
| `--no-gpt-review` | Skip ChromeDevTools GPT gates |
| `--loop` | Continue for up to 3 cycles |
| `--once` | Force single cycle (default) |

## How it works

### Phases

1. **Bootstrap** - Read project context (CLAUDE.md, AGENTS.md, handoff docs, git status)
2. **Select Task** - Pick next task by priority (explicit request > handoff > broken > planned > debt)
3. **Plan & Delegate** - Spawn parallel subagents (Explore, QA, Security, Docs) to gather context
4. **GPT Gate** - Push plan snapshot, get ChatGPT review via ChromeDevTools MCP
5. **Execute** - Implement with autonomous Tier 1/2 operations
6. **Cross-Review** - Spawn parallel review cluster (QA, Security, Performance, Compatibility, Docs)
7. **Verify** - Run tests, linter, type checker
8. **Document** - Update CLAUDE.md session log and handoff docs (structure-aware)
9. **Commit & Push** - Commit, push feature branch, get GPT review of code diff

### Safety

- **Tier 1**: Proceed autonomously (source files, tests, docs, linter, commits)
- **Tier 2**: Proceed then report (dependencies, CI/CD, config, deletes, restructuring)
- **Tier 3**: Stop and ask user (production, force push, secrets, external services, destructive commands)

### GPT Quality Gates

Uses ChromeDevTools MCP to drive ChatGPT as an external reviewer:
- Plan gate: reviews architecture and plan before implementation
- Checkpoint gate: reviews each completed phase diff
- Max 2 re-review rounds per gate
- BLOCK with CRITICAL/HIGH findings halts progress until fixed

## Requirements

- Claude Code with Agent tool available
- ChromeDevTools MCP server (for GPT gates)
- Logged-in ChatGPT session in Chrome
- Git repository in the target project

## License

MIT
