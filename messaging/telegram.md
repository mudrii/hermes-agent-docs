# Telegram

Hermes integrates with Telegram as a full-featured conversational bot using the [python-telegram-bot](https://python-telegram-bot.org/) library. Once connected, you can chat from any device, send voice memos that get auto-transcribed, receive scheduled task results, and use the agent in group chats or Telegram forum topics.

This document covers v0.2.0 through v0.6.0 (v2026.3.30).

---

## Prerequisites

- A Telegram account
- Access to [@BotFather](https://t.me/BotFather) to create a bot and get a token
- Your numeric Telegram user ID (not your username)

---

## Step 1: Create a Bot via BotFather

1. Open Telegram and message [@BotFather](https://t.me/BotFather)
2. Send `/newbot`
3. Choose a display name (e.g., "Hermes Agent") — this can be anything
4. Choose a username — must be unique and end in `bot` (e.g., `my_hermes_bot`)
5. BotFather replies with your API token in this format:

```
123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
```

Keep this token secret. Anyone with this token can control your bot. If it leaks, revoke it immediately via `/revoke` in BotFather.

---

## Step 2: Customize Your Bot (Optional)

These BotFather commands improve the user experience:

| Command | Purpose |
|---------|---------|
| `/setdescription` | The "What can this bot do?" text shown before a user starts chatting |
| `/setabouttext` | Short text on the bot's profile page |
| `/setuserpic` | Upload an avatar for your bot |
| `/setprivacy` | Control whether the bot sees all group messages |

---

## Step 3: Privacy Mode (Critical for Groups)

Telegram bots have **privacy mode** enabled by default. With privacy mode ON, your bot can only see:
- Messages that start with `/` commands
- Replies directly to the bot's own messages
- Service messages (member joins/leaves, pinned messages)

With privacy mode OFF, the bot receives every message in the group.

### How to disable privacy mode

1. Message **@BotFather**
2. Send `/mybots`
3. Select your bot
4. Go to **Bot Settings → Group Privacy → Turn off**

You must remove and re-add the bot to any group after changing the privacy setting. Telegram caches the privacy state when a bot joins a group.

An alternative: promote the bot to **group admin**. Admin bots always receive all messages regardless of the privacy setting.

---

## Step 4: Find Your User ID

Hermes uses numeric Telegram user IDs to control access. Your user ID is a number like `123456789` — not your username.

- Message [@userinfobot](https://t.me/userinfobot) — it instantly replies with your user ID
- Or message [@get_id_bot](https://t.me/get_id_bot)

---

## Step 5: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **Telegram** when prompted. The wizard asks for your bot token and allowed user IDs.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

```bash
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
TELEGRAM_ALLOWED_USERS=123456789    # Comma-separated for multiple users
```

### Start the Gateway

```bash
hermes gateway
```

The bot comes online within seconds. Send it a message on Telegram to verify.

---

## config.yaml Section

No Telegram-specific section is required in `config.yaml` for basic operation. The following optional keys are available:

```yaml
# Gateway-wide settings that apply to Telegram
group_sessions_per_user: true

# How to handle unauthorized DMs
unauthorized_dm_behavior: pair    # pair | ignore
```

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | Yes | Bot API token from BotFather |
| `TELEGRAM_ALLOWED_USERS` | Recommended | Comma-separated numeric user IDs |
| `TELEGRAM_HOME_CHANNEL` | No | Chat ID for cron job delivery |
| `TELEGRAM_HOME_CHANNEL_NAME` | No | Display name for the home channel (default: `Home`) |
| `TELEGRAM_ALLOW_ALL_USERS` | No | Allow all users (not recommended) |
| `HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS` | No | Album merge delay in seconds (default: `0.8`) |
| `HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS` | No | Text message batch delay in seconds (default: `0.6`) |

---

## Home Channel

Use `/sethome` in any Telegram chat (DM or group) to designate it as the home channel for cron job delivery. Or set it manually:

```bash
TELEGRAM_HOME_CHANNEL=-1001234567890
TELEGRAM_HOME_CHANNEL_NAME="My Notes"
```

Group chat IDs are negative numbers (e.g., `-1001234567890`). Your personal DM chat ID is the same as your user ID.

---

## Platform-Specific Features

### Forum Topics

Telegram supergroups can have forum mode enabled, which creates named topics (threads). The adapter fully supports forum topic session isolation — each topic gets its own session. The `thread_id` from the Telegram message is used as part of the session key.

### Private Chat Topics (v0.5.0)

Private Chat Topics ([PR #3163](https://github.com/NousResearch/hermes-agent/pull/3163)) extend forum-style topics to any supergroup by mapping each topic to a distinct Hermes project. Each topic gets its own isolated session with its own conversation history — sending a message in topic A does not affect the session in topic B within the same group.

**Per-topic skill binding:** You can bind a specific skill to a topic so the agent automatically loads a particular skill context when that topic is active. Configure this in `~/.hermes/gateway.json` or via `hermes gateway setup`:

```json
{
  "platforms": {
    "telegram": {
      "extra": {
        "topic_skills": {
          "42": "code-review",
          "87": "devops"
        }
      }
    }
  }
}
```

Keys are Telegram `thread_id` values (as strings); values are skill names.

**Thread not found fallback:** If a reply targets a topic whose `thread_id` no longer exists (the topic was deleted), the adapter falls back to sending the message without a thread ID instead of raising an error ([PR #3390](https://github.com/NousResearch/hermes-agent/pull/3390)).

### Message Formatting

The adapter converts standard markdown to Telegram MarkdownV2 format. Protected regions (code blocks, inline code) are extracted first so their contents are never modified. Conversions applied:

- `## Header` → bold `*Header*`
- `**bold**` → `*bold*`
- `*italic*` → `_italic_`
- `[link](url)` → MarkdownV2 link syntax
- All remaining special characters are escaped with backslashes

If MarkdownV2 parsing fails, the adapter falls back to plain text.

### Long Message Splitting

Telegram's message limit is 4096 characters. Longer responses are split at natural line breaks, preserving code block boundaries. Each chunk is labeled `(1/N)` when the response spans multiple messages.

### Media Batch Handling

When users send multiple photos (albums), Telegram delivers them as separate update events with a shared `media_group_id`. The adapter buffers them and merges them into a single `MessageEvent` before dispatching, preventing the second photo from being treated as an interrupt. The batch delay is configurable via `HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS` (default: `0.8`).

Similarly, text messages from a Telegram client that splits long messages into multiple parts are aggregated with a configurable delay (`HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS`, default: `0.6`).

### Sticker Analysis

Static stickers (WEBP format) are analyzed via the vision tool to extract a text description. The description is cached by `file_unique_id` so the same sticker is only analyzed once. Animated and video stickers receive a placeholder description noting their emoji.

### Voice Messages

Incoming `.ogg` voice messages are cached locally and transcribed by the configured STT provider. The agent receives the transcription as the message text.

Outgoing TTS audio is sent as native Telegram voice bubbles when the audio is in OGG/Opus format. Other formats (MP3) are sent as audio file attachments. The Edge TTS provider outputs MP3 and requires `ffmpeg` to convert to Opus for bubble delivery:

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt install ffmpeg
```

### Photo and Document Handling

Images sent to the bot are downloaded to `{HERMES_HOME}/image_cache/` using the highest-resolution version available. Documents are downloaded to `{HERMES_HOME}/document_cache/`. Supported document types: `.pdf`, `.md`, `.txt`, `.docx`, `.xlsx`, `.pptx`. Maximum document size: 20 MB (Telegram Bot API limit).

Text files (`.md`, `.txt`) up to 100 KB have their content injected directly into the message text.

### Location Sharing

When a user shares a location pin, the adapter formats the coordinates and a Google Maps link into the message text and prompts the agent to ask about nearby places.

### Token Conflict Protection

The adapter uses a scoped lock file to prevent two gateway instances from polling with the same bot token simultaneously. If a conflict is detected via Telegram's API error, polling stops immediately with `error_code: "telegram_polling_conflict"` and a non-retryable fatal error is recorded.

---

## Exec Approval

When the agent tries to run a potentially dangerous command, it asks for approval in the chat:

> This command is potentially dangerous (recursive delete). Reply `/approve` to allow or `/deny` to block.

Use `/approve` to allow the command or `/deny` to block it. This was changed from bare `yes`/`no` text in v0.4.0 ([PR #2002](https://github.com/NousResearch/hermes-agent/pull/2002)).

---

## Group Chat Usage

- Privacy mode determines what the bot can see (see Step 3)
- With privacy mode on, @mention the bot or reply to its messages
- With privacy mode off (or bot is admin), the bot sees all messages
- `TELEGRAM_ALLOWED_USERS` still applies in groups — only authorized users can trigger the bot

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot not responding at all | Verify `TELEGRAM_BOT_TOKEN` is correct. Check `hermes gateway` logs. |
| Bot responds with "unauthorized" | Your user ID is not in `TELEGRAM_ALLOWED_USERS`. Check with @userinfobot. |
| Bot ignores group messages | Privacy mode is likely on. Disable it (Step 3) or make the bot a group admin. Remove and re-add the bot after changing privacy. |
| Voice messages not transcribed | Verify STT is configured: install `faster-whisper` for local transcription, or set `GROQ_API_KEY` / `VOICE_TOOLS_OPENAI_KEY`. |
| Voice replies are files, not bubbles | Install `ffmpeg` (needed for Edge TTS Opus conversion). |
| Bot token conflict error | Another gateway instance is using this token. Stop the other instance first. |
| Bot token revoked/invalid | Generate a new token via `/revoke` then `/token` in BotFather. Update `~/.hermes/.env`. |

---

## Security

Always set `TELEGRAM_ALLOWED_USERS`. Without it, the gateway denies all users by default. Never share your bot token publicly — if compromised, revoke it immediately via BotFather's `/revoke` command.

---

## Webhook Mode (v0.6.0)

By default the adapter uses **long polling** — it makes outbound requests to Telegram. In v0.6.0, **webhook mode** is available as an alternative. Telegram pushes updates to your HTTP endpoint instead of waiting for the adapter to poll. Webhook mode is faster and better suited for cloud platforms (Fly.io, Railway) where inbound HTTP traffic can wake a suspended container.

### Requirements

- A publicly reachable HTTPS URL pointing to your server
- Port 443, 80, 88, or 8443 (Telegram requirement)

### Enabling webhook mode

Set the following environment variables before starting the gateway:

```bash
TELEGRAM_WEBHOOK_URL=https://your-server.example.com/telegram   # Required
TELEGRAM_WEBHOOK_PORT=8443                                        # Local listen port (default: 8443)
TELEGRAM_WEBHOOK_SECRET=your-random-secret-token                  # Optional but recommended
```

Hermes registers the webhook with Telegram automatically on startup. No manual `setWebhook` call is needed.

### Reverting to polling

Remove `TELEGRAM_WEBHOOK_URL` from your environment and restart the gateway. Hermes will fall back to long polling automatically.

PR [#3880](https://github.com/NousResearch/hermes-agent/pull/3880)

---

## Group Mention Gating (v0.6.0)

v0.6.0 adds configurable **group mention gating** — you can control when the bot responds in group and supergroup chats.

### `TELEGRAM_REQUIRE_MENTION`

Set this to `true` to make the bot ignore group messages unless it is explicitly triggered:

```bash
TELEGRAM_REQUIRE_MENTION=true
```

When `TELEGRAM_REQUIRE_MENTION=true`, the bot responds in groups only when:
- The message @mentions the bot by username
- The message is a reply to one of the bot's own messages
- The message matches a configured regex pattern (see below)
- The message is a slash command (`/...`)
- The chat ID is in `TELEGRAM_FREE_RESPONSE_CHATS`

When `TELEGRAM_REQUIRE_MENTION=false` (default), the bot responds to all messages in groups (subject to `TELEGRAM_ALLOWED_USERS`).

### `TELEGRAM_MENTION_PATTERNS`

Define one or more regex patterns that trigger the bot in groups without an @mention. Useful for wake-word style activation.

```bash
# Single pattern
TELEGRAM_MENTION_PATTERNS='["hey hermes", "hermes,"]'

# JSON array of patterns (case-insensitive)
TELEGRAM_MENTION_PATTERNS='["^hermes[,:]", "\\bhey bot\\b"]'
```

Patterns are matched case-insensitively against the message text and caption.

### `TELEGRAM_FREE_RESPONSE_CHATS`

Comma-separated list of chat IDs where the bot responds to all messages regardless of `TELEGRAM_REQUIRE_MENTION`:

```bash
TELEGRAM_FREE_RESPONSE_CHATS=-1001234567890,-1009876543210
```

### config.yaml equivalent

```yaml
gateway:
  platforms:
    telegram:
      extra:
        require_mention: true
        mention_patterns:
          - "^hermes[,:]"
          - "\\bhey bot\\b"
        free_response_chats:
          - "-1001234567890"
```

PR [#3870](https://github.com/NousResearch/hermes-agent/pull/3870)
