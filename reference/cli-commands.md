# CLI Commands Reference

This page covers the terminal commands you run from your shell.

For in-chat slash commands, see [Slash Commands Reference](./slash-commands.md).

## Global entrypoint

```bash
hermes [global-options] <command> [subcommand/options]
```

### Global options

| Option | Description |
|--------|-------------|
| `--version`, `-V` | Show version and exit. |
| `--profile <name>`, `-p <name>` | Select which Hermes profile to use for this invocation. Overrides the sticky default set by `hermes profile use`. |
| `--resume <session>`, `-r <session>` | Resume a previous session by ID or title. |
| `--continue [name]`, `-c [name]` | Resume the most recent session, or the most recent session matching a title. |
| `--worktree`, `-w` | Start in an isolated git worktree for parallel-agent workflows. |
| `--yolo` | Bypass dangerous-command approval prompts. |
| `--pass-session-id` | Include the session ID in the agent's system prompt. |

## Top-level commands

| Command | Purpose |
|---------|---------|
| `hermes chat` | Interactive or one-shot chat with the agent. |
| `hermes model` | Interactively choose the default provider and model. |
| `hermes gateway` | Run or manage the messaging gateway service. |
| `hermes setup` | Interactive setup wizard for all or part of the configuration. |
| `hermes whatsapp` | Configure and pair the WhatsApp bridge. |
| `hermes login` / `logout` | Authenticate with OAuth-backed providers. Deprecated — use `hermes auth` instead. |
| `hermes auth` | Manage credential pools -- add, list, remove, reset, set strategy. |
| `hermes logs` | Tail and filter agent log files (agent.log, errors.log, gateway.log). New in v0.8.0. |
| `hermes status` | Show agent, auth, and platform status. |
| `hermes cron` | Inspect and tick the cron scheduler. |
| `hermes webhook` | Manage dynamic webhook subscriptions for event-driven activation. |
| `hermes doctor` | Diagnose config and dependency issues. |
| `hermes config` | Show, edit, migrate, and query configuration files. |
| `hermes pairing` | Approve or revoke messaging pairing codes. |
| `hermes skills` | Browse, install, publish, audit, and configure skills. |
| `hermes honcho` | Manage Honcho cross-session memory integration. |
| `hermes acp` | Run Hermes as an ACP server for editor integration. |
| `hermes mcp` | Manage MCP server configurations and run Hermes as an MCP server. |
| `hermes plugins` | Manage Hermes Agent plugins (install, enable, disable, remove). |
| `hermes tools` | Configure enabled tools per platform. |
| `hermes sessions` | Browse, export, prune, rename, and delete sessions. |
| `hermes insights` | Show token/cost/activity analytics. |
| `hermes claw` | OpenClaw migration helpers. |
| `hermes profile` | Manage profiles -- multiple isolated Hermes instances. |
| `hermes completion` | Print shell completion scripts (bash/zsh). |
| `hermes version` | Show version information. |
| `hermes update` | Pull latest code and reinstall dependencies. |
| `hermes uninstall` | Remove Hermes from the system. |

## `hermes chat`

```bash
hermes chat [options]
```

Common options:

| Option | Description |
|--------|-------------|
| `-q`, `--query "..."` | One-shot, non-interactive prompt. |
| `-m`, `--model <model>` | Override the model for this run. |
| `-t`, `--toolsets <csv>` | Enable a comma-separated set of toolsets. |
| `--provider <provider>` | Force a provider: `auto`, `openrouter`, `nous`, `openai-codex`, `copilot-acp`, `copilot`, `anthropic`, `huggingface`, `zai`, `kimi-coding`, `minimax`, `minimax-cn`, `kilocode`. |
| `-s`, `--skills <name>` | Preload one or more skills for the session (can be repeated or comma-separated). |
| `-v`, `--verbose` | Verbose output. |
| `-Q`, `--quiet` | Programmatic mode: suppress banner/spinner/tool previews. |
| `--resume <session>` / `--continue [name]` | Resume a session directly from `chat`. |
| `--worktree` | Create an isolated git worktree for this run. |
| `--checkpoints` | Enable filesystem checkpoints before destructive file changes. |
| `--yolo` | Skip approval prompts. |
| `--pass-session-id` | Pass the session ID into the system prompt. |
| `--source <tag>` | Session source tag for filtering (default: `cli`). Use `tool` for third-party integrations that should not appear in user session lists. |

Examples:

```bash
hermes
hermes chat -q "Summarize the latest PRs"
hermes chat --provider openrouter --model anthropic/claude-sonnet-4.6
hermes chat --toolsets web,terminal,skills
hermes chat --quiet -q "Return only JSON"
hermes chat --worktree -q "Review this repo and open a PR"
```

## `hermes model`

Interactive provider + model selector.

```bash
hermes model
```

Use this to switch default providers, log into OAuth-backed providers, pick from provider-specific model lists, configure a custom/self-hosted endpoint, or save a new default into config.

Mid-session model switching via `/model`:

```
/model                              # Show current model and available options
/model claude-sonnet-4              # Switch model (auto-detects provider)
/model zai:glm-5                    # Switch provider and model
/model custom:qwen-2.5              # Use model on your custom endpoint
/model custom                       # Auto-detect model from custom endpoint
/model custom:local:qwen-2.5        # Use a named custom provider
/model openrouter:anthropic/claude-sonnet-4  # Switch back to cloud
```

Provider and base URL changes are persisted to `config.yaml` automatically. When switching away from a custom endpoint, the stale base URL is cleared.

## `hermes gateway`

```bash
hermes gateway <subcommand>
```

| Subcommand | Description |
|------------|-------------|
| `run` | Run the gateway in the foreground. |
| `start` | Start the installed gateway service. |
| `stop` | Stop the service. |
| `restart` | Restart the service. |
| `status` | Show service status. |
| `install` | Install as a user service (`systemd` on Linux, `launchd` on macOS). |
| `uninstall` | Remove the installed service. |
| `setup` | Interactive messaging-platform setup. |

## `hermes setup`

```bash
hermes setup [model|terminal|gateway|tools|agent] [--non-interactive] [--reset]
```

| Section | Description |
|---------|-------------|
| `model` | Provider and model setup. |
| `terminal` | Terminal backend and sandbox setup. |
| `gateway` | Messaging platform setup. |
| `tools` | Enable/disable tools per platform. |
| `agent` | Agent behavior settings. |

Options:

| Option | Description |
|--------|-------------|
| `--non-interactive` | Use defaults / environment values without prompts. |
| `--reset` | Reset configuration to defaults before setup. |

## `hermes whatsapp`

```bash
hermes whatsapp
```

Runs the WhatsApp pairing/setup flow, including mode selection and QR-code pairing.

## `hermes login` / `hermes logout`

```bash
hermes login [--provider nous|openai-codex] [--portal-url ...] [--inference-url ...]
hermes logout [--provider nous|openai-codex]
```

`login` supports Nous Portal OAuth/device flow and OpenAI Codex OAuth/device flow.

Options: `--no-browser`, `--timeout <seconds>`, `--ca-bundle <pem>`, `--insecure`.

## `hermes auth`

Manage credential pools for same-provider key rotation.

```bash
hermes auth                                              # Interactive wizard
hermes auth list                                         # Show all pools
hermes auth list openrouter                              # Show specific provider
hermes auth add openrouter --api-key sk-or-v1-xxx        # Add API key
hermes auth add anthropic --type oauth                   # Add OAuth credential
hermes auth remove openrouter 2                          # Remove by index
hermes auth reset openrouter                             # Clear cooldowns
```

Subcommands: `add`, `list`, `remove`, `reset`. When called with no subcommand, launches the interactive management wizard.

## `hermes logs`

New in v0.8.0.

```bash
hermes logs [source] [options]
```

Tail and filter Hermes log files from `~/.hermes/logs/`.

| Source | File | Contents |
|--------|------|----------|
| (default) | `agent.log` | General agent activity |
| `errors` | `errors.log` | Errors and exceptions only |
| `gateway` | `gateway.log` | Messaging gateway platform events |

| Option | Description |
|--------|-------------|
| `-n <lines>` | Number of lines (default: 50). |
| `-f`, `--follow` | Follow in real time. |
| `--level <level>` | Minimum log level: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. |
| `--session <id>` | Filter by session ID substring. |
| `--since <duration>` | Show lines from the last N minutes/hours (e.g., `30m`, `1h`). |

Examples:

```bash
hermes logs                          # last 50 lines of agent.log
hermes logs -f                       # follow agent.log
hermes logs errors                   # last 50 lines of errors.log
hermes logs gateway -n 100           # last 100 lines of gateway.log
hermes logs --level WARNING          # WARNING and above only
hermes logs --session abc123         # filter by session ID
hermes logs --since 1h               # lines from the last hour
hermes logs --since 30m -f           # follow, starting 30 min ago
```

## `hermes status`

```bash
hermes status [--all] [--deep]
```

| Option | Description |
|--------|-------------|
| `--all` | Show all details in a shareable redacted format. |
| `--deep` | Run deeper checks that may take longer. |

## `hermes cron`

```bash
hermes cron <list|create|edit|pause|resume|run|remove|status|tick>
```

| Subcommand | Description |
|------------|-------------|
| `list` | Show scheduled jobs. |
| `create` / `add` | Create a scheduled job from a prompt, optionally attaching skills via repeated `--skill`. |
| `edit` | Update a job's schedule, prompt, name, delivery, repeat count, or attached skills. Supports `--clear-skills`, `--add-skill`, `--remove-skill`. |
| `pause` | Pause a job without deleting it. |
| `resume` | Resume a paused job and compute its next future run. |
| `run` | Trigger a job on the next scheduler tick. |
| `remove` | Delete a scheduled job. |
| `status` | Check whether the cron scheduler is running. |
| `tick` | Run due jobs once and exit. |

## `hermes webhook`

```bash
hermes webhook <subscribe|list|remove|test>
```

Manage dynamic webhook subscriptions for event-driven agent activation.

| Subcommand | Description |
|------------|-------------|
| `subscribe` / `add` | Create a webhook route. Returns the URL and HMAC secret. |
| `list` / `ls` | Show all agent-created subscriptions. |
| `remove` / `rm` | Delete a dynamic subscription. |
| `test` | Send a test POST to verify a subscription is working. |

### `hermes webhook subscribe`

```bash
hermes webhook subscribe <name> [options]
```

| Option | Description |
|--------|-------------|
| `--prompt` | Prompt template with `{dot.notation}` payload references. |
| `--events` | Comma-separated event types to accept. Empty = all. |
| `--description` | Human-readable description. |
| `--skills` | Comma-separated skill names to load for the agent run. |
| `--deliver` | Delivery target: `log` (default), `telegram`, `discord`, `slack`, `github_comment`. |
| `--deliver-chat-id` | Target chat/channel ID for cross-platform delivery. |
| `--secret` | Custom HMAC secret. Auto-generated if omitted. |

Subscriptions persist to `~/.hermes/webhook_subscriptions.json` and are hot-reloaded by the webhook adapter.

## `hermes doctor`

```bash
hermes doctor [--fix]
```

`--fix` attempts automatic repairs where possible.

## `hermes config`

```bash
hermes config <subcommand>
```

| Subcommand | Description |
|------------|-------------|
| `show` | Show current config values. |
| `edit` | Open `config.yaml` in your editor. |
| `set <key> <value>` | Set a config value. |
| `path` | Print the config file path. |
| `env-path` | Print the `.env` file path. |
| `check` | Check for missing or stale config. |
| `migrate` | Add newly introduced options interactively. |

## `hermes pairing`

```bash
hermes pairing <list|approve|revoke|clear-pending>
```

| Subcommand | Description |
|------------|-------------|
| `list` | Show pending and approved users. |
| `approve <platform> <code>` | Approve a pairing code. |
| `revoke <platform> <user-id>` | Revoke a user's access. |
| `clear-pending` | Clear pending pairing codes. |

## `hermes skills`

```bash
hermes skills <subcommand>
```

| Subcommand | Description |
|------------|-------------|
| `browse` | Paginated browser for skill registries. |
| `search` | Search skill registries. |
| `install` | Install a skill. |
| `inspect` | Preview a skill without installing it. |
| `list` | List installed skills. |
| `check` | Check installed hub skills for upstream updates. |
| `update` | Reinstall hub skills with upstream changes. |
| `audit` | Re-scan installed hub skills. |
| `uninstall` | Remove a hub-installed skill. |
| `publish` | Publish a skill to a registry. |
| `snapshot` | Export/import skill configurations. |
| `tap` | Manage custom skill sources. |
| `config` | Interactive enable/disable configuration for skills by platform. |

Examples:

```bash
hermes skills browse
hermes skills browse --source official
hermes skills search react --source skills-sh
hermes skills inspect official/security/1password
hermes skills install official/migration/openclaw-migration
hermes skills install skills-sh/anthropics/skills/pdf --force
hermes skills check
hermes skills update
hermes skills config
```

Notes:
- `--force` can override non-dangerous policy blocks for third-party/community skills.
- `--force` does not override a `dangerous` scan verdict.
- `--source skills-sh` searches the public `skills.sh` directory.
- `--source well-known` lets you point Hermes at a site exposing `/.well-known/skills/index.json`.

## `hermes honcho`

```bash
hermes honcho <subcommand>
```

| Subcommand | Description |
|------------|-------------|
| `setup` | Interactive Honcho setup wizard. |
| `status` | Show current Honcho config and connection status. |
| `sessions` | List known Honcho session mappings. |
| `map` | Map the current directory to a Honcho session name. |
| `peer` | Show or update peer names and dialectic reasoning level. |
| `mode` | Show or set memory mode: `hybrid`, `honcho`, or `local`. |
| `tokens` | Show or set token budgets for context and dialectic. |
| `identity` | Seed or show the AI peer identity representation. |
| `migrate` | Migration guide from openclaw-honcho to Hermes Honcho. |

## `hermes acp`

```bash
hermes acp
```

Starts Hermes as an ACP (Agent Client Protocol) stdio server for editor integration.

Related entrypoints: `hermes-acp`, `python -m acp_adapter`.

Install support: `pip install -e '.[acp]'`.

## `hermes mcp`

```bash
hermes mcp <subcommand>
```

| Subcommand | Description |
|------------|-------------|
| `serve [-v\|--verbose]` | Run Hermes as an MCP server -- expose conversations to other agents. |
| `add <name> [--url URL] [--command CMD] [--args ...] [--auth oauth\|header]` | Add an MCP server with automatic tool discovery. |
| `remove <name>` (alias: `rm`) | Remove an MCP server from config. |
| `list` (alias: `ls`) | List configured MCP servers. |
| `test <name>` | Test connection to an MCP server. |
| `configure <name>` (alias: `config`) | Toggle tool selection for a server. |

## `hermes plugins`

```bash
hermes plugins [subcommand]
```

Running `hermes plugins` with no subcommand launches an interactive curses checklist to enable/disable installed plugins.

| Subcommand | Description |
|------------|-------------|
| `install <identifier> [--force]` | Install a plugin from a Git URL or `owner/repo`. |
| `update <name>` | Pull latest changes for an installed plugin. |
| `remove <name>` (aliases: `rm`, `uninstall`) | Remove an installed plugin. |
| `enable <name>` | Enable a disabled plugin. |
| `disable <name>` | Disable a plugin without removing it. |
| `list` (alias: `ls`) | List installed plugins with enabled/disabled status. |

Disabled plugins are stored in `config.yaml` under `plugins.disabled`.

## `hermes tools`

```bash
hermes tools [--summary]
```

Without `--summary`, launches the interactive per-platform tool configuration UI. With `--summary`, prints the current enabled-tools summary and exits. Requires an interactive terminal.

## `hermes sessions`

```bash
hermes sessions <subcommand>
```

| Subcommand | Description |
|------------|-------------|
| `list` | List recent sessions. |
| `browse` | Interactive session picker with search and resume. |
| `export <output> [--session-id ID]` | Export sessions to JSONL. |
| `delete <session-id>` | Delete one session. |
| `prune` | Delete old sessions. |
| `stats` | Show session-store statistics. |
| `rename <session-id> <title>` | Set or change a session title. |

## `hermes insights`

```bash
hermes insights [--days N] [--source platform]
```

| Option | Description |
|--------|-------------|
| `--days <n>` | Analyze the last `n` days (default: 30). |
| `--source <platform>` | Filter by source such as `cli`, `telegram`, or `discord`. |

## `hermes claw`

```bash
hermes claw migrate [options]
```

Migrate your OpenClaw setup to Hermes. Reads from `~/.openclaw` (or custom path) and writes to `~/.hermes`. Automatically detects legacy directory names (`~/.clawdbot`, `~/.moldbot`).

| Option | Description |
|--------|-------------|
| `--dry-run` | Preview what would be migrated without writing anything. |
| `--preset <name>` | Migration preset: `full` (default, includes secrets) or `user-data` (excludes API keys). |
| `--overwrite` | Overwrite existing Hermes files on conflicts (default: skip). |
| `--migrate-secrets` | Include API keys in migration (enabled by default with `--preset full`). |
| `--source <path>` | Custom OpenClaw directory (default: `~/.openclaw`). |
| `--workspace-target <path>` | Target directory for workspace instructions (AGENTS.md). |
| `--skill-conflict <mode>` | Handle skill name collisions: `skip` (default), `overwrite`, or `rename`. |
| `--yes` | Skip the confirmation prompt. |

## `hermes profile`

```bash
hermes profile <subcommand>
```

See [Profile Commands Reference](./profile-commands.md) for full details.

| Subcommand | Description |
|------------|-------------|
| `list` | List all profiles. |
| `use <name>` | Set a sticky default profile. |
| `create <name> [--clone] [--no-alias]` | Create a new profile. |
| `delete <name> [-y]` | Delete a profile. |
| `show <name>` | Show profile details. |
| `alias <name> [--remove] [--name NAME]` | Manage wrapper scripts. |
| `rename <old> <new>` | Rename a profile. |
| `export <name> [-o FILE]` | Export a profile to a `.tar.gz` archive. |
| `import <archive> [--name NAME]` | Import a profile from a `.tar.gz` archive. |

## Maintenance commands

| Command | Description |
|---------|-------------|
| `hermes version` | Print version information. |
| `hermes update` | Pull latest changes and reinstall dependencies. |
| `hermes uninstall [--full] [--yes]` | Remove Hermes, optionally deleting all config/data. |

## See also

- [Slash Commands Reference](./slash-commands.md)
- [Profile Commands Reference](./profile-commands.md)
- [Environment Variables Reference](./environment-variables.md)
