# CLI Reference

This document covers every terminal command you run from your shell, every in-session slash command, all environment variables that affect CLI behavior, and the Copilot OAuth device code flow introduced in v0.3.0.

For configuration file reference, see [configuration.md](./configuration.md).
For provider details, see [providers.md](./providers.md).

---

## Entry Points

The `pyproject.toml` defines three installed scripts:

| Binary | Module | Purpose |
|--------|--------|---------|
| `hermes` | `hermes_cli.main:main` | Primary CLI — all commands live here |
| `hermes-agent` | `run_agent:main` | Programmatic agent runner |
| `hermes-acp` | `acp_adapter.entry:main` | ACP stdio server for editor integration |

---

## Global Entrypoint

```bash
hermes [global-options] <command> [subcommand/options]
```

Running `hermes` with no arguments starts an interactive chat session.

### Global Options

| Option | Description |
|--------|-------------|
| `--version`, `-V` | Show version and exit. |
| `--resume <session>`, `-r <session>` | Resume a previous session by ID or title. |
| `--continue [name]`, `-c [name]` | Resume the most recent session, or the most recent session matching a title. |
| `--worktree`, `-w` | Start in an isolated git worktree for parallel-agent workflows. |
| `--yolo` | Bypass dangerous-command approval prompts. |
| `--pass-session-id` | Include the session ID in the agent's system prompt. |

---

## Top-Level Commands

| Command | Purpose |
|---------|---------|
| `hermes chat` | Interactive or one-shot chat with the agent. |
| `hermes model` | Interactively choose the default provider and model. |
| `hermes gateway` | Run or manage the messaging gateway service. |
| `hermes setup` | Interactive setup wizard for all or part of the configuration. |
| `hermes whatsapp` | Configure and pair the WhatsApp bridge. |
| `hermes login` / `hermes logout` | Authenticate with OAuth-backed providers. |
| `hermes status` | Show agent, auth, and platform status. |
| `hermes cron` | Inspect and tick the cron scheduler. |
| `hermes doctor` | Diagnose config and dependency issues. |
| `hermes config` | Show, edit, migrate, and query configuration files. |
| `hermes pairing` | Approve or revoke messaging pairing codes. |
| `hermes skills` | Browse, install, publish, audit, and configure skills. |
| `hermes honcho` | Manage Honcho cross-session memory integration. |
| `hermes acp` | Run Hermes as an ACP server for editor integration. |
| `hermes tools` | Configure enabled tools per platform. |
| `hermes sessions` | Browse, export, prune, rename, and delete sessions. |
| `hermes insights` | Show token/cost/activity analytics. |
| `hermes claw` | OpenClaw migration helpers. |
| `hermes version` | Show version information. |
| `hermes update` | Pull latest code and reinstall dependencies. |
| `hermes uninstall` | Remove Hermes from the system. |

---

## `hermes chat`

```bash
hermes chat [options]
```

The primary command for talking to the agent. Running `hermes` with no subcommand is equivalent to `hermes chat`.

### Options

| Option | Type | Description |
|--------|------|-------------|
| `-q`, `--query "..."` | string | One-shot, non-interactive prompt. Hermes runs the query and exits. |
| `-m`, `--model <model>` | string | Override the model for this run. |
| `-t`, `--toolsets <csv>` | string | Enable a comma-separated set of toolsets (e.g., `web,terminal,skills`). |
| `--provider <provider>` | string | Force a specific provider. See provider list below. |
| `-v`, `--verbose` | flag | Verbose output. |
| `-Q`, `--quiet` | flag | Programmatic mode: suppress banner, spinner, and tool previews. |
| `--resume <session>` | string | Resume a session directly from `chat`. |
| `--continue [name]` | string | Resume the most recent session, or a named session. |
| `--worktree` | flag | Create an isolated git worktree for this run. |
| `--checkpoints` | flag | Enable filesystem checkpoints before destructive file changes. |
| `--yolo` | flag | Skip approval prompts. |
| `--pass-session-id` | flag | Pass the session ID into the system prompt. |

### `--provider` Values

The `--provider` flag accepts these values (as defined in `hermes_cli/main.py` and the `PROVIDER_REGISTRY` in `hermes_cli/auth.py`):

`auto`, `openrouter`, `nous`, `openai-codex`, `copilot`, `copilot-acp`, `anthropic`, `zai`, `kimi-coding`, `minimax`, `minimax-cn`, `opencode-zen`, `opencode-go`, `ai-gateway`, `kilocode`, `alibaba`, `deepseek`, `custom`

Aliases: `--provider claude` and `--provider claude-code` both map to `--provider anthropic`. `--provider vercel` maps to `--provider ai-gateway`. `--provider kimi` and `--provider moonshot` map to `--provider kimi-coding`. `--provider dashscope`, `--provider qwen`, and `--provider aliyun` map to `--provider alibaba`. See the full alias table in [providers.md](./providers.md).

### Examples

```bash
# Start interactive session
hermes
hermes chat

# One-shot query
hermes chat -q "Summarize the latest PRs"

# Force provider and model
hermes chat --provider openrouter --model anthropic/claude-sonnet-4.6

# Enable specific toolsets
hermes chat --toolsets web,terminal,skills

# Programmatic / scripted mode
hermes chat --quiet -q "Return only JSON"

# Worktree isolation for parallel work
hermes chat --worktree -q "Review this repo and open a PR"

# Resume most recent session
hermes chat --continue

# Resume by name
hermes chat --continue "my-project"
```

---

## `hermes model`

```bash
hermes model
```

Interactive provider and model selector. Use this to:

- Switch default providers.
- Log into OAuth-backed providers (Nous Portal, OpenAI Codex, GitHub Copilot) during model selection.
- Pick from provider-specific model lists.
- Save the new default into `config.yaml`.

The selected provider and model are persisted to `~/.hermes/config.yaml` under the `model:` key so later sessions use the saved selection even when `HERMES_INFERENCE_PROVIDER` is not set in the shell.

---

## `hermes gateway`

```bash
hermes gateway <subcommand>
```

Manages the messaging gateway service that connects Hermes to Telegram, Discord, Slack, WhatsApp, Signal, Mattermost, Matrix, Email, DingTalk, SMS (Twilio), and other platforms.

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `run` | Run the gateway in the foreground (logs to stdout). Accepts `--verbose` and `--replace`. |
| `start` | Start the installed gateway service. Accepts `--system` on Linux. |
| `stop` | Stop the service. Accepts `--system` on Linux. |
| `restart` | Restart the service. Accepts `--system` on Linux. |
| `status` | Show service status. Accepts `--deep` and `--system`. |
| `install` | Install as a user service (`systemd` on Linux, `launchd` on macOS). Accepts `--force`, `--system`, and `--run-as-user`. |
| `uninstall` | Remove the installed service. Accepts `--system` on Linux. |
| `setup` | Interactive messaging-platform setup. |

### `--system` Flag (Linux)

On Linux, passing `--system` to `install`, `start`, `stop`, `restart`, `status`, or `uninstall` targets the system-level systemd service instead of the user service. System service install requires root (`sudo`). The system service starts at boot and runs as a non-root user configured via `--run-as-user`.

### Examples

```bash
hermes gateway run           # Foreground (development)
hermes gateway install       # Install as a user background service
hermes gateway install --system --run-as-user myuser  # System service (boot-time)
hermes gateway start         # Start installed service
hermes gateway status        # Check if running
hermes gateway status --deep # Detailed status with logs
hermes gateway stop          # Stop the service
hermes gateway restart       # Restart the service
hermes gateway setup         # Configure messaging platforms
```

---

## `hermes setup`

```bash
hermes setup [model|terminal|gateway|tools|agent] [--non-interactive] [--reset]
```

Interactive setup wizard. Run with no section to go through the full wizard, or pass a section name to jump directly to that part.

### Sections

| Section | Description |
|---------|-------------|
| `model` | Provider and model setup. |
| `terminal` | Terminal backend and sandbox setup. |
| `gateway` | Messaging platform setup. |
| `tools` | Enable/disable tools per platform. |
| `agent` | Agent behavior settings. |

### Options

| Option | Description |
|--------|-------------|
| `--non-interactive` | Use defaults or environment values without prompts. |
| `--reset` | Reset configuration to defaults before setup. |

---

## `hermes whatsapp`

```bash
hermes whatsapp
```

Runs the WhatsApp pairing/setup flow, including mode selection (`bot` or `self-chat`) and QR-code pairing.

---

## `hermes login` / `hermes logout`

```bash
hermes login [--provider nous|openai-codex] [--portal-url ...] [--inference-url ...]
hermes logout [--provider nous|openai-codex]
```

Authenticate with OAuth-backed providers. `login` supports:

- **Nous Portal** -- OAuth/device code flow against `https://portal.nousresearch.com`
- **OpenAI Codex** -- OAuth/device code flow against the OpenAI auth endpoint

### `login` Options

| Option | Description |
|--------|-------------|
| `--no-browser` | Print the auth URL instead of opening a browser. |
| `--timeout <seconds>` | Polling timeout for the device code flow. |
| `--ca-bundle <pem>` | Custom CA bundle for TLS verification. |
| `--insecure` | Disable TLS verification (development only). |

GitHub Copilot authentication is handled separately through `hermes model` (OAuth device code flow) or via environment variables (`COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`). See [providers.md](./providers.md) for the full Copilot auth flow.

---

## `hermes status`

```bash
hermes status [--all] [--deep]
```

Show agent, auth, and platform status.

| Option | Description |
|--------|-------------|
| `--all` | Show all details in a shareable, redacted format. |
| `--deep` | Run deeper checks that may take longer. |

---

## `hermes cron`

```bash
hermes cron <list|create|edit|pause|resume|run|remove|status|tick>
```

Manage the cron scheduler for scheduled agent tasks.

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `list` | Show all scheduled jobs. Accepts `--all` to include disabled jobs. |
| `create` / `add` | Create a scheduled job from a prompt. Attach skills via repeated `--skill`. Accepts `--name`, `--deliver`, `--repeat`. |
| `edit` | Update a job's schedule, prompt, name, delivery, repeat count, or attached skills. Supports `--clear-skills`, `--add-skill`, `--remove-skill`. |
| `pause` | Pause a job without deleting it. |
| `resume` | Resume a paused job and compute its next future run. |
| `run` | Trigger a job on the next scheduler tick. |
| `remove` (aliases: `rm`, `delete`) | Delete a scheduled job. |
| `status` | Check whether the cron scheduler is running. |
| `tick` | Run due jobs once and exit. |

---

## `hermes doctor`

```bash
hermes doctor [--fix]
```

Diagnose configuration and dependency issues. Checks Python environment, required/optional packages, configuration files, auth providers (`codex` CLI availability, Nous Portal and OpenAI Codex login status), directory structure, external tools (git, ripgrep, docker, node), API connectivity, submodules, tool availability, Skills Hub, and Honcho memory.

| Option | Description |
|--------|-------------|
| `--fix` | Attempt automatic repairs where possible (create missing directories, config files, SOUL.md). |

---

## `hermes config`

```bash
hermes config <subcommand>
```

Show, edit, and manage `config.yaml` and `.env`.

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `show` | Show current config values. |
| `edit` | Open `config.yaml` in your `$EDITOR`. |
| `set <key> <value>` | Set a config value. API keys are automatically routed to `.env`; everything else goes to `config.yaml`. |
| `path` | Print the config file path (`~/.hermes/config.yaml`). |
| `env-path` | Print the `.env` file path (`~/.hermes/.env`). |
| `check` | Check for missing or stale config options. |
| `migrate` | Add newly introduced options interactively. |

### Examples

```bash
hermes config show
hermes config edit
hermes config set model anthropic/claude-opus-4.6
hermes config set terminal.backend docker
hermes config set OPENROUTER_API_KEY sk-or-...   # Saves to .env
hermes config check
hermes config migrate
```

---

## `hermes pairing`

```bash
hermes pairing <list|approve|revoke|clear-pending>
```

Manage DM pairing codes for messaging platforms.

| Subcommand | Description |
|------------|-------------|
| `list` | Show pending and approved users. |
| `approve <platform> <code>` | Approve a pairing code. |
| `revoke <platform> <user-id>` | Revoke a user's access. |
| `clear-pending` | Clear pending pairing codes. |

---

## `hermes skills`

```bash
hermes skills <subcommand>
```

Browse, install, publish, audit, and configure skills from online registries.

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `browse` | Paginated browser for skill registries. |
| `search` | Search skill registries. |
| `install` | Install a skill. |
| `inspect` | Preview a skill without installing it. |
| `list` | List installed skills. |
| `check` | Check installed hub skills for upstream updates. |
| `update` | Reinstall hub skills with upstream changes when available. |
| `audit` | Re-scan installed hub skills. |
| `uninstall` | Remove a hub-installed skill. |
| `publish` | Publish a skill to a registry. |
| `snapshot` | Export/import skill configurations. |
| `tap` | Manage custom skill sources. |
| `config` | Interactive enable/disable configuration for skills by platform. |

### Examples

```bash
hermes skills browse
hermes skills browse --source official
hermes skills search react --source skills-sh
hermes skills search https://mintlify.com/docs --source well-known
hermes skills inspect official/security/1password
hermes skills install official/migration/openclaw-migration
hermes skills install skills-sh/anthropics/skills/pdf --force
hermes skills check
hermes skills update
hermes skills config
```

### Notes

- `--force` overrides non-dangerous policy blocks for third-party/community skills.
- `--force` does **not** override a `dangerous` scan verdict.
- `--source skills-sh` searches the public `skills.sh` directory.
- `--source well-known` points Hermes at a site exposing `/.well-known/skills/index.json`.

---

## `hermes honcho`

```bash
hermes honcho <subcommand>
```

Manage Honcho cross-session memory integration.

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `setup` | Interactive Honcho setup wizard. |
| `status` | Show current Honcho config and connection status. |
| `sessions` | List known Honcho session mappings. |
| `map` | Map the current directory to a Honcho session name. |
| `peer` | Show or update peer names and dialectic reasoning level. Accepts `--user NAME`, `--ai NAME`, `--reasoning LEVEL`. |
| `mode` | Show or set memory mode: `hybrid`, `honcho`, or `local`. |
| `tokens` | Show or set token budgets. Accepts `--context N` and `--dialectic N`. |
| `identity` | Seed or show the AI peer identity representation. Pass a file path (e.g., `SOUL.md`) to seed. |
| `migrate` | Migration guide from openclaw-honcho to Hermes Honcho. |

---

## `hermes acp`

```bash
hermes acp
```

Starts Hermes as an ACP (Agent Client Protocol) stdio server for editor integration.

Related entry points:

```bash
hermes-acp
python -m acp_adapter
```

Install ACP support first:

```bash
pip install -e '.[acp]'
```

---

## `hermes tools`

```bash
hermes tools [--summary]
```

| Option | Description |
|--------|-------------|
| `--summary` | Print the current enabled-tools summary and exit. |

Without `--summary`, this launches the interactive per-platform tool configuration UI.

---

## `hermes sessions`

```bash
hermes sessions <subcommand>
```

Browse, manage, and export conversation sessions.

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `list` | List recent sessions. |
| `browse` | Interactive session picker with search and resume (curses-based). |
| `export <output> [--session-id ID]` | Export sessions to JSONL. |
| `delete <session-id>` | Delete one session. |
| `prune` | Delete old sessions. |
| `stats` | Show session-store statistics. |
| `rename <session-id> <title>` | Set or change a session title. |

---

## `hermes insights`

```bash
hermes insights [--days N] [--source platform]
```

Show token usage, cost, and activity analytics.

| Option | Description |
|--------|-------------|
| `--days <n>` | Analyze the last `n` days (default: 30). |
| `--source <platform>` | Filter by source: `cli`, `telegram`, `discord`, etc. |

---

## `hermes claw`

```bash
hermes claw migrate [--dry-run]
```

OpenClaw migration helpers. Migrates settings, memories, skills, and keys from OpenClaw to Hermes. Pass `--dry-run` to preview without changes.

---

## Maintenance Commands

| Command | Description |
|---------|-------------|
| `hermes version` | Print version information. |
| `hermes update` | Pull latest changes and reinstall dependencies. |
| `hermes uninstall [--full] [--yes]` | Remove Hermes. `--full` also deletes config/data. `--yes` skips confirmation. |

---

## Slash Commands

Slash commands are dispatched from a central `COMMAND_REGISTRY` in `hermes_cli/commands.py`. They are available in two surfaces:

- **Interactive CLI** -- type `/` for autocomplete menu. Built-in commands are case-insensitive.
- **Messaging gateway** -- dispatched by `gateway/run.py`, available in Telegram, Discord, Slack, WhatsApp, Signal, Email, Mattermost, Matrix, DingTalk, SMS, and Home Assistant chats.

### Session Commands

| Command | CLI | Messaging | Description |
|---------|-----|-----------|-------------|
| `/new` (alias: `/reset`) | Yes | Yes | Start a new session (fresh session ID and history). |
| `/clear` | Yes | -- | Clear screen and start a new session. |
| `/history` | Yes | -- | Show conversation history. |
| `/save` | Yes | -- | Save the current conversation. |
| `/retry` | Yes | Yes | Retry the last message (resend to agent). |
| `/undo` | Yes | Yes | Remove the last user/assistant exchange. |
| `/title [name]` | Yes | Yes | Set a title for the current session. |
| `/compress` | Yes | Yes | Manually compress conversation context (flush memories and summarize). |
| `/rollback [number]` | Yes | Yes | List or restore filesystem checkpoints. |
| `/stop` | Yes | Yes | Kill all running background processes. |
| `/background <prompt>` (alias: `/bg`) | Yes | Yes | Run a prompt in a separate background session. Results appear as a panel when the task finishes. |
| `/resume [name]` | Yes | Yes | Resume a previously-named session. |
| `/approve [session\|always]` | -- | Yes | Approve a pending dangerous command. |
| `/deny` | -- | Yes | Deny a pending dangerous command. |
| `/status` | -- | Yes | Show session info. |
| `/sethome` (alias: `/set-home`) | -- | Yes | Set this chat as the home channel for cron/notification delivery. |

### Configuration Commands

| Command | CLI | Messaging | Description |
|---------|-----|-----------|-------------|
| `/config` | Yes | -- | Show current configuration. |
| `/model [name]` | Yes | Yes | Show or change the current model. Supports `provider:model` syntax. |
| `/provider` | Yes | Yes | Show available providers and current provider. |
| `/prompt [text]` | Yes | -- | View/set custom system prompt. Subcommand: `clear`. |
| `/personality [name]` | Yes | Yes | Set a predefined personality. |
| `/statusbar` (alias: `/sb`) | Yes | -- | Toggle the context/model status bar. |
| `/verbose` | Yes | -- | Cycle tool progress display: off -> new -> all -> verbose. |
| `/reasoning [level\|show\|hide]` | Yes | Yes | Manage reasoning effort and display. Levels: `none`, `low`, `minimal`, `medium`, `high`, `xhigh`. Subcommands: `show`, `hide`, `on`, `off`. |
| `/skin [name]` | Yes | -- | Show or change the display skin/theme. |
| `/voice [on\|off\|tts\|status]` | Yes | Yes | Toggle voice mode. Recording key defaults to `Ctrl+B` (configurable via `voice.record_key` in `config.yaml`). |

### Tools and Skills Commands

| Command | CLI | Messaging | Description |
|---------|-----|-----------|-------------|
| `/tools [list\|disable\|enable] [name...]` | Yes | -- | Manage tools for the current session. Disabling a tool removes it from the agent's toolset and triggers a session reset. |
| `/toolsets` | Yes | -- | List available toolsets. |
| `/browser [connect\|disconnect\|status]` | Yes | -- | Manage local Chrome CDP connection. `connect` attaches browser tools (default: `ws://localhost:9222`). |
| `/skills [search\|browse\|inspect\|install]` | Yes | -- | Search, install, inspect, or manage skills from online registries. |
| `/cron [list\|add\|create\|edit\|pause\|resume\|run\|remove]` | Yes | -- | Manage scheduled tasks. |
| `/reload-mcp` (alias: `/reload_mcp`) | Yes | Yes | Reload MCP servers from `config.yaml`. |
| `/plugins` | Yes | -- | List installed plugins and their status. |

### Info Commands

| Command | CLI | Messaging | Description |
|---------|-----|-----------|-------------|
| `/help` | Yes | Yes | Show help message. |
| `/usage` | Yes | Yes | Show token usage, cost breakdown, and session duration. |
| `/insights [days]` | Yes | Yes | Show usage insights and analytics (last 30 days). |
| `/platforms` (alias: `/gateway`) | Yes | -- | Show gateway/messaging platform status. |
| `/paste` | Yes | -- | Check clipboard for an image and attach it. |
| `/update` | -- | Yes | Update Hermes Agent to the latest version. |

### Exit Commands

| Command | CLI | Messaging | Description |
|---------|-----|-----------|-------------|
| `/quit` (aliases: `/exit`, `/q`) | Yes | -- | Exit the CLI. |

### Dynamic Slash Commands

| Command | Description |
|---------|-------------|
| `/<skill-name>` | Load any installed skill as an on-demand command. Example: `/gif-search`, `/github-pr-workflow`, `/plan`. |

### Quick Commands

User-defined quick commands from `quick_commands` in `~/.hermes/config.yaml` are available as slash commands. They run shell commands directly without invoking the LLM. See [configuration.md](./configuration.md) for the `quick_commands` schema.

---

## Environment Variables Affecting CLI Behavior

The following environment variables are checked at startup by `hermes_cli/main.py` and `cli.py`. All variables can be stored in `~/.hermes/.env` (loaded on every startup) or set in the shell.

### Provider Selection

| Variable | Description |
|----------|-------------|
| `HERMES_INFERENCE_PROVIDER` | Override provider selection. Values: `auto`, `openrouter`, `nous`, `openai-codex`, `copilot`, `anthropic`, `zai`, `kimi-coding`, `minimax`, `minimax-cn`, `kilocode`, `alibaba`, `deepseek`, `opencode-zen`, `opencode-go`, `ai-gateway`. Default: `auto`. Lower priority than `config.yaml` `model.provider`. |

### Model Selection

| Variable | Description |
|----------|-------------|
| `HERMES_MODEL` | Preferred model name. Checked before `LLM_MODEL`. Also used by the gateway. |
| `LLM_MODEL` | Default model name. Fallback when not set in `config.yaml`. |

### Home Directory

| Variable | Description |
|----------|-------------|
| `HERMES_HOME` | Override Hermes config directory. Default: `~/.hermes`. Also scopes the gateway PID file and systemd service name so multiple installations can run concurrently. |

### Agent Behavior

| Variable | Description |
|----------|-------------|
| `HERMES_MAX_ITERATIONS` | Max tool-calling iterations per conversation. Default: `90`. |
| `HERMES_QUIET` | Suppress non-essential output. Values: `true`/`false`. |
| `HERMES_API_TIMEOUT` | LLM API call timeout in seconds. Default: `900`. |
| `HERMES_EXEC_ASK` | Enable execution approval prompts in gateway mode. Values: `true`/`false`. |
| `HERMES_BACKGROUND_NOTIFICATIONS` | Background process notification mode in gateway. Values: `all` (default), `result`, `error`, `off`. |
| `HERMES_EPHEMERAL_SYSTEM_PROMPT` | Ephemeral system prompt injected at API-call time. Never persisted to sessions. |
| `HERMES_PREFILL_MESSAGES_FILE` | Path to a JSON file of ephemeral prefill messages injected at API-call time. |
| `HERMES_DUMP_REQUESTS` | Dump API request payloads to log files. Values: `true`/`false`. |
| `HERMES_TIMEZONE` | IANA timezone override. Example: `America/New_York`. |
| `HERMES_YOLO_MODE` | Skip all approval prompts. Set automatically by `--yolo` flag. |

### Human Delay

| Variable | Description |
|----------|-------------|
| `HERMES_HUMAN_DELAY_MODE` | Response pacing. Values: `off`, `natural`, `custom`. |
| `HERMES_HUMAN_DELAY_MIN_MS` | Custom delay range minimum in milliseconds. Default: `800`. |
| `HERMES_HUMAN_DELAY_MAX_MS` | Custom delay range maximum in milliseconds. Default: `2500`. |

### Session Settings

| Variable | Description |
|----------|-------------|
| `SESSION_IDLE_MINUTES` | Reset sessions after N minutes of inactivity. Default: `1440`. |
| `SESSION_RESET_HOUR` | Daily reset hour in 24h format. Default: `4`. |

---

## GitHub Copilot Auth Flow (v0.3.0)

The Copilot provider introduced in v0.3.0 supports two authentication paths:

### Environment Variable Resolution

Credential resolution is implemented in `hermes_cli/copilot_auth.py`. Variables are checked in this order:

1. `COPILOT_GITHUB_TOKEN`
2. `GH_TOKEN`
3. `GITHUB_TOKEN`
4. `gh auth token` CLI fallback (checks common install locations: `$PATH`, `/opt/homebrew/bin/gh`, `/usr/local/bin/gh`, `~/.local/bin/gh`)

Classic PATs (`ghp_*` prefix) are explicitly rejected. Supported token types:

| Type | Prefix | How to get |
|------|--------|------------|
| OAuth token | `gho_` | `hermes model` -> GitHub Copilot -> Login with GitHub |
| Fine-grained PAT | `github_pat_` | GitHub Settings -> Developer settings -> Fine-grained tokens (needs Copilot Requests permission) |
| GitHub App token | `ghu_` | Via GitHub App installation |

### OAuth Device Code Flow

When no environment variable provides a usable token, `hermes model` offers a device code login. The flow:

1. Hermes sends a device code request to `https://github.com/login/device/code` using OAuth client ID `Ov23li8tweQw6odWQebz` (same as opencode and the Copilot CLI).
2. Hermes displays a verification URL and a user code.
3. You open the URL in a browser and enter the code.
4. Hermes polls `https://github.com/login/oauth/access_token` (5-second interval, with `slow_down` handling) until the token arrives or the flow times out (default: 300 seconds).
5. The resulting `gho_*` token is stored in `~/.hermes/auth.json` for subsequent sessions.

### API Mode Routing

Once a token is resolved, `copilot_runtime_api_mode()` in `hermes_cli/runtime_provider.py` selects the API mode:

- GPT-5+ models (except `gpt-5-mini`) automatically use the **Responses API** (`api_mode = codex_responses`).
- All other models (GPT-4o, Claude, Gemini, etc.) use **Chat Completions** (`api_mode = chat_completions`).
- Model auto-detection is done against the live Copilot catalog at `https://api.githubcopilot.com/models`.

### Copilot ACP Mode

`--provider copilot-acp` spawns the local Copilot CLI as a subprocess:

```bash
hermes chat --provider copilot-acp --model copilot-acp
```

Relevant environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `HERMES_COPILOT_ACP_COMMAND` | `copilot` | Override the Copilot ACP CLI binary path. |
| `COPILOT_CLI_PATH` | -- | Alias for `HERMES_COPILOT_ACP_COMMAND`. |
| `HERMES_COPILOT_ACP_ARGS` | `--acp --stdio` | Override Copilot ACP arguments. |
| `COPILOT_ACP_BASE_URL` | `acp://copilot` | Override Copilot ACP base URL. |

Requires the GitHub Copilot CLI in `$PATH` and an existing `copilot login` session. No token exchange is performed by Hermes -- the subprocess handles auth.

---

## Configuration Precedence

Settings are resolved in this order (highest priority first):

1. **CLI arguments** -- e.g., `hermes chat --model anthropic/claude-sonnet-4` (per-invocation override)
2. **`~/.hermes/config.yaml`** -- the primary config file
3. **`~/.hermes/.env`** -- fallback for env vars; required for secrets
4. **Built-in defaults** -- hardcoded safe defaults

For provider resolution specifically (`hermes_cli/runtime_provider.py`):

1. Explicit `--provider` flag
2. `model.provider` in `config.yaml`
3. `HERMES_INFERENCE_PROVIDER` environment variable
4. Auto-detection (`auto`)
