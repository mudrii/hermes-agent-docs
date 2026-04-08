# Filesystem Checkpoints

Hermes Agent automatically snapshots your working directory before making file changes, giving you a safety net to roll back if something goes wrong. Checkpoints are **enabled by default** — there is zero cost when no file-mutating tools fire.

The checkpoint system is powered by an internal **Checkpoint Manager** that maintains a separate shadow git repository under `~/.hermes/checkpoints/`. Your real project `.git` is never touched.

> **v0.8.0 (v2026.4.8):** No checkpoint-related changes. See [memory.md](memory.md) for the memory system updates in this release.

## Quick Reference

| Command | Description |
|---------|-------------|
| `/rollback` | List all checkpoints with change stats |
| `/rollback <N>` | Restore to checkpoint N (also undoes last chat turn) |
| `/rollback diff <N>` | Preview diff between checkpoint N and current state |
| `/rollback <N> <file>` | Restore a single file from checkpoint N |

## What Triggers Checkpoints

Checkpoints are taken automatically before:

- **File tools** — `write_file` and `patch`
- **Destructive terminal commands** — `rm`, `mv`, `sed -i`, `truncate`, `shred`, output redirects (`>`), and `git reset`/`clean`/`checkout`

The agent creates at most one checkpoint per directory per conversation turn, so long-running sessions don't create excessive snapshots.

## How Checkpoints Work

At a high level:

1. Hermes detects when tools are about to modify files in your working tree
2. Once per conversation turn (per directory), it resolves the project root for the file
3. It initializes or reuses a **shadow git repo** tied to that directory at `~/.hermes/checkpoints/<hash>/`
4. It stages and commits the current state with a short, human-readable reason (e.g., `before patch`, `before terminal: rm -rf build/`)
5. These commits form a checkpoint history inspectable and restorable via `/rollback`

## Configuration

```yaml
# ~/.hermes/config.yaml
checkpoints:
  enabled: true          # master switch (default: true)
  max_snapshots: 50      # max checkpoints per directory
```

To disable entirely:

```yaml
checkpoints:
  enabled: false
```

When disabled, the Checkpoint Manager is a no-op and never attempts git operations.

---

## Context Compression and Checkpoints

Checkpoints and context compression are independent systems, but they interact:

- Checkpoints snapshot the **filesystem** before destructive tool calls.
- Context compression summarizes the **conversation history** to free context window space.

Both can be active simultaneously. `/compress` triggers manual compression without creating a filesystem checkpoint. `/rollback` restores the filesystem and undoes the corresponding conversation turns.

---

## v0.4.0 Compression Changes (PR #2323, #1727, #2224)

The context compressor was overhauled in v0.4.0:

- **Structured summaries** replace flat text: the compressor generates a structured summary with iterative updates rather than a single-pass summary block.
- **Token-budget tail protection**: the compressor calculates a tail budget and always preserves the most recent `protect_last_n` messages regardless of compression pressure.
- **Configurable `summary_base_url`**: the summarization model can be pointed at any OpenAI-compatible endpoint.
- **Fallback model support**: if the primary summary model fails, the compressor falls back to the next available model.

---

## v0.5.0 Compression Changes (PR #2554)

- **Ratio-based scaling replaces `summary_target_tokens`**: the deprecated `summary_target_tokens` key is removed. Token budget for retained content is now computed as `target_ratio × threshold × context_length`.
- **`target_ratio`, `protect_last_n`, `threshold`** are exposed as first-class keys in `DEFAULT_CONFIG` (sourced from `hermes_cli/config.py`).
- Summary output is capped at 12K tokens internally.
- On `/clear` and `/new`, the compressor summary and turn counter are reset (PR #3102).

### Current DEFAULT_CONFIG values (sourced from `hermes_cli/config.py`)

| Key | Default | Description |
|-----|---------|-------------|
| `compression.enabled` | `true` | Enable auto-compression |
| `compression.threshold` | `0.50` | Trigger at 50% of context limit |
| `compression.target_ratio` | `0.20` | Preserve 20% of threshold as tail |
| `compression.protect_last_n` | `20` | Always keep last 20 messages |
| `compression.summary_model` | `""` | Empty = use main model |
| `compression.summary_provider` | `"auto"` | Auto-select provider |

## Listing Checkpoints

From a CLI session:

```
/rollback
```

Output:

```text
Checkpoints for /path/to/project:

  1. 4270a8c  2026-03-16 04:36  before patch  (1 file, +1/-0)
  2. eaf4c1f  2026-03-16 04:35  before write_file
  3. b3f9d2e  2026-03-16 04:34  before terminal: sed -i s/old/new/ config.py  (1 file, +1/-1)

  /rollback <N>             restore to checkpoint N
  /rollback diff <N>        preview changes since checkpoint N
  /rollback <N> <file>      restore a single file from checkpoint N
```

Each entry shows:
- Short hash
- Timestamp
- Reason (what triggered the snapshot)
- Change summary (files changed, insertions/deletions)

## Previewing Changes with /rollback diff

Before committing to a restore, preview what has changed since a checkpoint:

```
/rollback diff 1
```

Output:

```text
test.py | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/test.py b/test.py
--- a/test.py
+++ b/test.py
@@ -1 +1 @@
-print('original content')
+print('modified content')
```

Long diffs are capped at 80 lines to avoid flooding the terminal.

## Restoring with /rollback

Restore to a checkpoint by number:

```
/rollback 1
```

Behind the scenes, Hermes:

1. Verifies the target commit exists in the shadow repo
2. Takes a **pre-rollback snapshot** of the current state so you can "undo the undo" later
3. Restores tracked files in your working directory
4. **Undoes the last conversation turn** so the agent's context matches the restored filesystem state

On success:

```text
Restored to checkpoint 4270a8c5: before patch
A pre-rollback snapshot was saved automatically.
Undid 4 message(s). Removed: "Now update test.py to ..."
  4 message(s) remaining in history.
  Chat turn undone to match restored file state.
```

The conversation undo ensures the agent doesn't remember changes that have been rolled back, avoiding confusion on the next turn.

## Single-File Restore

Restore just one file from a checkpoint without affecting the rest of the directory:

```
/rollback 1 src/broken_file.py
```

This is useful when the agent changed multiple files but only one needs to be reverted.

## Git Worktree Isolation

When using git worktrees, each worktree gets its own separate checkpoint history. The shadow repo hash is derived from the absolute path of the working directory, so:

- `../repo-experiment-a` has its own checkpoint history
- `../repo-experiment-b` has its own checkpoint history
- Each can use `/rollback` independently without affecting the other

This makes combining worktrees with checkpoints the safest way to run multiple parallel agents on the same repository.

```bash
# Create isolated worktrees for each agent
git worktree add ../repo-experiment-a feature/hermes-a
git worktree add ../repo-experiment-b feature/hermes-b

# Run agents independently
cd ../repo-experiment-a && hermes
cd ../repo-experiment-b && hermes
```

Alternatively, use the automatic `-w` flag:

```bash
hermes -w          # auto-creates a disposable worktree under .worktrees/
hermes -w -q "Fix issue #123"
```

## Where Checkpoints Live

All shadow repos live under:

```text
~/.hermes/checkpoints/
  ├── <hash1>/   # shadow git repo for one working directory
  ├── <hash2>/
  └── ...
```

Each `<hash>` is derived from the absolute path of the working directory. Inside each shadow repo:

- Standard git internals (`HEAD`, `refs/`, `objects/`)
- An `info/exclude` file with a curated ignore list
- A `HERMES_WORKDIR` file pointing back to the original project root

You normally never need to touch these manually. Checkpoint data is usually very small (git delta compression) and is not automatically pruned when worktrees are removed.

## Safety and Performance Guards

- **Git availability** — if `git` is not found on `PATH`, checkpoints are transparently disabled
- **Directory scope** — Hermes skips overly broad directories (root `/`, home `$HOME`)
- **Repository size** — directories with more than 50,000 files are skipped to avoid slow git operations
- **No-change snapshots** — if there are no changes since the last snapshot, the checkpoint is skipped
- **Non-fatal errors** — all errors inside the Checkpoint Manager are logged at debug level; tools continue to run

## Best Practices

- **Leave checkpoints enabled** — they're on by default and have zero cost when no files are modified
- **Use `/rollback diff` before restoring** — preview what will change to pick the right checkpoint
- **Use `/rollback` instead of `git reset`** when you want to undo agent-driven changes only
- **Combine with git worktrees** for maximum safety — each Hermes session in its own worktree with checkpoints as an extra layer
- **Use git commits for milestones** — checkpoints are a safety net; use regular git commits to capture meaningful progress
