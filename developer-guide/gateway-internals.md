# Gateway Internals

The messaging gateway is the long-running process that connects Hermes to external platforms.

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

## Authorization Layers

The gateway can authorize through:

- Platform allowlists
- Gateway-wide allowlists
- DM pairing flows
- Explicit allow-all settings

Pairing support is implemented in `gateway/pairing.py`.

## Delivery Path

Outgoing deliveries are handled by `gateway/delivery.py`, which knows how to:

- Deliver to a home channel
- Resolve explicit targets
- Mirror some remote deliveries back into local history/session tracking

## Hooks

Gateway events emit hook callbacks through `gateway/hooks.py`. Hooks are local trusted Python code and can observe or extend gateway lifecycle events. Four hooks are active:

| Hook | Fires when |
|------|-----------|
| `pre_llm_call` | Immediately before each API call to the LLM |
| `post_llm_call` | Immediately after the LLM response is received |
| `on_session_start` | When a new session is started |
| `on_session_end` | When a session ends (clean exit, reset, or timeout) |

## Background Maintenance

The gateway runs maintenance tasks such as:

- Cron ticking
- Cache refreshes
- Session expiry checks
- Proactive memory flush before reset/expiry

## Platform Adapters

The gateway ships adapters for 8 platforms (Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant, API server). Each adapter translates platform-specific events into a common format that the `GatewayRunner` can route to `AIAgent`.

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
