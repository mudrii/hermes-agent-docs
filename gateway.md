# Gateway — Multi-Platform Messaging

The Hermes gateway is a long-running process that connects Hermes Agent to messaging platforms. A single gateway instance can serve all platforms simultaneously, routing messages to the same AIAgent core.

---

## Supported Platforms

| Platform | Auth | Message Types | Session Key |
|----------|------|---------------|-------------|
| **Telegram** | Bot Token | Text, images, audio, documents, voice, stickers | `telegram:{chat_id}` |
| **Discord** | Bot Token (OAuth) | Text, images, audio, files, threads | `discord:{channel_id}` |
| **Slack** | 2-Token (Socket Mode) | Text, images, audio, files, threads, slash commands | `slack:{channel_id}` |
| **WhatsApp** | Node.js Bridge (Baileys) | Text, images, audio, documents, voice | `whatsapp:{phone}` |
| **Signal** | signal-cli daemon | Text, images, audio, documents, voice | `signal:{phone}` |
| **Email** | IMAP/SMTP | Text, HTML, MIME attachments | `email:{address}` |
| **Home Assistant** | Long-Lived Token | State changes, entity updates | `homeassistant:{entity_id}` |

---

## Quick Start

### 1. Configure Platform Credentials

Set environment variables in `~/.hermes/.env`:

```bash
# Telegram
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrstUVWxyz
TELEGRAM_HOME_CHANNEL=123456789   # Your Telegram user ID

# Discord
DISCORD_BOT_TOKEN=MTIz...
DISCORD_HOME_CHANNEL=987654321    # Channel ID for cron deliveries

# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
SLACK_HOME_CHANNEL=C0123456789

# WhatsApp (requires Node.js + Baileys)
WHATSAPP_ENABLED=true
WHATSAPP_ALLOWED_USERS=+1234567890

# Signal (requires signal-cli running on localhost)
SIGNAL_HTTP_URL=http://localhost:8080
SIGNAL_ACCOUNT=+1234567890        # Your Signal phone number
SIGNAL_HOME_CHANNEL=+0987654321

# Email
EMAIL_ADDRESS=bot@example.com
EMAIL_PASSWORD=...
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com

# Home Assistant
HASS_TOKEN=eyJ0eXAiOiJKV1Qi...
HASS_URL=http://homeassistant.local:8123
```

### 2. Run Gateway

```bash
hermes gateway run       # Foreground
hermes gateway start     # Background service
hermes gateway status    # Check status
hermes gateway stop      # Stop service
```

### 3. Interactive Setup Wizard

```bash
hermes gateway setup
```

Walks through each platform's token setup, tests connectivity, and writes `~/.hermes/gateway.json`.

---

## Platform Setup Details

### Telegram

1. Talk to [@BotFather](https://t.me/BotFather) on Telegram
2. Create a bot: `/newbot`
3. Copy the token to `TELEGRAM_BOT_TOKEN`
4. Get your user ID from [@userinfobot](https://t.me/userinfobot)
5. Set `TELEGRAM_HOME_CHANNEL` to your user ID

**Forum topic support:** Messages in forum topics create thread-isolated sessions.

**Voice messages:** Automatically transcribed via faster-whisper.

**Documents:** PDF and text documents processed automatically.

### Discord

1. Create a bot at [discord.com/developers/applications](https://discord.com/developers/applications)
2. Enable **Message Content Intent** under Bot → Privileged Gateway Intents
3. Invite bot with: `applications.commands`, `bot` scopes + `Send Messages`, `Read Message History`, `Attach Files` permissions
4. Set `DISCORD_BOT_TOKEN`

**Mention requirement:** By default, the bot only responds when `@mentioned`. Configure free-response channels:

```json
{
  "discord": {
    "require_mention": true,
    "free_response_channels": ["987654321"],
    "auto_thread": false
  }
}
```

**Thread support:** Use `auto_thread: true` to auto-create threads for each conversation.

### Slack

1. Create a Slack app at [api.slack.com/apps](https://api.slack.com/apps)
2. Enable **Socket Mode** (generates `xapp-` App Token)
3. Add OAuth scopes: `app_mentions:read`, `channels:history`, `chat:write`, `files:write`
4. Install app to workspace
5. Set `SLACK_BOT_TOKEN` (`xoxb-...`) and `SLACK_APP_TOKEN` (`xapp-...`)

**Threading:** Replies are sent in threads automatically.

**Slash commands:** Configure custom slash commands in the Slack app settings.

### WhatsApp

Uses [Baileys](https://github.com/WhiskeySockets/Baileys) (Node.js library for WhatsApp Web protocol).

1. Ensure Node.js ≥ 18 is installed
2. Set `WHATSAPP_ENABLED=true`
3. On first start, scan the QR code shown in the terminal
4. Optionally restrict to specific phone numbers via `WHATSAPP_ALLOWED_USERS`

**Note:** Baileys uses the WhatsApp Web protocol for personal accounts. Use with a dedicated WhatsApp account.

### Signal

Uses [signal-cli](https://github.com/AsamK/signal-cli):

1. Install signal-cli
2. Register/link your Signal account: `signal-cli -a +1234567890 register`
3. Start the HTTP daemon: `signal-cli -a +1234567890 daemon --http localhost:8080`
4. Set `SIGNAL_HTTP_URL` and `SIGNAL_ACCOUNT`

**E164 phone numbers** required for all Signal contacts (e.g., `+12125550000`).

### Email (IMAP/SMTP)

1. Set IMAP credentials (incoming mail)
2. Set SMTP credentials (outgoing mail)
3. For Gmail: create an [App Password](https://myaccount.google.com/apppasswords) if using 2FA

```bash
EMAIL_ADDRESS=bot@example.com
EMAIL_PASSWORD=abcd-efgh-ijkl-mnop   # App password for Gmail
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_IMAP_PORT=993    # Optional, defaults to 993
EMAIL_SMTP_PORT=587    # Optional, defaults to 587
```

### Home Assistant

1. Create a Long-Lived Access Token in HA → Profile → Long-Lived Access Tokens
2. Set `HASS_TOKEN` and `HASS_URL`

**Event filtering:** Configure which domains and entities trigger the bot:

```json
{
  "homeassistant": {
    "watch_domains": ["sensor", "binary_sensor"],
    "watch_entities": ["sensor.temperature_living_room"],
    "ignore_entities": ["sensor.uptime"]
  }
}
```

---

## Session Management

### Session Keys

Each conversation gets a unique session key:

```
telegram:123456789           # DM session
telegram:123456789:987       # Forum topic thread
discord:987654321            # Channel session
discord:987654321:thread_123 # Thread session
```

Sessions are stored in SQLite (`~/.hermes/state.db`) and persist across gateway restarts.

### Session Reset Policy

Control when sessions reset (start fresh conversation):

```json
{
  "default_reset_policy": {
    "mode": "both",          // both | daily | idle | none
    "at_hour": 0,            // Hour for daily reset (0-23)
    "idle_minutes": 60       // Minutes of inactivity before idle reset
  }
}
```

Per-platform and per-chat-type overrides:

```json
{
  "reset_by_platform": {
    "telegram": { "mode": "idle", "idle_minutes": 30 }
  },
  "reset_by_type": {
    "group": { "mode": "daily", "at_hour": 6 }
  }
}
```

**Reset triggers:** Users can type `/new` or `/reset` to reset their session manually.

### Group Sessions

```json
{
  "group_sessions_per_user": true
}
```

When `true`, each user in a group chat gets their own session (default). When `false`, all users share one session.

---

## Message Delivery

### Delivery Targets

Messages can be delivered to:

| Target | Description |
|--------|-------------|
| `"origin"` | Back to source chat |
| `"local"` | Local file only (`~/.hermes/cron/output/`) |
| `"telegram"` | Telegram home channel |
| `"telegram:123456"` | Specific Telegram chat |
| `"discord"` | Discord home channel |
| `"discord:channel_id"` | Specific Discord channel |
| `"slack"` | Slack home channel |
| `"email"` | Email home address |

Used by cron jobs and agent-to-platform messaging.

### Message Limits

| Platform | Max Characters |
|----------|----------------|
| Telegram | 4,096 |
| Discord | 2,000 |
| Slack | 39,000 |
| WhatsApp | 65,536 |
| Signal | 64,000 |

Messages exceeding limits are chunked automatically.

---

## Voice Transcription

When a user sends a voice message, the gateway automatically transcribes it via faster-whisper before passing text to the agent.

Configure STT in `~/.hermes/config.yaml`:

```yaml
stt:
  enabled: true
  provider: "local"           # local | groq | openai
  local:
    model: "base"             # tiny | base | small | medium | large-v3
```

---

## Quick Commands

Define short commands users can send in chat:

```json
{
  "quick_commands": {
    "/standup": "Prepare my daily standup based on recent work",
    "/brief": "Give me a morning briefing: weather, calendar, emails",
    "/review": "Review recent code changes"
  }
}
```

---

## Service Management (Linux)

```bash
# Install as systemd service
hermes gateway install

# Enable auto-start
systemctl enable hermes-gateway

# Check logs
journalctl -u hermes-gateway -f
```

The systemd unit restarts automatically on failure.

---

## Channel Directory

The gateway maintains a cache of all reachable channels and contacts:

```
~/.hermes/channel_directory.json
```

Refreshed every 5 minutes. Sources:
- Discord: enumerate all guilds and text channels
- Slack: list channels via API
- Telegram/WhatsApp/Signal: built from session history

Used by the `send_message` tool to resolve platform targets.
