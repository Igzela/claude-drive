# claude-drive-lite

Lightweight version of `/drive` — same autonomous cycle, fewer rules.

## vs Full Version

| | `/drive` (full) | `/drive-lite` |
|---|---|---|
| Phases | 9 (1-8 + 3.5) | 8 (1-8) |
| Subagents per cycle | 9+ (4 plan + 5 review) | 6 (3 plan + 3 review) |
| Optional agents | 5 extra roles | none |
| GPT gates | plan gate + checkpoint gate + feedback persistence | plan gate only (optional) |
| Flags | 6 (`--dry-run`, `--no-commit`, `--no-push`, `--no-gpt-review`, `--loop`, `--once`) | 2 (`--dry-run`, `--loop`) |
| Output fields | 10 | 8 |
| Lines | ~450 | ~120 |

## Installation

```bash
cp commands/drive-lite.md ~/.claude/commands/drive-lite.md
```

## Usage

```
/drive-lite                                    # one full cycle
/drive-lite /path/to/project "add auth"        # specific task
/drive-lite --dry-run                          # plan only
/drive-lite --loop                             # up to 3 cycles
```

## When to use which

- **`/drive`** — complex projects, multi-checkpoint tasks, need full GPT review at every phase
- **`/drive-lite`** — simpler tasks, quick iterations, or when you want less orchestration overhead
