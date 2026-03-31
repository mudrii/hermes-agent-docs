# Discord

Hermes integrates with Discord as a bot using the `discord.py` library. The bot receives messages, processes them through the full Hermes Agent pipeline (tool use, memory, reasoning), and responds in real time. It supports text, voice messages, file attachments, images, video, and voice channels.

This document covers v0.2.0 through v0.6.0 (v2026.3.30).

---

## Prerequisites

- A Discord account with access to the [Discord Developer Portal](https://discord.com/developers/applications)
- Permission to invite bots to a server (Manage Server permission)
- Your Discord User ID (found via Developer Mode)

---

## How Hermes Behaves

| Context | Behavior |
|---------|----------|
| **DMs** | Hermes responds to every message. No `@mention` needed. Each DM has its own session. |
| **Server channels** | By default, Hermes only responds when you `@mention` it. |
| **Free-response channels** | Specific channels can be made mention-free with `DISCORD_FREE_RESPONSE_CHANNELS`. |
| **Threads** | Hermes replies in the same thread. Mention rules apply unless the thread is configured as free-response. Threads have isolated session history from the parent channel. |
| **Shared channels** | By default, Hermes isolates session history per user. Alice and Bob in the same channel each have separate conversation histories. |

---

## Step 1: Create a Discord Application

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications) and sign in
2. Click **New Application** in the top-right corner
3. Enter a name (e.g., "Hermes Agent") and accept the Developer Terms of Service
4. Click **Create**

Note the **Application ID** — you'll need it to build the invite URL.

---

## Step 2: Create the Bot

1. In the left sidebar, click **Bot**
2. Discord automatically creates a bot user for your application
3. Under **Authorization Flow**:
   - Set **Public Bot** to **OFF** — prevents others from inviting your bot
   - Leave **Require OAuth2 Code Grant** set to **OFF**

---

## Step 3: Enable Privileged Gateway Intents (Critical)

On the **Bot** page, scroll down to **Privileged Gateway Intents**:

| Intent | Required? |
|--------|-----------|
| Presence Intent | Optional |
| Server Members Intent | Optional |
| **Message Content Intent** | **Required** |

Enable **Message Content Intent** by toggling it ON and clicking **Save Changes**.

Without Message Content Intent, the bot receives message events but the message text is empty. This is the most common reason a Discord bot appears online but never responds.

If your bot is in 100 or more servers, Discord requires submitting a verification application to use privileged intents. For personal use, this is not a concern.

---

## Step 4: Get the Bot Token

1. On the **Bot** page, click **Reset Token**
2. Enter your 2FA code if required
3. Copy the displayed token immediately — it is only shown once

Store the token in a password manager. Anyone with this token has full control of your bot. Never commit it to Git.

---

## Step 5: Generate the Invite URL

### Option A: Using the Installation Tab (Recommended)

1. In the left sidebar, click **Installation**
2. Under **Installation Contexts**, enable **Guild Install**
3. For **Install Link**, select **Discord Provided Link**
4. Under **Default Install Settings** for Guild Install:
   - **Scopes**: `bot` and `applications.commands`
   - **Permissions**: see the required permissions below

### Option B: Manual URL

```
https://discord.com/oauth2/authorize?client_id=YOUR_APP_ID&scope=bot+applications.commands&permissions=274878286912
```

Replace `YOUR_APP_ID` with the Application ID from Step 1.

### Required Permissions

| Permission | Purpose |
|------------|---------|
| View Channels | See the channels the bot has access to |
| Send Messages | Respond to messages |
| Embed Links | Format rich responses |
| Attach Files | Send images, audio, and file outputs |
| Read Message History | Maintain conversation context |

### Recommended Additional Permissions

| Permission | Purpose |
|------------|---------|
| Send Messages in Threads | Respond in thread conversations |
| Add Reactions | React to messages for acknowledgment |

Permission integers: minimal `117760`, recommended `274878286912`.

---

## Step 6: Invite to Your Server

1. Open the invite URL in your browser
2. Select your server from the dropdown
3. Click **Continue**, then **Authorize**
4. Complete the CAPTCHA if prompted

The bot appears in your server's member list (offline until you start the gateway).

---

## Step 7: Find Your Discord User ID

1. Open Discord (desktop or web)
2. Go to **Settings → Advanced → toggle Developer Mode ON**
3. Close settings
4. Right-click your own username → **Copy User ID**

Your User ID is a long number like `284102345871466496`. Developer Mode also lets you copy Channel IDs and Server IDs via right-click.

---

## Step 8: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **Discord** when prompted, then paste your bot token and user ID.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

```bash
# Required
DISCORD_BOT_TOKEN=your-bot-token
DISCORD_ALLOWED_USERS=284102345871466496

# Multiple allowed users (comma-separated)
# DISCORD_ALLOWED_USERS=284102345871466496,198765432109876543
```

Optional settings in `~/.hermes/config.yaml`:

```yaml
discord:
  require_mention: true       # true (default) | false
  free_response_channels:
    - "123456789012345678"    # channel IDs that don't require @mention
  auto_thread: false          # auto-create threads on @mention (v0.3.0)

group_sessions_per_user: true
```

### Start the Gateway

```bash
hermes gateway
```

---

## config.yaml Section

```yaml
discord:
  require_mention: true
  free_response_channels:
    - "123456789012345678"
  auto_thread: false

group_sessions_per_user: true
```

- `require_mention: true` — keep Hermes quiet in server channels unless @mentioned (default)
- `require_mention: false` — respond to all messages in channels (use with care in busy servers)
- `free_response_channels` — list of channel IDs where @mention is not required
- `auto_thread` — when true, automatically creates a thread for each @mention response (v0.3.0)
- `group_sessions_per_user: true` — keep each participant's context isolated (default, recommended)

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `DISCORD_BOT_TOKEN` | Yes | Bot token from Developer Portal |
| `DISCORD_ALLOWED_USERS` | Recommended | Comma-separated numeric User IDs |
| `DISCORD_HOME_CHANNEL` | No | Channel ID for cron job delivery |
| `DISCORD_HOME_CHANNEL_NAME` | No | Display name for the home channel |
| `DISCORD_REQUIRE_MENTION` | No | `true`/`false` — also readable from `config.yaml` |
| `DISCORD_FREE_RESPONSE_CHANNELS` | No | Comma-separated channel IDs — also readable from `config.yaml` |
| `DISCORD_AUTO_THREAD` | No | `true`/`false` — also readable from `config.yaml` (v0.3.0) |
| `DISCORD_ALLOW_ALL_USERS` | No | Allow all users (not recommended) |

---

## Home Channel

Type `/sethome` in any Discord channel where the bot is present, or set manually:

```bash
DISCORD_HOME_CHANNEL=123456789012345678
DISCORD_HOME_CHANNEL_NAME="#bot-updates"
```

---

## Session Model

By default:
- Each DM gets its own session
- Each thread gets its own session namespace
- Each user in a shared channel gets their own session inside that channel

With `group_sessions_per_user: true` (default):
- Alice's request in `#research` only affects Alice's session
- Bob can keep talking in the same channel without inheriting Alice's history
- Alice interrupting her own in-flight request only affects Alice's session

With `group_sessions_per_user: false`:
- The whole channel shares one running-agent slot
- Follow-up messages from different people interrupt or queue behind each other
- Users share context growth and token costs

Channel topic is included in session context so the agent is aware of the channel's purpose.

---

## Voice Support

### Incoming Voice Messages

Voice/audio messages are automatically transcribed using the configured STT provider: local `faster-whisper` (no key), Groq Whisper (`GROQ_API_KEY`), or OpenAI Whisper (`VOICE_TOOLS_OPENAI_KEY`).

### Text-to-Speech

Use `/voice tts` to have the bot send spoken audio responses alongside text replies.

### Discord Voice Channels

Hermes supports joining Discord voice channels, listening to users speaking, and responding in the channel. This requires the full voice mode setup. See the Voice Mode documentation for details.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot is online but not responding | Message Content Intent is disabled. Go to Developer Portal → your app → Bot → Privileged Gateway Intents → enable Message Content Intent → Save Changes → restart gateway. |
| "Disallowed Intents" error on startup | Enable all three Privileged Gateway Intents in Bot settings, then restart. |
| Bot can't see a specific channel | The bot's role doesn't have permission to view that channel. Go to channel settings → Permissions → add the bot's role with View Channel and Read Message History. |
| 403 Forbidden errors | Missing required permissions. Re-invite the bot with the correct permissions URL, or adjust the bot's role permissions in Server Settings. |
| Bot is offline | The gateway isn't running, or the token is incorrect. Check `hermes gateway` is running and verify `DISCORD_BOT_TOKEN`. |
| "User not allowed" / bot ignores you | Your User ID isn't in `DISCORD_ALLOWED_USERS`. Add it and restart the gateway. |
| Shared channel context bleeding between users | Set `group_sessions_per_user: true` in `config.yaml` and restart. |

---

## Security

Always set `DISCORD_ALLOWED_USERS`. Without it, the gateway denies all users by default. Only add User IDs of people you trust — authorized users have full access to the agent's capabilities including terminal access.

---

## v0.6.0 Changes

### Processing reactions ([PR #3871](https://github.com/NousResearch/hermes-agent/pull/3871))

The bot now adds an emoji reaction to your message while it is processing, and removes it once the response is sent. This gives immediate visual feedback in busy channels — you can see at a glance which messages are being worked on and which are complete.

The "Add Reactions" permission must be enabled on the bot's role for this to work (see the recommended permissions in Step 5).

### DISCORD_IGNORE_NO_MENTION ([PR #3640](https://github.com/NousResearch/hermes-agent/pull/3640))

By default, if your message @mentions another user or bot but not Hermes, Hermes still checks whether it should respond (based on `require_mention` and channel rules). Set `DISCORD_IGNORE_NO_MENTION=true` to skip these messages entirely:

```bash
DISCORD_IGNORE_NO_MENTION=true
```

This is useful in active servers where many messages contain @mentions directed at other bots.

### Deferred "thinking..." cleanup fix ([PR #3674](https://github.com/NousResearch/hermes-agent/pull/3674))

Slash command responses that trigger a deferred "thinking..." indicator now properly clean up the indicator after the agent turn completes. Previously, the "thinking..." message could persist in the channel after Hermes had already sent its reply. This is a follow-up to the phantom typing indicator fix from v0.5.0.

---

## Changelog

- **v0.6.0:** Processing reactions on inbound messages ([PR #3871](https://github.com/NousResearch/hermes-agent/pull/3871)).
- **v0.6.0:** `DISCORD_IGNORE_NO_MENTION` env var to skip messages mentioning other users ([PR #3640](https://github.com/NousResearch/hermes-agent/pull/3640)).
- **v0.6.0:** Deferred "thinking..." indicator cleanup fix ([PR #3674](https://github.com/NousResearch/hermes-agent/pull/3674)).
- **v0.4.0:** Document caching and text-file injection ([PR #2503](https://github.com/NousResearch/hermes-agent/pull/2503)). Persistent typing indicator for DMs ([PR #2468](https://github.com/NousResearch/hermes-agent/pull/2468)). DM vision support for inline images and attachments ([PR #2186](https://github.com/NousResearch/hermes-agent/pull/2186)).
- **v0.5.0:** Phantom typing indicator after agent turn completes is now stopped correctly ([PR #3003](https://github.com/NousResearch/hermes-agent/pull/3003)).
