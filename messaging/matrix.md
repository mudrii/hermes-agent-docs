# Matrix

Hermes Agent integrates with Matrix, the open, federated messaging protocol. Matrix lets you run your own homeserver or use a public one like matrix.org — either way, you keep control of your communications. The bot connects via the `matrix-nio` Python SDK, processes messages through the Hermes Agent pipeline (including tool use, memory, and reasoning), and responds in real time. It supports text, file attachments, images, audio, video, and optional end-to-end encryption (E2EE).

Hermes works with any Matrix homeserver — Synapse, Conduit, Dendrite, or matrix.org.

This document covers v0.2.0 through v0.6.0 (v2026.3.30).

---

## How Hermes Behaves

| Context | Behavior |
|---------|----------|
| **DMs** | Hermes responds to every message. No `@mention` needed. Each DM has its own session. |
| **Rooms** | Hermes responds to all messages in rooms it has joined. Room invites are auto-accepted. |
| **Threads** | Hermes supports Matrix threads (MSC3440). If you reply in a thread, Hermes keeps the thread context isolated from the main room timeline. |
| **Shared rooms with multiple users** | By default, Hermes isolates session history per user inside the room. Two people talking in the same room do not share one transcript. |

The bot automatically joins rooms when invited. Just invite the bot's Matrix user to any room and it will join and start responding immediately.

---

## Session Model

By default:
- Each DM gets its own session
- Each thread gets its own session namespace
- Each user in a shared room gets their own session inside that room

This is controlled by `config.yaml`:

```yaml
group_sessions_per_user: true
```

Set to `false` only if you explicitly want one shared conversation for the entire room. Shared sessions mean users share context growth and token costs, and one person's in-flight run can interrupt another person's follow-up.

---

## Prerequisites

- A Matrix account for the bot (see Step 1)
- `matrix-nio` Python package (`pip install 'matrix-nio[e2e]'` or `pip install 'hermes-agent[matrix]'`)
- For E2EE: `libolm` system library

---

## Step 1: Create a Bot Account

### Option A: Register on Your Homeserver (Recommended)

If you run your own homeserver (Synapse, Conduit, Dendrite):

```bash
# Synapse example
register_new_matrix_user -c /etc/synapse/homeserver.yaml http://localhost:8008
```

Choose a username like `hermes` — the full user ID will be `@hermes:your-server.org`.

### Option B: Use matrix.org or Another Public Homeserver

1. Go to [Element Web](https://app.element.io) and create a new account
2. Pick a username for your bot (e.g., `hermes-bot`)

### Option C: Use Your Own Account

You can run Hermes as your own user. This means the bot posts as you — useful for personal assistants.

---

## Step 2: Get an Access Token

### Option A: Via Element (Recommended)

1. Log in to [Element](https://app.element.io) with the bot account
2. Go to **Settings** → **Help & About**
3. Scroll down and expand **Advanced** — the access token is displayed there
4. Copy it immediately

### Option B: Via the API

```bash
curl -X POST https://your-server/_matrix/client/v3/login \
  -H "Content-Type: application/json" \
  -d '{
    "type": "m.login.password",
    "user": "@hermes:your-server.org",
    "password": "your-password"
  }'
```

The response includes an `access_token` field.

### Option C: Password Login

Instead of an access token, provide the bot's user ID and password. Hermes will log in automatically on startup. Simpler, but the password is stored in your `.env` file.

The access token gives full access to the bot's Matrix account. Never share it publicly or commit it to Git.

---

## Step 3: Find Your Matrix User ID

Matrix User IDs follow the format `@username:server` (e.g., `@alice:matrix.org`).

To find yours:
1. Open [Element](https://app.element.io) or your preferred Matrix client
2. Click your avatar → **Settings**
3. Your User ID is displayed at the top of the profile

---

## Step 4: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **Matrix** when prompted, then provide your homeserver URL, access token (or user ID + password), and allowed user IDs.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

**Using an access token:**

```bash
# Required
MATRIX_HOMESERVER=https://matrix.example.org
MATRIX_ACCESS_TOKEN=your-access-token

# Optional: user ID (auto-detected from token if omitted)
# MATRIX_USER_ID=@hermes:matrix.example.org

# Security: restrict who can interact with the bot
MATRIX_ALLOWED_USERS=@alice:matrix.example.org,@bob:matrix.example.org
```

**Using password login:**

```bash
# Required
MATRIX_HOMESERVER=https://matrix.example.org
MATRIX_USER_ID=@hermes:matrix.example.org
MATRIX_PASSWORD=your-password

# Security
MATRIX_ALLOWED_USERS=@alice:matrix.example.org
```

### Start the Gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

The bot connects to your homeserver and starts syncing within a few seconds.

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
| `MATRIX_HOMESERVER` | Yes | Homeserver URL (e.g., `https://matrix.example.org`) |
| `MATRIX_ACCESS_TOKEN` | One of these | Access token for the bot account |
| `MATRIX_PASSWORD` | One of these | Bot account password (used with `MATRIX_USER_ID`) |
| `MATRIX_USER_ID` | See notes | Bot's Matrix user ID; auto-detected from token if omitted |
| `MATRIX_ALLOWED_USERS` | Recommended | Comma-separated Matrix user IDs (full `@user:server` format) |
| `MATRIX_HOME_ROOM` | No | Room ID for cron job delivery |
| `MATRIX_HOME_ROOM_NAME` | No | Display name for the home room (default: `Home`) |
| `MATRIX_ENCRYPTION` | No | Set to `true` to enable E2EE |
| `MATRIX_ALLOW_ALL_USERS` | No | Allow all users (not recommended) |

---

## Home Room

Designate a home room where the bot sends proactive messages (cron job output, reminders, notifications).

**Using the slash command:** Type `/sethome` in any Matrix room where the bot is present.

**Manual configuration:**

```bash
MATRIX_HOME_ROOM=!abc123def456:matrix.example.org
```

To find a Room ID: in Element, go to the room → **Settings** → **Advanced** → the **Internal room ID** is shown there (starts with `!`).

---

## End-to-End Encryption (E2EE)

Hermes supports Matrix end-to-end encryption for chatting with your bot in encrypted rooms.

### Requirements

E2EE requires `matrix-nio` with encryption extras and the `libolm` C library:

```bash
# Install matrix-nio with E2EE support
pip install 'matrix-nio[e2e]'

# Or install with hermes extras
pip install 'hermes-agent[matrix]'
```

Install `libolm` on your system:

```bash
# Debian/Ubuntu
sudo apt install libolm-dev

# macOS
brew install libolm

# Fedora
sudo dnf install libolm-devel
```

### Enable E2EE

Add to `~/.hermes/.env`:

```bash
MATRIX_ENCRYPTION=true
```

When E2EE is enabled, Hermes:
- Stores encryption keys in `~/.hermes/matrix/store/`
- Uploads device keys on first connection
- Decrypts incoming messages and encrypts outgoing messages automatically
- Auto-joins encrypted rooms when invited

If `matrix-nio[e2e]` is not installed or `libolm` is missing, the bot falls back to a plain (unencrypted) client automatically with a warning in the logs.

Do not delete the `~/.hermes/matrix/store/` directory — the bot loses its encryption keys and you will need to verify the device again in your Matrix client.

### Access-Token Hardening (v0.5.0)

v0.5.0 hardened E2EE access-token handling ([PR #3562](https://github.com/NousResearch/hermes-agent/pull/3562)). The access token is now validated and rotated defensively on startup to prevent sessions from using stale tokens that cause silent decryption failures. If you run E2EE and see "could not decrypt event" errors after upgrading, re-authenticate by setting a fresh `MATRIX_ACCESS_TOKEN` and restarting the gateway.

**v0.5.0** also added exponential backoff for `SyncError` in the sync loop ([PR #3280](https://github.com/NousResearch/hermes-agent/pull/3280)), preventing tight reconnect loops during homeserver maintenance windows.

---

## Message Length and Media Support

Matrix messages are limited to 4,000 characters (the spec has no hard limit, but clients render poorly above this). Longer responses are split at natural boundaries.

Hermes can send and receive images, audio, video, and file attachments. Media is uploaded to your homeserver using the Matrix content repository API.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot not responding to messages | Invite the bot to the room (it auto-joins). Verify your User ID is in `MATRIX_ALLOWED_USERS` using the full `@user:server` format. Restart the gateway. |
| "Failed to authenticate" / "whoami failed" on startup | Verify `MATRIX_HOMESERVER` includes `https://` with no trailing slash. Validate your token: `curl -H "Authorization: Bearer YOUR_TOKEN" https://your-server/_matrix/client/v3/account/whoami` |
| "matrix-nio not installed" error | Run `pip install 'matrix-nio[e2e]'` or `pip install 'hermes-agent[matrix]'` |
| Encryption errors / "could not decrypt event" | Verify `libolm` is installed. Set `MATRIX_ENCRYPTION=true`. In Element, go to the bot's profile → Sessions → verify/trust the bot's device. |
| Sync issues / bot falls behind | The sync loop automatically retries every 5 seconds on error. Check Hermes logs for sync-related warnings. |
| Bot is offline | Check `hermes gateway` is running. Look for error messages in terminal output. Common causes: wrong homeserver URL, expired access token, homeserver unreachable. |
| "User not allowed" / bot ignores you | Add your User ID to `MATRIX_ALLOWED_USERS` in `~/.hermes/.env` and restart. Use the full `@user:server` format. |

---

## Security

Always set `MATRIX_ALLOWED_USERS` to restrict who can interact with the bot. Without it, the gateway denies all users by default. Only add User IDs of people you trust — authorized users have full access to the agent's capabilities, including tool use and system access.

- Works with federation: users from other servers can be added to `MATRIX_ALLOWED_USERS` using their full `@user:server` IDs
- Protect `~/.hermes/matrix/store/` — it contains encryption keys
- Access tokens and passwords are stored in `~/.hermes/.env` — protect this file (`chmod 600`)

---

## v0.6.0 Changes

### Native voice messages via MSC3245 ([PR #3877](https://github.com/NousResearch/hermes-agent/pull/3877))

Hermes now sends TTS audio responses as native Matrix voice messages conforming to [MSC3245](https://github.com/matrix-org/matrix-spec-proposals/pull/3245). In Matrix clients that support MSC3245 (Element Web, Element X, FluffyChat), these messages display as playable voice bubbles with a waveform visualization — rather than generic file attachments.

Clients that do not support MSC3245 fall back gracefully to the standard `m.audio` event type and display the audio as a downloadable file attachment.

No configuration change is required — the adapter automatically uses the MSC3245 format when sending voice messages.

---

## Changelog

- **v0.6.0:** Native MSC3245 voice messages with waveform player support ([PR #3877](https://github.com/NousResearch/hermes-agent/pull/3877)).
- **v0.5.0:** Access-token hardening for E2EE — token validated and rotated defensively on startup ([PR #3562](https://github.com/NousResearch/hermes-agent/pull/3562)).
- **v0.5.0:** Exponential backoff for `SyncError` in sync loop ([PR #3280](https://github.com/NousResearch/hermes-agent/pull/3280)).
