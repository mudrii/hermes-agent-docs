# Gateway Internals

The messaging gateway is the long-running process that connects Hermes to external platforms. This document covers v0.8.0 (v2026.4.8).

Key files: `gateway/run.py`, `gateway/config.py`, `gateway/session.py`, `gateway/delivery.py`, `gateway/pairing.py`, `gateway/channel_directory.py`, `gateway/hooks.py`, `gateway/mirror.py`, `gateway/platforms/*`

## Core Responsibilities

The gateway process is responsible for:

- Loading configuration from `.env`, `config.yaml`, and `gateway.json`
- Starting platform adapters
- Authorizing users
- Routing incoming events to sessions
- Maintaining per-chat session continuity
- Dispatching messages to `AIAgent`
- Running cron ticks and background maintenance tasks
- Mirroring/proactively delivering output to configured channels

## Config Sources

The gateway has a multi-source config model:

- Environment variables
- `~/.hermes/gateway.json`
- Selected bridged values from `~/.hermes/config.yaml`

## Session Routing

`gateway/session.py` and `GatewayRunner` cooperate to map incoming messages to active session IDs. Session keying can depend on:

- Platform
- User/chat identity
- Thread/topic identity
- Special platform-specific routing behavior

## Session Timeout: Inactivity-Based Eviction (v0.8.0)

Prior to v0.8.0, sessions were evicted based on wall-clock age alone. Starting in v0.8.0, the gateway uses **inactivity-based timeout**: a session is only evicted when the agent has been *idle* for at least `AGENT_TIMEOUT` seconds, not merely old.

The eviction check in `gateway/run.py`:

1. Calls `agent.get_activity_summary()` to retrieve `seconds_since_activity` — the elapsed time since the agent last performed any action (tool call, LLM response, etc.)
2. Evicts when `seconds_since_activity >= timeout` (agent is idle) OR when wall-clock age exceeds `max(10 × timeout, 2 hours)` (safety fallback for agents with no activity tracker)
3. Logs both age and idle time in the eviction notice for diagnostics

This means a long-running tool task (e.g., a multi-hour computation) is not killed mid-execution by a stale timer. Sessions expire when the agent genuinely goes quiet.

## Authorization Layers

The gateway can authorize through:

- Platform allowlists
- Gateway-wide allowlists
- DM pairing flows
- Explicit allow-all settings

Pairing support is implemented in `gateway/pairing.py`.

### Thread-Safe PairingStore (v0.8.0)

`PairingStore` in `gateway/pairing.py` was hardened in v0.8.0 to be safe for concurrent access. Multiple platform adapters can run simultaneously in separate threads sharing one `PairingStore` instance. Writes are atomic: updates go to a temp file that is then renamed into place, preventing partial-read corruption if two adapters write at the same time.

## Delivery Path

Outgoing deliveries are handled by `gateway/delivery.py`, which knows how to:

- Deliver to a home channel
- Resolve explicit targets
- Mirror some remote deliveries back into local history/session tracking

## Duplicate Message Prevention (v0.8.0)

The gateway includes two layers of duplicate suppression added in v0.8.0:

1. **Gateway-level dedup** — the `GatewayRunner` tracks recently processed message IDs per platform. A message that arrives twice (e.g., from a platform retry) is silently dropped on the second arrival.
2. **Partial stream guard** — if a streaming response is interrupted and the platform re-delivers the triggering message, the guard prevents the agent from processing the same input twice and sending a duplicate response.

## Hooks

Gateway events are dispatched through `gateway/hooks.py` using an event-driven registry. Hooks are discovered from `~/.hermes/hooks/` — each hook is a directory containing a `HOOK.yaml` manifest and a `handler.py` with an `async def handle(event_type, context)` function. Errors in hook handlers are caught and logged but never block the main pipeline.

The full set of event types:

| Event | Fires when |
|-------|-----------|
| `gateway:startup` | Gateway process starts |
| `session:start` | New session created (first message of a new session) |
| `session:end` | Session ends (user ran `/new` or `/reset`) |
| `session:reset` | Session reset completed (new session entry created) |
| `agent:start` | Agent begins processing a message |
| `agent:step` | Each turn in the tool-calling loop |
| `agent:end` | Agent finishes processing |
| `command:*` | Any slash command executed (wildcard — matches any `command:…` event) |

A built-in `boot-md` hook (`gateway/builtin_hooks/boot_md.py`) is always active: it runs `~/.hermes/BOOT.md` on `gateway:startup`.

### Writing a Hook

Create `~/.hermes/hooks/my-hook/HOOK.yaml`:

```yaml
name: my-hook
description: Log every agent step
events:
  - agent:step
```

And `~/.hermes/hooks/my-hook/handler.py`:

```python
async def handle(event_type, context):
    print(f"[my-hook] {event_type}: {context}")
```

The `context` dict contains event-specific data (session key, platform, message content, etc.).

## Background Maintenance

The gateway runs maintenance tasks such as:

- Cron ticking
- Cache refreshes
- Session expiry checks (inactivity-based — see above)
- Proactive memory flush before reset/expiry

## Platform Adapters

The gateway ships adapters for 15 platforms. Each adapter translates platform-specific events into a common format that the `GatewayRunner` can route to `AIAgent`.

| Adapter | File | Notes |
|---------|------|-------|
| Telegram | `platforms/telegram.py` | Tier 1. Inline keyboard approval buttons, emoji reactions, group topic skill binding, paginated model picker (v0.8.0) |
| Discord | `platforms/discord.py` | Tier 1. `ignored_channels`, `no_thread_channels` channel controls (v0.8.0) |
| Slack | `platforms/slack.py` | Tier 1. Block Kit approval buttons, thread engagement auto-respond, mrkdwn in `edit_message` (v0.8.0) |
| Matrix | `platforms/matrix.py` | Tier 1 as of v0.8.0. Reactions, read receipts, `MATRIX_REQUIRE_MENTION`, `MATRIX_AUTO_THREAD`, `MATRIX_REACTIONS`, `MATRIX_DEVICE_ID`, E2EE encrypted media event types, Synapse compat |
| WhatsApp | `platforms/whatsapp.py` | Baileys-based bridge. Voice transcription, native media (v0.6.0+) |
| Signal | `platforms/signal.py` | `signal-cli` bridge. Full `MEDIA:` tag delivery — `send_image_file`, `send_voice`, `send_video` (v0.8.0) |
| Email | `platforms/email.py` | IMAP/SMTP. HTML and plain-text delivery |
| Home Assistant | `platforms/homeassistant.py` | WebSocket event subscription + REST service calls |
| SMS (Twilio) | `platforms/sms.py` | Twilio webhook receiver |
| DingTalk | `platforms/dingtalk.py` | Stream Mode WebSocket, no public URL needed |
| Feishu | `platforms/feishu.py` | Interactive card approval buttons, reconnect/ACL fixes (v0.8.0) |
| Mattermost | `platforms/mattermost.py` | File attachment support via `DOCUMENT` message type (v0.8.0) |
| WeCom | `platforms/wecom.py` | WeChat Work enterprise chat |
| Webhook | `platforms/webhook.py` | HTTP outbound. `{__raw__}` template token, `thread_id` passthrough (v0.8.0) |
| API Server | `platforms/api_server.py` | OpenAI-compatible HTTP server for Open WebUI and other clients |

## Profile-Aware Service Units (v0.8.0)

When the gateway is installed as a systemd or launchd service, the generated unit file is now profile-aware: it reads the active Hermes profile from the environment so that users with multiple profiles get the correct `.env` and config loaded at service startup. Previously the unit always loaded the default profile regardless of which profile was active when `hermes gateway install` was run.

## Honcho Interaction

When Honcho is enabled, the gateway keeps persistent Honcho managers aligned with session lifetimes and platform-specific session keys.

### Session Routing

Honcho tools (`honcho_profile`, `honcho_search`, `honcho_context`, `honcho_conclude`) need to execute against the correct user's Honcho session. In a multi-user gateway, the process-global module state is insufficient -- multiple sessions may be active concurrently.

The solution threads session context through the call chain:

```
AIAgent._invoke_tool()
  -> handle_function_call(honcho_manager=..., honcho_session_key=...)
    -> registry.dispatch(**kwargs)
      -> _handle_honcho_*(args, **kw)
        -> _resolve_session_context(**kw)
```

`_resolve_session_context()` checks for `honcho_manager` and `honcho_session_key` in the kwargs first, falling back to the module-global for CLI mode where there is only one session.

### Memory Flush Lifecycle

When a session is reset, resumed, or expires, the gateway flushes memories before discarding context. The flush creates a temporary `AIAgent` with the old session's ID and Honcho session key. After the flush completes, any queued Honcho writes are drained and the gateway-level Honcho manager is shut down for that session key.

## SSL Certificate Auto-Detection

`gateway/run.py` runs `_ensure_ssl_certs()` before any HTTP library is imported. This resolves SSL certificate paths for NixOS, containerized deployments, and other non-standard systems by checking Python defaults, certifi, and common distro/macOS locations.
