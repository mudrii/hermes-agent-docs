# QQ Bot

Connect Hermes to QQ through the Official QQ Bot API v2. The adapter supports private chats, group `@` mentions, guild messages, file attachments, and voice transcription.

## What it supports

- persistent WebSocket connection to the QQ gateway
- REST replies for text and markdown messages
- image, voice, and file attachment handling
- built-in QQ ASR first, then fallback STT if configured

## Prerequisites

1. Register a QQ Bot application at [q.qq.com](https://q.qq.com).
2. Record your `App ID` and `App Secret`.
3. Enable the required intents for private, group, and guild messages.

## Setup

### Interactive

```bash
hermes setup gateway
```

Choose `QQ Bot` from the platform list.

### Manual

Add these to `~/.hermes/.env`:

```bash
QQ_APP_ID=your-app-id
QQ_CLIENT_SECRET=your-app-secret
```

## Environment variables

| Variable | Description |
|----------|-------------|
| `QQ_APP_ID` | QQ Bot App ID |
| `QQ_CLIENT_SECRET` | QQ Bot App Secret |
| `QQ_HOME_CHANNEL` | Default OpenID used for cron or notifications |
| `QQ_HOME_CHANNEL_NAME` | Display name for the home channel |
| `QQ_ALLOWED_USERS` | Comma-separated allowed user OpenIDs |
| `QQ_ALLOW_ALL_USERS` | Set to `true` to allow all DMs |
| `QQ_MARKDOWN_SUPPORT` | Enable QQ markdown replies |
| `QQ_STT_API_KEY` | Optional speech-to-text provider key |
| `QQ_STT_BASE_URL` | Optional STT endpoint base URL |
| `QQ_STT_MODEL` | STT model name |

## Voice transcription

The adapter tries transcription in two stages:

1. QQ built-in ASR if `asr_refer_text` is present in the message attachment
2. A configured OpenAI-compatible STT provider if QQ does not return text

## Troubleshooting

### Bot disconnects immediately

Check the App ID, secret, configured intents, and whether the bot is still limited to QQ sandbox mode.

### Messages are not delivered

Check allowlists, whether the bot is being `@` mentioned where required, and whether `QQ_HOME_CHANNEL` is valid for proactive delivery.

### Voice messages are not transcribed

Inspect gateway logs for QQ STT fallback errors and verify `QQ_STT_API_KEY`, `QQ_STT_BASE_URL`, and `QQ_STT_MODEL` if you are using a custom provider.

