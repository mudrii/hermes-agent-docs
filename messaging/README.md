# Messaging Platforms

The Hermes gateway supports 14 messaging platforms plus an OpenAI-compatible API server (15 adapters total). All platforms share the same session model, authorization system, and chat commands — only the setup steps and platform-specific quirks differ.

This section covers the released messaging and API surface through v0.8.0 (`v2026.4.8`). Individual pages may still call out older release milestones where later releases did not materially change setup.

---

## Platform Overview

| Platform | Protocol | Auth type | Requires public URL | Voice | Streaming |
|----------|----------|-----------|---------------------|-------|-----------|
| [Telegram](telegram.md) | Bot API + polling | Bot token | No | Voice bubbles, TTS, STT | Yes |
| [Discord](discord.md) | WebSocket + REST | Bot token | No | Voice messages, voice channels | Yes |
| [Slack](slack.md) | Socket Mode (WebSocket) | Bot token + App token | No | Audio file attachments | Yes |
| [WhatsApp](whatsapp.md) | Baileys bridge (WhatsApp Web) | QR code scan | No | OGG, MP3 attachments | Yes |
| [Signal](signal.md) | signal-cli HTTP/SSE | QR code link | No | Audio attachments | No |
| [Email](email.md) | IMAP + SMTP | Email credentials | No | — | No |
| [Matrix](matrix.md) | matrix-nio (sync) | Access token or password | No | Audio attachments | Yes |
| [Mattermost](mattermost.md) | REST API + WebSocket | Bot token | No | Audio attachments | Yes |
| [Feishu / Lark](feishu.md) | WebSocket or Webhook | App ID + Secret | No (WebSocket) / Yes (Webhook) | — | No |
| [WeCom](wecom.md) | AI Bot WebSocket | Bot ID + Secret | No | — | No |
| [DingTalk](dingtalk.md) | Stream Mode (WebSocket) | Client ID + Secret | No | — | No |
| [Home Assistant](homeassistant.md) | WebSocket + REST | Long-lived access token | No | — | No |
| [SMS (Twilio)](sms.md) | Twilio webhook | Account SID + Auth token | Yes | — | No |
| [Open WebUI / API Server](open-webui.md) | OpenAI-compatible HTTP | Optional API key | No | — | No |

Streaming uses the `edit` transport: the bot sends an initial message then progressively edits it as tokens arrive, showing a cursor indicator. Platforms without `edit_message` support fall back gracefully to sending the final response in one message.

---

## Quick Setup

The interactive setup wizard handles all platforms:

```bash
hermes gateway setup
```

This walks through platform selection, credential entry, and writes configuration to `~/.hermes/.env`. After setup, start the gateway with `hermes gateway`.

---

## Common Concepts

### Authorization

Every platform uses the same authorization model. By default, the gateway denies all users. To grant access:

1. **Allowlist** — Set `<PLATFORM>_ALLOWED_USERS` in `~/.hermes/.env` with comma-separated user IDs
2. **DM Pairing** — Unknown users receive a one-time pairing code you approve with `hermes pairing approve <platform> <code>`
3. **Per-platform allow-all** — Set `<PLATFORM>_ALLOW_ALL_USERS=true` (e.g., `DISCORD_ALLOW_ALL_USERS=true`)
4. **Global allow-all** — Set `GATEWAY_ALLOW_ALL_USERS=true` (not recommended for bots with terminal access)

The `unauthorized_dm_behavior` setting controls what happens when an unlisted user DMs the bot:
- `pair` (default) — send the user a pairing code
- `ignore` — silently ignore the message

### Home Channels

A home channel is the default destination for cron job output and proactive notifications. Set it in the chat using `/sethome`, or configure it in `~/.hermes/.env`:

```bash
TELEGRAM_HOME_CHANNEL=-1001234567890
DISCORD_HOME_CHANNEL=123456789012345678
```

### Session Isolation

By default, each user in a shared group or channel gets their own isolated session. This is controlled by `group_sessions_per_user: true` in `~/.hermes/config.yaml`. See the [Gateway Architecture](../gateway.md) document for details.

### Chat Commands

All platforms support the same slash commands: `/new`, `/reset`, `/model`, `/status`, `/stop`, `/sethome`, `/compress`, `/usage`, `/background`, and more. See the [Gateway Architecture](../gateway.md) document for the full command reference.

### Voice Support

Platforms that support voice messages automatically transcribe incoming audio using the configured STT provider:
- `local` — `faster-whisper` on the machine running Hermes (no API key required)
- `groq` — Groq Whisper (requires `GROQ_API_KEY`)
- `openai` — OpenAI Whisper (requires `VOICE_TOOLS_OPENAI_KEY`)

TTS replies are delivered natively on Telegram (voice bubbles) and as audio files on other platforms.

---

## Platform-Specific Toolsets

Each platform has a named toolset that determines which tools the agent can use:

| Platform | Toolset | Special tools |
|----------|---------|---------------|
| Telegram | `hermes-telegram` | — |
| Discord | `hermes-discord` | — |
| Slack | `hermes-slack` | — |
| WhatsApp | `hermes-whatsapp` | — |
| Signal | `hermes-signal` | — |
| SMS | `hermes-sms` | — |
| Email | `hermes-email` | — |
| Home Assistant | `hermes-homeassistant` | `ha_list_entities`, `ha_get_state`, `ha_call_service`, `ha_list_services` |
| Mattermost | `hermes-mattermost` | — |
| Matrix | `hermes-matrix` | — |
| Feishu / Lark | `hermes-feishu` | — |
| WeCom | `hermes-wecom` | — |
| DingTalk | `hermes-dingtalk` | — |
| API Server | `hermes` | — |

All toolsets include full terminal access, file operations, web search, and other standard tools unless specifically disabled via `hermes tools`.

---

## Platform Guides

- [Telegram](telegram.md) — Bot API, forum topics, voice bubbles, sticker analysis
- [Discord](discord.md) — Server bots, voice channels, auto-threading, per-user session isolation
- [Slack](slack.md) — Socket Mode, thread replies, event subscriptions
- [WhatsApp](whatsapp.md) — Baileys bridge, QR pairing, bot vs self-chat modes
- [Signal](signal.md) — signal-cli daemon, E2E encryption, group access control
- [Email](email.md) — IMAP/SMTP polling, thread headers, attachment handling
- [Matrix](matrix.md) — Any homeserver, E2EE, room auto-join
- [Mattermost](mattermost.md) — Self-hosted, thread reply mode, REST API
- [Feishu / Lark](feishu.md) — WebSocket or Webhook, message cards, group @mention gating, interactive card approval buttons (v0.6.0+)
- [WeCom](wecom.md) — AI Bot WebSocket, Enterprise WeChat, configurable DM/group policies (v0.6.0)
- [DingTalk](dingtalk.md) — Stream Mode, no public URL required
- [Home Assistant](homeassistant.md) — State change events, smart home tools
- [SMS (Twilio)](sms.md) — Twilio webhook, plain text, 1600 char limit
- [Open WebUI / API Server](open-webui.md) — OpenAI-compatible frontend, Docker setup
