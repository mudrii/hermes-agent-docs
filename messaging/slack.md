# Slack

Hermes connects to Slack using Socket Mode — a WebSocket-based connection that does not require a public HTTP endpoint. The adapter uses the `slack-bolt` Python SDK with `slack_sdk`. Hermes works behind firewalls, on laptops, and on private servers without any tunnel or port forwarding.

Classic Slack apps (RTM API) were fully deprecated in March 2025. Hermes uses the modern Bolt SDK with Socket Mode.

This document covers the released Slack adapter surface through v0.8.0 (`v2026.4.8`).

---

## Prerequisites

- A Slack workspace where you have admin rights (or can request a bot account)
- Two tokens: a Bot Token (`xoxb-`) and an App-Level Token (`xapp-`)
- Your Slack Member ID for the allowlist

---

## How Hermes Behaves

| Context | Behavior |
|---------|----------|
| **DMs** | Hermes responds to every message. No @mention needed. Each DM has its own session. |
| **Channels** | Hermes **only responds when @mentioned**. Channel replies go into a thread attached to your message. |
| **Threads** | If you @mention Hermes inside an existing thread, it replies in that same thread. |

---

## Step 1: Create a Slack App

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App**
3. Choose **From scratch**
4. Enter an app name (e.g., "Hermes Agent") and select your workspace
5. Click **Create App**

---

## Step 2: Configure Bot Token Scopes

Navigate to **Features → OAuth & Permissions** in the sidebar. Under **Scopes → Bot Token Scopes**, add:

| Scope | Purpose |
|-------|---------|
| `chat:write` | Send messages as the bot |
| `app_mentions:read` | Detect when @mentioned in channels |
| `channels:history` | Read messages in public channels the bot is in |
| `channels:read` | List and get info about public channels |
| `groups:history` | Read messages in private channels the bot is invited to |
| `im:history` | Read direct message history |
| `im:read` | View basic DM info |
| `im:write` | Open and manage DMs |
| `users:read` | Look up user information |
| `files:write` | Upload files (images, audio, documents) |

Without `channels:history` and `groups:history`, the bot will not receive messages in channels — it will only work in DMs. These are the most commonly missed scopes.

Optional: `groups:read` — list and get info about private channels.

---

## Step 3: Enable Socket Mode

1. In the sidebar, go to **Settings → Socket Mode**
2. Toggle **Enable Socket Mode** to ON
3. You'll be prompted to create an **App-Level Token**:
   - Name it anything (e.g., `hermes-socket`)
   - Add the **`connections:write`** scope
   - Click **Generate**
4. Copy the token — it starts with `xapp-`. This is your `SLACK_APP_TOKEN`

---

## Step 4: Subscribe to Events

1. In the sidebar, go to **Features → Event Subscriptions**
2. Toggle **Enable Events** to ON
3. Under **Subscribe to bot events**, add:

| Event | Required? | Purpose |
|-------|-----------|---------|
| `message.im` | Yes | Bot receives direct messages |
| `message.channels` | Yes | Bot receives messages in public channels |
| `message.groups` | Recommended | Bot receives messages in private channels |
| `app_mention` | Yes | Prevents Bolt SDK errors on @mention |

4. Click **Save Changes**

If the bot works in DMs but not in channels, you almost certainly forgot to add `message.channels` (for public channels) and/or `message.groups` (for private channels).

---

## Step 5: Install App to Workspace

1. In the sidebar, go to **Settings → Install App**
2. Click **Install to Workspace**
3. Review the permissions and click **Allow**
4. Copy the **Bot User OAuth Token** starting with `xoxb-`. This is your `SLACK_BOT_TOKEN`

After changing scopes or event subscriptions, you must reinstall the app for the changes to take effect.

---

## Step 6: Find Your Member ID

To find your Slack Member ID:

1. In Slack, click on your name or avatar
2. Click **View full profile**
3. Click the **⋮** (more) button
4. Select **Copy member ID**

Member IDs look like `U01ABC2DEF3`.

---

## Step 7: Configure Hermes

Add to `~/.hermes/.env`:

```bash
# Required
SLACK_BOT_TOKEN=xoxb-your-bot-token-here
SLACK_APP_TOKEN=xapp-your-app-token-here
SLACK_ALLOWED_USERS=U01ABC2DEF3      # Comma-separated Member IDs

# Optional
SLACK_HOME_CHANNEL=C01234567890
```

For the released v0.6.0 multi-workspace flow, Hermes can also load additional workspace bot tokens from `~/.hermes/slack_tokens.json`. `SLACK_BOT_TOKEN` remains the primary token used to bootstrap Socket Mode; extra workspace tokens are appended from the OAuth token file when present.

Or use interactive setup:

```bash
hermes gateway setup    # Select Slack when prompted
```

Start the gateway:

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

---

## Step 8: Invite the Bot to Channels

After starting the gateway, invite the bot to any channel where you want it to respond:

```
/invite @Hermes Agent
```

The bot will not automatically join channels.

---

## config.yaml Section

```yaml
group_sessions_per_user: true

unauthorized_dm_behavior: pair    # pair | ignore

# Slack-specific settings (in gateway.json platforms.slack.extra or config.yaml)
slack:
  reply_in_thread: true      # true (default) | false — thread replies vs flat messages
  reply_broadcast: false     # Also post thread replies to the main channel
```

- `group_sessions_per_user: true` controls whether multiple users in the same channel share a session or have isolated sessions
- `reply_broadcast: false` (default) keeps replies only in the thread; set to `true` to also post the first chunk of a thread reply to the main channel (useful for visibility in busy channels)
- `reply_in_thread: true` (default) replies in a thread attached to the user's message; set to `false` to post flat replies in the channel instead of threading

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `SLACK_BOT_TOKEN` | Yes | Bot User OAuth Token (`xoxb-...`) |
| `SLACK_APP_TOKEN` | Yes | App-Level Token for Socket Mode (`xapp-...`) |
| `SLACK_ALLOWED_USERS` | Recommended | Comma-separated Slack Member IDs |
| `SLACK_HOME_CHANNEL` | No | Channel ID for cron job delivery |
| `SLACK_HOME_CHANNEL_NAME` | No | Display name for the home channel |
| `SLACK_ALLOW_ALL_USERS` | No | Allow all users (not recommended) |

---

## Home Channel

Set `SLACK_HOME_CHANNEL` to a channel ID where Hermes delivers scheduled messages and cron job results.

To find a Channel ID:
1. Right-click the channel name in Slack
2. Click **View channel details**
3. Scroll to the bottom — the Channel ID is shown there

The bot must be invited to the home channel before it can deliver messages.

---

## Thread Behavior

In channels, Hermes always replies in a thread attached to your message. Tool progress messages and the final response all appear in the same thread.

The v0.3.0 thread handling overhaul ensures that:
- Progress messages go to the correct thread
- Responses are threaded to the originating message
- Session isolation correctly scopes to thread level when messages come from the same channel thread

---

## Voice Support

- **Incoming voice/audio** — automatically transcribed using the configured STT provider: local `faster-whisper`, Groq Whisper (`GROQ_API_KEY`), or OpenAI Whisper (`VOICE_TOOLS_OPENAI_KEY`)
- **Outgoing TTS** — sent as audio file attachments via the `files:write` scope

---

## Platform-Specific Limits

- Maximum message length: 39,000 characters (Slack API allows 40,000; the adapter leaves a margin). This was fixed in v0.3.0 from an incorrect 3,900 character limit.
- File uploads fall back to posting the file path as text if the upload fails, preserving thread context.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot doesn't respond to DMs | Verify `message.im` is in your event subscriptions and the app is reinstalled |
| Bot works in DMs but not channels | Add `message.channels` and `message.groups` to event subscriptions, reinstall the app, and invite the bot to the channel |
| Bot doesn't respond to @mentions | Check `message.channels` event is subscribed; bot must be invited; `channels:history` scope must be added; reinstall after scope changes |
| Bot ignores private channels | Add `message.groups` event and `groups:history` scope, then reinstall and `/invite` the bot |
| "not_authed" or "invalid_auth" errors | Regenerate your Bot Token and App Token, update `.env` |
| "missing_scope" error | Add the required scope in OAuth & Permissions, then reinstall the app |
| Changed scopes/events but nothing changed | You must reinstall the app to your workspace after any scope or event subscription change |

### Quick Checklist for Channel Issues

1. `message.channels` event is subscribed (public channels)
2. `message.groups` event is subscribed (private channels)
3. `app_mention` event is subscribed
4. `channels:history` scope is added (public channels)
5. `groups:history` scope is added (private channels)
6. App was reinstalled after adding scopes/events
7. Bot was invited to the channel (`/invite @Hermes Agent`)
8. You are @mentioning the bot in your message

---

## Security

Always set `SLACK_ALLOWED_USERS` with the Member IDs of authorized users. Without this setting, the gateway will deny all messages by default.

- Store tokens in `~/.hermes/.env` with file permissions `600`
- Socket Mode means no public endpoint is exposed — one less attack surface
- Rotate tokens periodically via the Slack app settings

---

## Multi-Workspace OAuth (v0.6.0)

v0.6.0 allows a single Hermes gateway to connect to **multiple Slack workspaces** simultaneously. Each workspace gets its own bot token, and the correct token is resolved per incoming event.

### Token file

Save OAuth tokens for each workspace in `~/.hermes/slack_tokens.json`:

```json
{
  "T01ABC2DEF": {
    "token": "xoxb-workspace-one-token",
    "team_name": "My Company"
  },
  "T09XYZ3GHI": {
    "token": "xoxb-workspace-two-token",
    "team_name": "Side Project"
  }
}
```

Keys are Slack Team IDs (the `T...` value visible in the workspace URL). Hermes loads this file on startup and merges the tokens with any bot tokens already configured via `SLACK_BOT_TOKEN`.

### config.yaml / env var

`SLACK_BOT_TOKEN` (single-workspace) continues to work unchanged. To add workspaces, create the token file above — no additional config key is needed.

```bash
# Single workspace (existing behavior)
SLACK_BOT_TOKEN=xoxb-single-workspace-token
SLACK_APP_TOKEN=xapp-your-app-token

# Multi-workspace: add slack_tokens.json and keep SLACK_APP_TOKEN
```

Tokens from the file are merged with `SLACK_BOT_TOKEN`. Duplicate tokens are ignored automatically.

### Scoped token locking

Each bot token is protected by a machine-local lock file at `{XDG_STATE_HOME}/hermes/gateway-locks/` to prevent two gateway instances from using the same Slack token simultaneously. If a second gateway attempts to start with a token already locked, it logs an error and refuses to connect that workspace.

### Team-aware client routing

When multiple workspaces are connected, the adapter maintains a **team-to-client mapping**. Each inbound Socket Mode event includes a `team_id`; the adapter resolves the correct `WebClient` for that team before sending any API call. This ensures replies, file uploads, and reactions always target the correct workspace.

PR [#3903](https://github.com/NousResearch/hermes-agent/pull/3903)

---

## v0.8.0 Changes

### Thread engagement ([PR #5897](https://github.com/NousResearch/hermes-agent/pull/5897))

The Slack adapter now automatically responds in threads that the bot started, and in threads where it has been mentioned, without requiring a new `@mention` for every reply. This makes multi-turn conversations in Slack threads much more natural.

### Approval buttons ([PR #5890](https://github.com/NousResearch/hermes-agent/pull/5890))

Dangerous command approval prompts now render as **Block Kit interactive buttons** — click **Approve** or **Deny** directly in the Slack message. Thread context is preserved during the approval wait, so follow-up messages land in the right thread.

### `mrkdwn` formatting in `edit_message` ([PR #5733](https://github.com/NousResearch/hermes-agent/pull/5733))

When streaming, progressive message edits now convert standard markdown to Slack `mrkdwn` format (bold, italic, code, etc.) so the streaming response renders correctly in the Slack client.

---

## Changelog

- **v0.8.0:** Auto-respond in bot-started and mentioned threads without requiring `@mention` per reply ([PR #5897](https://github.com/NousResearch/hermes-agent/pull/5897)).
- **v0.8.0:** Native Block Kit approval buttons for dangerous command prompts ([PR #5890](https://github.com/NousResearch/hermes-agent/pull/5890)).
- **v0.8.0:** `mrkdwn` formatting applied in `edit_message` for streaming responses ([PR #5733](https://github.com/NousResearch/hermes-agent/pull/5733)).
- **v0.6.0:** Multi-workspace OAuth token file support ([PR #3903](https://github.com/NousResearch/hermes-agent/pull/3903)).
- **v0.5.0:** Tool call progress messages are now sent to the correct Slack thread ([PR #3063](https://github.com/NousResearch/hermes-agent/pull/3063)).
- **v0.5.0:** Progress thread fallback logic is now scoped to Slack only and no longer fires on other platforms ([PR #3488](https://github.com/NousResearch/hermes-agent/pull/3488)).
- **v0.5.0:** Media download retry added ([PR #3323](https://github.com/NousResearch/hermes-agent/pull/3323)).
