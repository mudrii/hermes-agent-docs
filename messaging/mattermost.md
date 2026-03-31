# Mattermost

Hermes Agent integrates with Mattermost as a bot, letting you chat with your AI assistant through direct messages or team channels. Mattermost is a self-hosted, open-source Slack alternative — you run it on your own infrastructure, keeping full control of your data. The bot connects via Mattermost's REST API v4 and WebSocket for real-time events, processes messages through the Hermes Agent pipeline (including tool use, memory, and reasoning), and responds in real time. It supports text, file attachments, images, and slash commands.

No external Mattermost library is required — the adapter uses `aiohttp`, which is already a Hermes dependency.

This document covers v0.2.0 through v0.6.0 (v2026.3.30).

---

## How Hermes Behaves

| Context | Behavior |
|---------|----------|
| **DMs** | Hermes responds to every message. No `@mention` needed. Each DM has its own session. |
| **Public/private channels** | Hermes responds when you `@mention` it. Without a mention, Hermes ignores the message. |
| **Threads** | If `MATTERMOST_REPLY_MODE=thread`, Hermes replies in a thread under your message. Thread context stays isolated from the parent channel. |
| **Shared channels with multiple users** | By default, Hermes isolates session history per user inside the channel. Two people talking in the same channel do not share one transcript. |

---

## Session Model

By default:
- Each DM gets its own session
- Each thread gets its own session namespace
- Each user in a shared channel gets their own session inside that channel

Controlled by `config.yaml`:

```yaml
group_sessions_per_user: true
```

Set to `false` only if you explicitly want one shared conversation for the entire channel. Shared sessions mean users share context growth and token costs, and one person's in-flight run can interrupt another's follow-up.

---

## Step 1: Enable Bot Accounts

Bot accounts must be enabled on your Mattermost server before you can create one.

1. Log in to Mattermost as a **System Admin**
2. Go to **System Console** → **Integrations** → **Bot Accounts**
3. Set **Enable Bot Account Creation** to **true**
4. Click **Save**

---

## Step 2: Create a Bot Account

1. In Mattermost, click the **☰** menu (top-left) → **Integrations** → **Bot Accounts**
2. Click **Add Bot Account**
3. Fill in the details:
   - **Username**: e.g., `hermes`
   - **Display Name**: e.g., `Hermes Agent`
   - **Role**: `Member` is sufficient
4. Click **Create Bot Account**
5. Mattermost will display the **bot token** — **copy it immediately**

The bot token is only displayed once when you create the bot account. If you lose it, you will need to regenerate it from the bot account settings.

You can also use a **personal access token** instead of a bot account. Go to **Profile** → **Security** → **Personal Access Tokens** → **Create Token**. This is useful if you want Hermes to post as your own user.

---

## Step 3: Add the Bot to Channels

The bot needs to be a member of any channel where you want it to respond:

1. Open the channel where you want the bot
2. Click the channel name → **Add Members**
3. Search for your bot username and add it

For DMs, simply open a direct message with the bot — it can respond immediately.

---

## Step 4: Find Your Mattermost User ID

Hermes uses your Mattermost User ID (not your username) to control access. Your User ID is a 26-character alphanumeric string like `3uo8dkh1p7g1mfk49ear5fzs5c`.

To find it:
1. Click your **avatar** (top-left corner) → **Profile**
2. Your User ID is displayed in the profile dialog — click it to copy

Via the API:

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  https://your-mattermost-server/api/v4/users/me | jq .id
```

To get a **Channel ID**: click the channel name → **View Info**. The Channel ID is shown in the info panel. You will need this to set a home channel manually.

---

## Step 5: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **Mattermost** when prompted, then paste your server URL, bot token, and user ID.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

```bash
# Required
MATTERMOST_URL=https://mm.example.com
MATTERMOST_TOKEN=your-bot-token

# Security (recommended)
MATTERMOST_ALLOWED_USERS=3uo8dkh1p7g1mfk49ear5fzs5c

# Multiple allowed users (comma-separated)
# MATTERMOST_ALLOWED_USERS=3uo8dkh1p7g1mfk49ear5fzs5c,8fk2jd9s0a7bncm1xqw4tp6r3e

# Optional: reply mode
# MATTERMOST_REPLY_MODE=thread
```

### Start the Gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

---

## config.yaml Section

```yaml
group_sessions_per_user: true

unauthorized_dm_behavior: pair    # pair | ignore
```

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `MATTERMOST_URL` | Yes | Mattermost server URL (e.g., `https://mm.example.com`) |
| `MATTERMOST_TOKEN` | Yes | Bot account token or personal access token |
| `MATTERMOST_ALLOWED_USERS` | Recommended | Comma-separated 26-character User IDs |
| `MATTERMOST_REPLY_MODE` | No | `thread` or `off` (default: `off`) |
| `MATTERMOST_HOME_CHANNEL` | No | Channel ID for cron job delivery |
| `MATTERMOST_HOME_CHANNEL_NAME` | No | Display name for the home channel (default: `Home`) |
| `MATTERMOST_ALLOW_ALL_USERS` | No | Allow all users (not recommended) |

---

## Home Channel

Designate a home channel where the bot sends proactive messages (cron job output, reminders, notifications).

**Using the slash command:** Type `/sethome` in any Mattermost channel where the bot is present.

**Manual configuration:**

```bash
MATTERMOST_HOME_CHANNEL=abc123def456ghi789jkl012mn
```

Replace the ID with the actual channel ID (click the channel name → View Info → copy the ID).

---

## Message Length Limit

Mattermost posts are limited to 4,000 characters (the server default is 16,383 but 4,000 is the practical limit for readable messages). Longer responses are split at natural boundaries.

---

## Reply Mode

The `MATTERMOST_REPLY_MODE` setting controls how Hermes posts responses:

| Mode | Behavior |
|------|----------|
| `off` (default) | Hermes posts flat messages in the channel, like a normal user |
| `thread` | Hermes replies in a thread under your original message. Keeps channels clean. |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot not responding to messages | Verify bot is a member of the channel (channel name → Add Members). Verify your User ID is in `MATTERMOST_ALLOWED_USERS`. Restart the gateway. |
| 403 Forbidden errors | Check `MATTERMOST_TOKEN` is correct. Ensure bot account is not deactivated. Verify bot has been added to the channel. |
| WebSocket disconnects / reconnection loops | The adapter automatically reconnects with exponential backoff (2s → 60s). Check your server's WebSocket configuration — reverse proxies (nginx, Apache) need WebSocket upgrade headers. |
| "Failed to authenticate" on startup | Verify `MATTERMOST_URL` includes `https://` with no trailing slash. Validate token: `curl -H "Authorization: Bearer YOUR_TOKEN" https://your-server/api/v4/users/me` |
| Bot is offline | Check `hermes gateway` is running. Common causes: wrong URL, expired token, Mattermost server unreachable. |
| "User not allowed" / bot ignores you | Add your User ID to `MATTERMOST_ALLOWED_USERS`. Use the 26-character alphanumeric ID, not your `@username`. |

For nginx WebSocket proxying, ensure your config includes:

```nginx
location /api/v4/websocket {
    proxy_pass http://mattermost-backend;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 600s;
}
```

---

## Security

Always set `MATTERMOST_ALLOWED_USERS` to restrict who can interact with the bot. Without it, the gateway denies all users by default. Only add User IDs of people you trust — authorized users have full access to the agent's capabilities, including tool use and system access.

- Works with both Mattermost Team Edition (free) and Enterprise Edition
- No extra dependencies required — the adapter uses `aiohttp` which is already included with Hermes Agent
- Store the bot token in `~/.hermes/.env` with file permissions `600`

---

## v0.6.0 Changes

### MATTERMOST_REQUIRE_MENTION ([PR #3664](https://github.com/NousResearch/hermes-agent/pull/3664))

By default, the bot requires an `@mention` in non-DM channels. v0.6.0 adds `MATTERMOST_REQUIRE_MENTION` to make this configurable:

```bash
# Disable the mention requirement — respond to all channel messages
MATTERMOST_REQUIRE_MENTION=false
```

| Value | Behavior |
|-------|----------|
| `true` (default) | Bot ignores non-DM messages that do not @mention it |
| `false` | Bot responds to all messages in channels it is a member of |

For more granular control, set `MATTERMOST_FREE_RESPONSE_CHANNELS` to a comma-separated list of channel IDs where the bot responds without a mention, while still requiring a mention everywhere else:

```bash
MATTERMOST_FREE_RESPONSE_CHANNELS=abc123def456,xyz789ghi012
```

---

## Changelog

- **v0.6.0:** `MATTERMOST_REQUIRE_MENTION` env var for configurable mention gating ([PR #3664](https://github.com/NousResearch/hermes-agent/pull/3664)).
- **v0.4.0:** Mattermost adapter added with `@`-mention-only channel filter ([PR #1683](https://github.com/NousResearch/hermes-agent/pull/1683), [#2443](https://github.com/NousResearch/hermes-agent/pull/2443)).
- **v0.4.0:** MIME type detection for media attachments ([PR #2329](https://github.com/NousResearch/hermes-agent/pull/2329)).
- **v0.5.0:** Media download retry added ([PR #3323](https://github.com/NousResearch/hermes-agent/pull/3323)).
- **v0.5.0:** Request timeouts added ([PR #3258](https://github.com/NousResearch/hermes-agent/pull/3258)).
