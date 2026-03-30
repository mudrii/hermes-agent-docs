# Gateway Architecture

The Hermes gateway is the long-running background process that connects Hermes Agent to external messaging platforms. It manages incoming messages, session state, platform authentication, cron scheduling, and outbound delivery — all through a single unified pipeline.

This document covers v0.2.0 through v0.5.0 (v2026.3.28).

---

## What the Gateway Is

The gateway is not a webhook handler or a stateless API proxy. It is a persistent process that:

- Maintains long-lived connections to all configured platforms simultaneously
- Routes incoming messages through a per-chat session store that preserves conversation history
- Dispatches messages to the `AIAgent` pipeline, which handles tool execution, memory, and reasoning
- Runs the cron scheduler, ticking every 60 seconds to execute any due jobs
- Proactively delivers output to configured home channels
- Manages session lifecycle: creation, reset, compression, and expiry
- Emits hook callbacks to trusted local Python code via `gateway/hooks.py`

The gateway is implemented in `gateway/run.py` with supporting modules for configuration (`gateway/config.py`), session management (`gateway/session.py`), delivery (`gateway/delivery.py`), pairing (`gateway/pairing.py`), status tracking (`gateway/status.py`), and platform adapters (`gateway/platforms/`).

---

## Architecture

```
flowchart TB
    subgraph Gateway["Hermes Gateway"]
        subgraph Adapters["Platform adapters"]
            tg[Telegram]
            dc[Discord]
            wa[WhatsApp]
            sl[Slack]
            sig[Signal]
            sms[SMS]
            em[Email]
            ha[Home Assistant]
            mm[Mattermost]
            mx[Matrix]
            dt[DingTalk]
            api["API Server (OpenAI-compatible)"]
            wh[Webhook]
        end

        store["Session store (per chat)"]
        agent["AIAgent — run_agent.py"]
        cron["Cron scheduler — ticks every 60s"]
    end

    tg --> store
    dc --> store
    wa --> store
    sl --> store
    sig --> store
    sms --> store
    em --> store
    ha --> store
    mm --> store
    mx --> store
    dt --> store
    api --> store
    wh --> store
    store --> agent
    cron --> store
```

Each platform adapter:

1. Connects to the external platform using that platform's native protocol
2. Normalizes all incoming events into a `MessageEvent` object (defined in `gateway/platforms/base.py`)
3. Routes the event through the session store to load or create the appropriate conversation context
4. Dispatches the hydrated event to `AIAgent`
5. Collects the agent's response and delivers it back to the platform

---

## Platform Abstraction

All platform adapters inherit from `BasePlatformAdapter` in `gateway/platforms/base.py`. The base class defines the interface that every platform must implement:

- `connect() -> bool` — Connect to the platform. Returns True on success.
- `disconnect() -> None` — Gracefully disconnect.
- `send(chat_id, content, reply_to, metadata) -> SendResult` — Send a message to a chat.
- `get_chat_info(chat_id) -> dict` — Return chat name and type.

The base class also provides shared implementations for:

- `handle_message(event)` — Dispatches messages to background tasks, tracks active sessions, and handles interrupt signaling
- `extract_images(content)` — Parses markdown and HTML image tags from responses and routes them to `send_image()`
- `extract_media(content)` — Parses `MEDIA:<path>` tags and `[[audio_as_voice]]` directives from TTS tool output
- `extract_local_files(content)` — Auto-detects bare local file paths in responses for native media delivery
- `truncate_message(content, max_length)` — Splits long messages into chunks, preserving code block fence boundaries, with `(1/N)` indicators appended to each chunk
- `_keep_typing(chat_id)` — Continuously refreshes the typing indicator every 2 seconds while the agent is running

Platform adapters that override media sending methods (`send_image`, `send_voice`, `send_video`, `send_document`, `send_animation`, `send_image_file`) provide native attachment delivery. Platforms that do not override these methods fall back to sending plain-text descriptions.

### Message and Media Caching

When users send images, voice messages, or documents on messaging platforms, the base adapter downloads the file to a local cache directory so the agent can access it even after platform URLs expire:

- Images cached at `{HERMES_HOME}/image_cache/` — available to the vision tool
- Audio cached at `{HERMES_HOME}/audio_cache/` — available for STT transcription
- Documents cached at `{HERMES_HOME}/document_cache/` — available for file access

Supported document types for download and caching: `.pdf`, `.md`, `.txt`, `.docx`, `.xlsx`, `.pptx`.

---

## Session Management

### Session Keying

The gateway creates and looks up sessions by a session key computed from the incoming `MessageEvent`. The key depends on:

- The platform identifier
- The chat ID (group, DM, or channel)
- The user ID (when `group_sessions_per_user` is enabled)
- The thread or topic ID (when present)

This means each distinct (platform, chat, user, thread) combination gets its own isolated session with its own conversation history.

Session key format:
- DMs: `agent:main:<platform>:dm:<chat_id>[:<thread_id>]`
- Groups/channels: `agent:main:<platform>:<chat_type>:<chat_id>[:<thread_id>][:<user_id>]`

The `user_id` component is included only when `group_sessions_per_user` is enabled (default). For Signal, the adapter uses the UUID (`user_id_alt`) in preference to the phone number for session keying.

### Session Storage

Sessions are stored in SQLite (via `hermes_state.SessionDB`) as the primary store, with legacy JSONL files as a fallback. The session index lives at `~/.hermes/sessions/sessions.json`. Transcripts are stored both in the SQLite database and as `~/.hermes/sessions/<session_id>.jsonl` files for backward compatibility.

### Session Persistence

Sessions persist across messages until they reset. The agent remembers your conversation context between messages without any action required on your part.

### Reset Policies

Sessions reset based on configurable policies. The policy is stored in `GatewayConfig` and loaded from `gateway.json` or `config.yaml`.

| Policy mode | Description |
|-------------|-------------|
| `daily` | Reset at a specific hour each day (default: 4:00 AM) |
| `idle` | Reset after N minutes of inactivity (default: 1440 minutes = 24 hours) |
| `both` | Whichever triggers first — daily boundary or idle timeout |
| `none` | Never auto-reset; context managed only by compression |

Configure per-platform reset overrides in `~/.hermes/gateway.json`:

```json
{
  "reset_by_platform": {
    "telegram": { "mode": "idle", "idle_minutes": 240 },
    "discord": { "mode": "idle", "idle_minutes": 60 }
  }
}
```

Or set global overrides via environment variables:

```bash
SESSION_IDLE_MINUTES=120
SESSION_RESET_HOUR=3
```

Reset policy priority (highest wins): platform override > type override (dm/group/thread) > default.

### Multi-User Isolation in Group Chats

When multiple users share a channel or group chat, the gateway isolates their sessions by default. The config key is `group_sessions_per_user` (default: `true`).

With isolation enabled (default):
- Each user's messages in a shared channel are keyed to their user ID
- Two people chatting in the same Discord channel each have separate histories
- One user's in-flight tool run cannot be interrupted by another user's message

With isolation disabled (`group_sessions_per_user: false`):
- All users in a channel share one session
- Useful for a shared collaborative room
- Carries shared context costs and shared interrupt behavior

Set in `~/.hermes/config.yaml`:

```yaml
group_sessions_per_user: true
```

This became the enforced default in v0.3.0 (previously it could fall back to shared sessions in some circumstances).

### PII Redaction in System Prompts

When `redact_pii` is enabled, the dynamic system prompt that the agent receives has user IDs and chat IDs replaced with deterministic hashes. This prevents the LLM from seeing raw phone numbers or numeric IDs. Redaction is applied only on platforms where the LLM does not need raw IDs for mentions: WhatsApp, Signal, and Telegram. Discord is excluded because its mention syntax (`<@user_id>`) requires the real user ID.

---

## Starting the Gateway

### Interactive Setup

The recommended first step is the interactive setup wizard, which walks through configuring each platform:

```bash
hermes gateway setup
```

This shows which platforms are already configured, prompts for credentials, and offers to start or restart the gateway when done.

### Running the Gateway

```bash
hermes gateway              # Run in foreground (Ctrl+C to stop)
hermes gateway install      # Install as a user service (Linux systemd / macOS launchd)
sudo hermes gateway install --system   # Linux only: install a boot-time system service
hermes gateway start        # Start the installed service
hermes gateway stop         # Stop the installed service
hermes gateway status       # Check service status
hermes gateway status --system         # Linux only: inspect the system service
```

### Linux (systemd)

```bash
hermes gateway install               # Install as user service
hermes gateway start                 # Start the service
hermes gateway stop                  # Stop the service
hermes gateway status                # Check status
journalctl --user -u hermes-gateway -f  # View logs

# Enable lingering so the service survives logout
sudo loginctl enable-linger $USER

# Or install a boot-time system service
sudo hermes gateway install --system
sudo hermes gateway start --system
journalctl -u hermes-gateway -f
```

Use the user service on laptops and dev machines. Use the system service on VPS or headless hosts that need to come back at boot without relying on systemd linger. Hermes warns if both the user and system gateway units are installed simultaneously, because `start`/`stop`/`status` behavior becomes ambiguous.

Multiple Hermes installations on the same machine (different `HERMES_HOME` directories) each get a distinct systemd service name. The default `~/.hermes` uses `hermes-gateway`; others use `hermes-gateway-<hash>`. The `hermes gateway` commands automatically target the correct service for the current `HERMES_HOME`.

v0.3.0 also added: system service mode for boot-time operation, gateway install scope prompts (user vs system), auto-enable systemd linger during gateway install on headless servers, and auto-detect D-Bus session bus for `systemctl --user` on headless servers.

### macOS (launchd)

```bash
hermes gateway install
launchctl start ai.hermes.gateway
launchctl stop ai.hermes.gateway
tail -f ~/.hermes/logs/gateway.log
```

### Status Tracking

The gateway writes a PID file at `{HERMES_HOME}/gateway.pid` and a runtime status file at `{HERMES_HOME}/gateway_state.json`. The status file tracks:

- `gateway_state` — `starting`, `running`, or `stopped`
- `exit_reason` — reason for shutdown if applicable
- `platforms` — per-platform state (`connected`, `disconnected`, `fatal`) with error codes and messages
- `pid`, `start_time`, `updated_at`

The `is_gateway_running()` function in `gateway/status.py` checks the PID file, verifies the process is alive, validates it matches the expected command-line pattern, and cleans up stale PID files automatically.

---

## Configuration Structure

The gateway loads configuration from multiple sources with a defined priority order (highest wins):

1. Environment variables
2. `~/.hermes/config.yaml` (primary user-facing config)
3. `~/.hermes/gateway.json` (legacy; provides defaults under `config.yaml`)
4. Built-in defaults

### config.yaml Keys that Apply to the Gateway

```yaml
# Session reset policy (global default)
session_reset:
  mode: both          # daily | idle | both | none
  at_hour: 4          # 0-23, local time
  idle_minutes: 1440  # minutes of inactivity

# Multi-user session isolation in group chats
group_sessions_per_user: true

# What to do when an unauthorized user DMs the bot
unauthorized_dm_behavior: pair   # pair | ignore

# Reset trigger commands
reset_triggers:
  - /new
  - /reset

# User-defined quick commands (bypass the agent loop)
quick_commands:
  /hello: "Hello! How can I help you?"

# Always save cron outputs to local files
always_log_local: true

# Auto-transcribe inbound voice messages
stt:
  enabled: true

# Real-time token streaming to messaging platforms (v0.3.0)
# Supported platforms: Telegram, Discord, Slack, WhatsApp, Mattermost, Matrix
# Platforms without edit_message support (Signal, Email, Home Assistant, DingTalk,
# SMS) gracefully fall back to sending the final response in one message.
streaming:
  enabled: false
  transport: edit           # "edit" = progressive editMessageText
  edit_interval: 0.3        # seconds between message edits
  buffer_threshold: 40      # chars before forcing an edit
  cursor: " ▉"             # cursor shown during streaming

# Tool progress notifications
display:
  tool_progress: all        # off | new | all | verbose
  background_process_notifications: all  # all | result | error | off

# Per-platform settings (Discord example)
discord:
  require_mention: true
  free_response_channels:
    - "123456789012345678"
  auto_thread: false

# Per-platform unauthorized_dm_behavior override
whatsapp:
  unauthorized_dm_behavior: ignore
```

### gateway.json Keys

The `~/.hermes/gateway.json` file is the legacy configuration path. Settings here are overridden by `config.yaml`. It is used for:

- `platforms` — per-platform enable/token/home_channel configuration
- `reset_by_platform` — per-platform reset policy overrides
- `reset_by_type` — reset overrides by session type (dm, group, thread)

For Home Assistant event filtering, `gateway.json` is currently the required location:

```json
{
  "platforms": {
    "homeassistant": {
      "enabled": true,
      "extra": {
        "watch_domains": ["climate", "binary_sensor", "alarm_control_panel"],
        "ignore_entities": ["sensor.uptime", "sensor.cpu_usage"],
        "cooldown_seconds": 30
      }
    }
  }
}
```

### Environment Variables for Platform Credentials

Environment variables are set in `~/.hermes/.env`. They override all config file settings.

Note: DingTalk and Home Assistant do not auto-enable from environment variables alone. They must be enabled in `~/.hermes/gateway.json` or `~/.hermes/config.yaml` under the `platforms` section (or via `hermes gateway setup`).

| Platform | Key Variables |
|----------|---------------|
| Telegram | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, `TELEGRAM_HOME_CHANNEL` |
| Discord | `DISCORD_BOT_TOKEN`, `DISCORD_ALLOWED_USERS`, `DISCORD_HOME_CHANNEL` |
| Slack | `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`, `SLACK_ALLOWED_USERS`, `SLACK_HOME_CHANNEL` |
| WhatsApp | `WHATSAPP_ENABLED`, `WHATSAPP_MODE`, `WHATSAPP_ALLOWED_USERS` |
| Signal | `SIGNAL_HTTP_URL`, `SIGNAL_ACCOUNT`, `SIGNAL_ALLOWED_USERS`, `SIGNAL_HOME_CHANNEL` |
| Mattermost | `MATTERMOST_TOKEN`, `MATTERMOST_URL`, `MATTERMOST_ALLOWED_USERS`, `MATTERMOST_HOME_CHANNEL` |
| Matrix | `MATRIX_ACCESS_TOKEN`, `MATRIX_HOMESERVER`, `MATRIX_USER_ID`, `MATRIX_PASSWORD`, `MATRIX_ALLOWED_USERS`, `MATRIX_HOME_ROOM` |
| Home Assistant | `HASS_TOKEN`, `HASS_URL` |
| Email | `EMAIL_ADDRESS`, `EMAIL_PASSWORD`, `EMAIL_IMAP_HOST`, `EMAIL_SMTP_HOST`, `EMAIL_ALLOWED_USERS` |
| SMS | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER`, `SMS_ALLOWED_USERS` |
| DingTalk | `DINGTALK_CLIENT_ID`, `DINGTALK_CLIENT_SECRET`, `DINGTALK_ALLOWED_USERS` |
| Webhook | No credential env vars — authentication is per-route via HMAC `secret` in `config.yaml` |
| API Server | `API_SERVER_ENABLED`, `API_SERVER_KEY`, `API_SERVER_PORT`, `API_SERVER_HOST` |

---

## Authorization

By default, the gateway denies all users not in an allowlist or paired via DM. This is the safe default for a bot with terminal access.

Authorization checks run in this order (first match wins):
1. Home Assistant events are always authorized (authenticated by `HASS_TOKEN`)
2. Per-platform allow-all flag (e.g., `DISCORD_ALLOW_ALL_USERS=true`)
3. DM pairing approved list (checked regardless of allowlists)
4. Platform-specific allowlist (e.g., `TELEGRAM_ALLOWED_USERS`)
5. Global allowlist (`GATEWAY_ALLOWED_USERS`)
6. Global allow-all (`GATEWAY_ALLOW_ALL_USERS=true`)
7. Default: deny

### Allowlists

Set platform-specific allowlists in `~/.hermes/.env`:

```bash
TELEGRAM_ALLOWED_USERS=123456789,987654321
DISCORD_ALLOWED_USERS=284102345871466496
SIGNAL_ALLOWED_USERS=+15551234567,+10987654321
SMS_ALLOWED_USERS=+15551234567
EMAIL_ALLOWED_USERS=trusted@example.com
MATTERMOST_ALLOWED_USERS=3uo8dkh1p7g1mfk49ear5fzs5c
MATRIX_ALLOWED_USERS=@alice:matrix.org
DINGTALK_ALLOWED_USERS=user-id-1
```

Or use a gateway-wide allowlist:

```bash
GATEWAY_ALLOWED_USERS=123456789,987654321
```

Per-platform allow-all flags bypass the allowlist for a single platform:

```bash
TELEGRAM_ALLOW_ALL_USERS=true
DISCORD_ALLOW_ALL_USERS=true
SLACK_ALLOW_ALL_USERS=true
WHATSAPP_ALLOW_ALL_USERS=true
SIGNAL_ALLOW_ALL_USERS=true
EMAIL_ALLOW_ALL_USERS=true
SMS_ALLOW_ALL_USERS=true
MATTERMOST_ALLOW_ALL_USERS=true
MATRIX_ALLOW_ALL_USERS=true
DINGTALK_ALLOW_ALL_USERS=true
```

Or explicitly allow all users on all platforms (not recommended for bots with terminal access):

```bash
GATEWAY_ALLOW_ALL_USERS=true
```

### DM Pairing

Instead of manually configuring user IDs, enable DM pairing. Unknown users who DM the bot receive a one-time pairing code. You approve them from the CLI:

```bash
# The user sees: "Pairing code: XKGH5N7P"
hermes pairing approve telegram XKGH5N7P

# List pending and approved users
hermes pairing list

# Revoke access
hermes pairing revoke telegram 123456789
```

Pairing security features:
- 8-character codes from a 32-character unambiguous alphabet (no 0/O/1/I)
- Cryptographic randomness via `secrets.choice()`
- Codes expire after 1 hour
- Maximum 3 pending codes per platform
- Rate limiting: 1 request per user per 10 minutes
- Lockout after 5 failed approval attempts (1 hour cooldown)
- All pairing data files are stored with chmod `0600`
- Codes are never logged to stdout
- Storage: `~/.hermes/pairing/`

DM pairing behavior is controlled by `unauthorized_dm_behavior`:
- `pair` (default) — unknown DM senders receive a pairing code
- `ignore` — unknown DM senders are silently ignored

This can be set globally in `config.yaml` or overridden per-platform:

```yaml
# Global default
unauthorized_dm_behavior: pair

# WhatsApp-specific override
whatsapp:
  unauthorized_dm_behavior: ignore
```

---

## Approval System and /stop Command

### Smart Approvals

v0.3.0 introduced a Codex-inspired approval system. When the agent tries to run a potentially dangerous command, it asks for approval in the chat:

> This command is potentially dangerous (recursive delete). Reply "yes" to approve.

Reply `yes` or `y` to approve, `no` or `n` to deny. The system remembers your decisions using a policy table and applies them to future similar commands, avoiding repeated prompts for commands you've already approved.

v0.4.0 replaced bare text confirmation with explicit `/approve` and `/deny` commands ([PR #2002](https://github.com/NousResearch/hermes-agent/pull/2002)). The agent now prompts:

> This command is potentially dangerous. Reply `/approve` to allow or `/deny` to block.

The old `yes`/`no` text responses no longer work for approvals in gateway mode.

### /stop Command

The `/stop` command interrupts the currently running agent without queuing a follow-up message. This is distinct from sending a regular message during a run (which interrupts and queues the new message as the next prompt).

Complete interrupt behavior:
- Any message sent while the agent is running triggers an interrupt
- In-progress terminal commands are killed immediately (SIGTERM, then SIGKILL after 1 second)
- Tool calls are cancelled — only the currently-executing tool runs; the rest are skipped
- Multiple messages sent during interruption are combined into one prompt
- `/stop` interrupts without queuing any follow-up message

---

## Chat Commands

These commands are available in any connected messaging platform:

| Command | Description |
|---------|-------------|
| `/new` or `/reset` | Start a fresh conversation |
| `/model [provider:model]` | Show or change the model (supports `provider:model` syntax) |
| `/provider` | Show available providers with auth status |
| `/personality [name]` | Set a personality |
| `/retry` | Retry the last message |
| `/undo` | Remove the last exchange |
| `/status` | Show session info |
| `/stop` | Stop the running agent |
| `/approve` | Approve a pending dangerous-command request (v0.4.0, PR #2002) |
| `/deny` | Deny a pending dangerous-command request (v0.4.0, PR #2002) |
| `/cost` | Show live pricing and usage for this session (v0.4.0, PR #2180) |
| `/verbose` | Toggle tool output verbosity from chat (v0.5.0, PR #3262) |
| `/sethome` | Set this chat as the home channel |
| `/compress` | Manually compress conversation context |
| `/title [name]` | Set or show the session title |
| `/resume [name]` | Resume a previously named session |
| `/usage` | Show token usage for this session |
| `/insights [days]` | Show usage insights and analytics |
| `/reasoning [level\|show\|hide]` | Change reasoning effort or toggle display |
| `/voice [on\|off\|tts\|join\|leave\|status]` | Control voice replies and Discord voice-channel behavior |
| `/rollback [number]` | List or restore filesystem checkpoints |
| `/background <prompt>` | Run a prompt in a separate background session |
| `/reload-mcp` | Reload MCP servers from config |
| `/update` | Update Hermes Agent to the latest version |
| `/help` | Show available commands |
| `/<skill-name>` | Invoke any installed skill |

---

## Platform-Specific Tool Configuration

Each platform has its own named toolset. The toolset name determines which tools the agent can use when responding to messages from that platform.

| Platform | Toolset name | Capabilities |
|----------|-------------|--------------|
| CLI | `hermes-cli` | Full access |
| Telegram | `hermes-telegram` | Full tools including terminal |
| Discord | `hermes-discord` | Full tools including terminal |
| WhatsApp | `hermes-whatsapp` | Full tools including terminal |
| Slack | `hermes-slack` | Full tools including terminal |
| Signal | `hermes-signal` | Full tools including terminal |
| SMS | `hermes-sms` | Full tools including terminal |
| Email | `hermes-email` | Full tools including terminal |
| Home Assistant | `hermes-homeassistant` | Full tools + `ha_list_entities`, `ha_get_state`, `ha_call_service`, `ha_list_services` |
| Mattermost | `hermes-mattermost` | Full tools including terminal |
| Matrix | `hermes-matrix` | Full tools including terminal |
| DingTalk | `hermes-dingtalk` | Full tools including terminal |
| Webhook | `hermes-webhook` | Full tools including terminal |
| API Server | `hermes-api-server` | Full tools including terminal |

Per-platform tool configuration is managed via `hermes tools`, which provides a curses UI for enabling and disabling tools on a per-platform basis.

---

## Logging and Status Monitoring

### Log Locations

On Linux with systemd user service:
```bash
journalctl --user -u hermes-gateway -f
```

On Linux with systemd system service:
```bash
journalctl -u hermes-gateway -f
```

On macOS with launchd:
```bash
tail -f ~/.hermes/logs/gateway.log
```

When running in the foreground (`hermes gateway`), the gateway logs directly to stdout/stderr.

### Runtime Status File

The file `~/.hermes/gateway_state.json` contains the live runtime status. It is updated whenever a platform connects, disconnects, or encounters an error:

```json
{
  "pid": 12345,
  "kind": "hermes-gateway",
  "gateway_state": "running",
  "exit_reason": null,
  "platforms": {
    "telegram": {
      "state": "connected",
      "error_code": null,
      "error_message": null,
      "updated_at": "2026-03-17T12:00:00+00:00"
    }
  },
  "updated_at": "2026-03-17T12:00:00+00:00"
}
```

Platform states: `connected`, `disconnected`, `fatal`.

Fatal errors record an `error_code` and `error_message`. For example, a Telegram polling conflict produces `error_code: "telegram_polling_conflict"` and the platform stops attempting to reconnect.

### Scoped Locks

The gateway uses machine-local lock files stored at `{XDG_STATE_HOME}/hermes/gateway-locks/` (defaulting to `~/.local/state/hermes/gateway-locks/`) to prevent two gateway instances from using the same bot token simultaneously. If a second gateway attempts to start with a token already in use by another local process, it logs an error and refuses to connect that platform with `error_code: "telegram_token_lock"`.

### Tool Progress Notifications

Control how much tool activity is visible in chat via `~/.hermes/config.yaml`:

```yaml
display:
  tool_progress: all    # off | new | all | verbose
```

When enabled, the bot sends status messages as it works:

```
💻 `ls -la`...
🔍 web_search...
📄 web_extract...
🐍 execute_code...
```

### Background Process Notifications

Control what you receive when an agent running in a background session uses `terminal(background=true)`:

```yaml
display:
  background_process_notifications: all    # all | result | error | off
```

| Mode | What you receive |
|------|-----------------|
| `all` | Running-output updates and the final completion message (default) |
| `result` | Only the final completion message |
| `error` | Only the final message when exit code is non-zero |
| `off` | No process watcher messages |

This can also be set via environment variable: `HERMES_BACKGROUND_NOTIFICATIONS=result`.

---

## Background Sessions

Run a prompt in a separate background session so the agent works independently while your main chat stays responsive:

```
/background Check all servers in the cluster and report any issues
```

Hermes confirms immediately with a task ID. The result is delivered to the same chat when the task finishes, prefixed with "Background task complete" or "Background task failed".

Background sessions:
- Have their own isolated session with separate conversation history
- Have no knowledge of your current chat context — only the prompt you provide
- Inherit your model, provider, toolsets, and reasoning settings from the gateway setup
- Are non-blocking — your main chat stays fully interactive
- Can themselves use `terminal(background=true)` which emits background process notifications

---

## Honcho Integration

When Honcho is enabled, the gateway keeps persistent Honcho managers aligned with session lifetimes and platform-specific session keys. In a multi-user gateway, session context is threaded explicitly through the call chain so each user's Honcho session remains separate. v0.3.0 formalized this isolation for multi-user gateway mode.

When a session resets, resumes, or expires, the gateway flushes memories before discarding context. The flush creates a temporary `AIAgent` with:
- `session_id` set to the old session's ID (so transcripts load correctly)
- `honcho_session_key` set to the gateway session key (so Honcho writes go to the right place)
- `sync_honcho=False` passed to `run_conversation()` (so the synthetic flush turn doesn't write back to Honcho's conversation history)

After the flush completes, queued Honcho writes are drained and the gateway-level Honcho manager is shut down for that session key.

---

## Event Hooks

The gateway fires event hooks at key lifecycle points. Hooks are Python scripts discovered from `~/.hermes/hooks/` directories:

```
~/.hermes/hooks/
  my-hook/
    HOOK.yaml      # name, description, events list
    handler.py     # async def handle(event_type, context)
```

Available events:
- `gateway:startup` -- Gateway process starts
- `session:start` -- New session created
- `session:end` -- Session ends (user ran `/new` or `/reset`)
- `session:reset` -- Session reset completed
- `agent:start` -- Agent begins processing a message
- `agent:step` -- Each turn in the tool-calling loop
- `agent:end` -- Agent finishes processing
- `command:*` -- Any slash command executed (wildcard match)

Errors in hooks are caught and logged but never block the main pipeline.

---

## Delivery System

The delivery router (`gateway/delivery.py`) handles outbound message routing for cron job outputs and proactive notifications. Delivery targets can be:

- `"origin"` -- back to the chat where the job was created
- `"local"` -- save to `~/.hermes/cron/output/` as a file
- `"telegram"` -- to the platform's home channel
- `"telegram:123456"` -- to a specific chat ID on a platform

Oversized cron output (over 4,000 characters) is truncated for platform delivery, with the full output saved to disk at `~/.hermes/cron/output/`.

### Channel Directory

The gateway maintains a cached channel directory at `~/.hermes/channel_directory.json`, refreshed periodically. It is used by the `send_message` tool to resolve human-friendly channel names to numeric IDs. Discord and Slack channels are enumerated via their APIs; Telegram, WhatsApp, Signal, Email, and SMS channels are discovered from session history.

### Session Mirroring

When a message is delivered to a platform via `send_message` or cron delivery, the mirror system (`gateway/mirror.py`) appends a record to the target session's transcript so the agent has context about what was sent from outside the session.

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `gateway/run.py` | Main gateway runner, GatewayRunner class |
| `gateway/config.py` | GatewayConfig, PlatformConfig, SessionResetPolicy, StreamingConfig |
| `gateway/session.py` | Session keying and routing logic |
| `gateway/delivery.py` | Outbound delivery to home channels and explicit targets |
| `gateway/pairing.py` | DM pairing code generation and approval |
| `gateway/channel_directory.py` | Channel/chat metadata lookup |
| `gateway/hooks.py` | Gateway lifecycle hook callbacks |
| `gateway/mirror.py` | Session mirroring for remote deliveries |
| `gateway/status.py` | PID file, runtime status, scoped locks |
| `gateway/stream_consumer.py` | Progressive message editing for real-time streaming |
| `gateway/platforms/base.py` | BasePlatformAdapter, MessageEvent, SendResult |
| `gateway/platforms/telegram.py` | TelegramAdapter |
| `gateway/platforms/discord.py` | Discord adapter with VoiceReceiver |
| `gateway/platforms/slack.py` | Slack Socket Mode adapter |
| `gateway/platforms/whatsapp.py` | WhatsApp Baileys bridge adapter |
| `gateway/platforms/signal.py` | Signal via signal-cli HTTP/SSE adapter |
| `gateway/platforms/email.py` | IMAP/SMTP email adapter |
| `gateway/platforms/matrix.py` | Matrix (matrix-nio) adapter |
| `gateway/platforms/mattermost.py` | Mattermost REST API + WebSocket adapter |
| `gateway/platforms/dingtalk.py` | DingTalk Stream Mode adapter |
| `gateway/platforms/homeassistant.py` | Home Assistant WebSocket event adapter |
| `gateway/platforms/api_server.py` | OpenAI-compatible API server |
| `gateway/platforms/sms.py` | Twilio webhook SMS adapter |
| `gateway/platforms/webhook.py` | Generic inbound webhook adapter |

---

## Changelog

### v0.4.0 (v2026.3.23)

- **Auto-reconnect with exponential backoff** — failed platform adapters reconnect automatically without requiring a gateway restart ([PR #2584](https://github.com/NousResearch/hermes-agent/pull/2584)).
- **Gateway prompt caching** — `AIAgent` instances are now cached per session in the gateway. The Anthropic API's prompt cache is preserved across turns: the system prompt, injected skills, and all assistant turns accumulate cache tokens between messages. Practical effect: on Anthropic, turns after the first in a long conversation incur no re-compute cost for the context already seen. Cache signatures include the full auth token so sessions are never shared between credentials ([PR #2282](https://github.com/NousResearch/hermes-agent/pull/2282), [#2284](https://github.com/NousResearch/hermes-agent/pull/2284), [#2361](https://github.com/NousResearch/hermes-agent/pull/2361)).
- **`/approve` and `/deny` commands** — replaced bare `yes`/`no` text in the approval flow with explicit slash commands ([PR #2002](https://github.com/NousResearch/hermes-agent/pull/2002)).
- **`/cost` command** — shows live pricing and usage tracking in gateway mode ([PR #2180](https://github.com/NousResearch/hermes-agent/pull/2180)).
- **6 new platform adapters** — Signal, DingTalk, SMS (Twilio), Mattermost, Matrix, and Webhook added ([PR #2206](https://github.com/NousResearch/hermes-agent/pull/2206), [#1685](https://github.com/NousResearch/hermes-agent/pull/1685), [#1688](https://github.com/NousResearch/hermes-agent/pull/1688), [#1683](https://github.com/NousResearch/hermes-agent/pull/1683), [#2166](https://github.com/NousResearch/hermes-agent/pull/2166)).

### v0.5.0 (v2026.3.28)

- **Telegram Private Chat Topics** — project-based conversations with per-topic skill binding in a single Telegram chat ([PR #3163](https://github.com/NousResearch/hermes-agent/pull/3163)). When a topic's `thread_id` is not found (deleted topic), the adapter falls back to sending without a thread instead of failing ([PR #3390](https://github.com/NousResearch/hermes-agent/pull/3390)).
- **Progress thread fallback scoped to Slack only** — the generic progress-thread fallback that was firing on all platforms is now scoped to Slack only ([PR #3488](https://github.com/NousResearch/hermes-agent/pull/3488)).
- **User-local bin paths in systemd unit** — the generated systemd service unit now includes `~/.local/bin` and other user-local bin paths in `PATH`, so tools installed via `pip install --user` are found when the service runs at boot ([PR #3527](https://github.com/NousResearch/hermes-agent/pull/3527)).
- **AsyncOpenAI/httpx cross-loop deadlock fix** — prevents a deadlock that could occur in gateway mode when `AsyncOpenAI` clients were shared across asyncio event loops ([PR #2701](https://github.com/NousResearch/hermes-agent/pull/2701)).
- **Transcript preserved on `/compress`** — running `/compress` or hygiene compression no longer discards the session transcript ([PR #3556](https://github.com/NousResearch/hermes-agent/pull/3556)).
- **`/verbose` command** — toggle tool output verbosity from any connected chat without restarting the gateway ([PR #3262](https://github.com/NousResearch/hermes-agent/pull/3262)).
- **Matrix added to PLATFORMS dict** — Matrix was missing from the global platforms registry, causing it not to appear in `hermes gateway status` ([PR #3473](https://github.com/NousResearch/hermes-agent/pull/3473)).
