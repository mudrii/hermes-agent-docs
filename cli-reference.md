# CLI Reference

Hermes Agent provides three entry points:

| Command | Purpose |
|---------|---------|
| `hermes` | Interactive terminal UI |
| `hermes-agent` | Programmatic agent runner |
| `hermes-acp` | Editor integration (ACP protocol) |

---

## Top-Level Commands

```
hermes [COMMAND] [FLAGS]
```

| Command | Description |
|---------|-------------|
| `hermes` / `hermes chat` | Start interactive CLI session |
| `hermes setup` | Interactive setup wizard |
| `hermes config` | Configuration management |
| `hermes model` | Model selection |
| `hermes gateway` | Messaging platform gateway |
| `hermes cron` | Scheduled task management |
| `hermes honcho` | Honcho memory management |
| `hermes skills` | Skills management |
| `hermes sessions` | Session history |
| `hermes status` | System status |
| `hermes doctor` | Diagnostics |
| `hermes version` | Show version |
| `hermes update` | Update to latest |
| `hermes uninstall` | Uninstall Hermes |

---

## Chat Flags

```bash
hermes [FLAGS]
hermes chat [FLAGS]
```

| Flag | Description |
|------|-------------|
| `--model <model>` | Override default model for this session |
| `--provider <provider>` | Override provider |
| `--toolset <name>` | Load specific toolset |
| `--skill <name>` | Load skill at start |
| `-w, --worktree` | Run in isolated git worktree |
| `--session <id>` | Resume existing session by ID or prefix |
| `--no-memory` | Disable persistent memory for this session |
| `--compact` | Compact display mode (less verbose) |
| `--max-turns <n>` | Override max agent turns |

---

## Interactive Slash Commands

These commands are available during an interactive chat session:

### Session Management

| Command | Description |
|---------|-------------|
| `/new` or `/reset` | Start a fresh conversation |
| `/retry` | Redo the last turn |
| `/undo` | Revert last message pair |
| `/stop` | Interrupt current agent work |
| `/compress` | Manually compress context |
| `/rollback` | Restore to last filesystem checkpoint |

### Model & Configuration

| Command | Description |
|---------|-------------|
| `/model [provider:model]` | Switch model mid-session |
| `/tools` | Show or configure enabled tools |
| `/toolset <name>` | Switch active toolset |
| `/reasoning [level]` | Set reasoning effort (xhigh/high/medium/low/minimal/none) |
| `/personality [name]` | Set display personality |
| `/skin [name]` | Change CLI skin/theme |

### Information

| Command | Description |
|---------|-------------|
| `/usage` | Show token usage for current session |
| `/insights [--days N]` | Session analytics and stats |
| `/platforms` | Show connected messaging platforms |

### Skills & Memory

| Command | Description |
|---------|-------------|
| `/skills` | Browse and manage skills |
| `/memory` | View or edit persistent memory |

### Session History

| Command | Description |
|---------|-------------|
| `/sessions browse` | Interactive session picker |
| `/session_search <query>` | Search past conversations with FTS5 |
| `/title <name>` | Set or rename current session |

---

## `hermes gateway` — Messaging Platform Gateway

```bash
hermes gateway [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `run` | Run gateway in foreground |
| `start` | Start gateway as background service |
| `stop` | Stop running gateway |
| `status` | Show gateway status and connected platforms |
| `setup` | Interactive platform setup wizard |
| `install` | Install as systemd unit (Linux) |
| `uninstall` | Remove systemd unit |

---

## `hermes cron` — Scheduled Tasks

```bash
hermes cron [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `list` | Show all scheduled tasks |
| `status` | Check if cron service is running |
| `add "<schedule>"` | Add task (natural language or cron expression) |
| `pause <id>` | Pause a task |
| `resume <id>` | Resume a paused task |
| `run <id>` | Run a task immediately |
| `remove <id>` | Delete a task |

**Schedule formats:**
```bash
hermes cron add "every 30m"           # Recurring every 30 minutes
hermes cron add "every 2h"            # Recurring every 2 hours
hermes cron add "0 9 * * *"           # Cron expression (daily at 9am)
hermes cron add "2026-12-25T09:00:00" # One-time at specific datetime
hermes cron add "daily 9am"           # Natural language
```

---

## `hermes honcho` — Honcho Memory Management

```bash
hermes honcho [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `setup` | Configure Honcho integration |
| `status` | Show current Honcho settings |
| `sessions` | List session mappings |
| `map <name>` | Map current directory to a session name |
| `peer` | Configure AI and user peer names |
| `mode [hybrid\|honcho\|context]` | Switch memory backend mode |
| `tokens [--context N]` | Set context token budgets |
| `identity <file>` | Seed AI identity (SOUL.md) |
| `migrate` | Show migration guide from old config |

**Memory modes:**
- `hybrid` — Auto-inject context + Honcho tools available (recommended)
- `honcho` — Only Honcho tools (explicit retrieval)
- `context` — Only auto-injected context (no tools)

---

## `hermes skills` — Skills Management

```bash
hermes skills [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `list` | Show installed skills |
| `browse [--source <src>]` | Browse available skills by source |
| `search <query>` | Search skills across all sources |
| `install <id>` | Install a skill |
| `remove <id>` | Uninstall a skill |
| `view <id>` | Show skill definition |
| `create <name>` | Create a new skill interactively |
| `edit <id>` | Edit an existing skill |
| `enable <id>` | Enable a disabled skill |
| `disable <id>` | Disable a skill without removing it |

**Skill sources:**
```bash
hermes skills browse --source official      # Official optional skills
hermes skills browse --source skills-sh     # skills.sh community directory
hermes skills browse --source community     # Community submissions
hermes skills install official/security/1password
hermes skills install openai/skills/k8s     # GitHub direct
```

---

## `hermes sessions` — Session History

```bash
hermes sessions [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `list [--days N]` | List recent sessions |
| `browse` | Interactive session browser |
| `view <id>` | View session messages |
| `search <query>` | Full-text search across sessions |
| `export <id>` | Export session to JSONL |
| `export-all` | Export all sessions |
| `prune [--days N]` | Delete old sessions |

---

## `hermes model` — Model Management

```bash
hermes model [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `(none)` | Interactive model selector |
| `list` | Show available models for current provider |
| `set <provider:model>` | Set default model |
| `test` | Test current model with a quick message |

**Model format:** `provider/model-name` (OpenRouter format)

**Examples:**
```bash
hermes model set anthropic/claude-opus-4.6
hermes model set openai/gpt-5.4-pro
hermes model set google/gemini-3-pro-preview
```

---

## `hermes config` — Configuration Management

```bash
hermes config [SUBCOMMAND]
```

| Subcommand | Description |
|------------|-------------|
| `show` | Display current configuration |
| `set <key> <value>` | Set a config value |
| `get <key>` | Get a config value |
| `edit` | Open config in $EDITOR |

---

## `hermes-agent` — Programmatic Runner

```bash
hermes-agent [FLAGS] "<prompt>"
```

Runs a single agent conversation non-interactively. Useful for scripting and automation.

| Flag | Description |
|------|-------------|
| `--model <model>` | LLM model to use |
| `--toolset <name>` | Toolset to enable |
| `--skill <name>` | Skill to load |
| `--session <id>` | Continue existing session |
| `--max-turns <n>` | Maximum agent turns |
| `--output-format <fmt>` | text / json / jsonl |

---

## `hermes-acp` — ACP Editor Integration

```bash
hermes-acp
```

Starts the ACP (Agent Communication Protocol) server on stdio, enabling editor integration. Typically invoked by VS Code, Zed, or JetBrains plugins automatically.

See [ACP Integration](acp.md) for setup instructions.

---

## Shell Completions

```bash
hermes completions bash   >> ~/.bashrc
hermes completions zsh    >> ~/.zshrc
hermes completions fish   > ~/.config/fish/completions/hermes.fish
```
