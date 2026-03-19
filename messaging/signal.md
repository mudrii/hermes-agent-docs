# Signal

Hermes connects to Signal through the [signal-cli](https://github.com/AsamK/signal-cli) daemon running in HTTP mode. The adapter streams messages in real-time via SSE (Server-Sent Events) and sends responses via JSON-RPC.

Signal is the most privacy-focused mainstream messenger — end-to-end encrypted by default, open-source protocol, minimal metadata collection. This makes it ideal for security-sensitive agent workflows.

The Signal adapter uses `httpx` (already a core Hermes dependency) for all communication. No additional Python packages are required beyond `signal-cli` itself.

This document covers v0.2.0 (v2026.3.12) and v0.3.0 (v2026.3.17).

---

## Prerequisites

- **signal-cli** — Java-based Signal client ([GitHub](https://github.com/AsamK/signal-cli))
- **Java 17+** runtime — required by signal-cli
- **A phone number** with Signal installed (for linking as a secondary device)

---

## Installing signal-cli

```bash
# Linux (Debian/Ubuntu)
sudo apt install signal-cli

# macOS
brew install signal-cli

# Manual install (any platform)
# Download from https://github.com/AsamK/signal-cli/releases
# Extract and add to PATH
```

### Alternative: Docker (signal-cli-rest-api)

```bash
docker run -d --name signal-cli \
  -p 8080:8080 \
  -v $HOME/.local/share/signal-cli:/home/.local/share/signal-cli \
  -e MODE=json-rpc \
  bbernhard/signal-cli-rest-api
```

Use `MODE=json-rpc` for best performance. The `normal` mode spawns a JVM per request and is much slower.

---

## Step 1: Link Your Signal Account

Signal-cli works as a **linked device** — like WhatsApp Web, but for Signal. Your phone stays the primary device.

```bash
# Generate a linking URI (displays a QR code or link)
signal-cli link -n "HermesAgent"
```

1. Open **Signal** on your phone
2. Go to **Settings → Linked Devices**
3. Tap **Link New Device**
4. Scan the QR code or enter the URI

---

## Step 2: Start the signal-cli Daemon

```bash
# Replace +1234567890 with your Signal phone number (E.164 format)
signal-cli --account +1234567890 daemon --http 127.0.0.1:8080
```

Keep this running in the background using `systemd`, `tmux`, or `screen`.

Verify it's running:

```bash
curl http://127.0.0.1:8080/api/v1/check
# Should return: {"versions":{"signal-cli":...}}
```

---

## Step 3: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **Signal** from the platform menu. The wizard checks if signal-cli is installed, prompts for the HTTP URL, tests connectivity, and asks for your account phone number.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

```bash
# Required
SIGNAL_HTTP_URL=http://127.0.0.1:8080
SIGNAL_ACCOUNT=+1234567890

# Security (recommended)
SIGNAL_ALLOWED_USERS=+1234567890,+0987654321    # E.164 format or UUIDs

# Optional
SIGNAL_GROUP_ALLOWED_USERS=groupId1,groupId2     # Specific groups, or * for all
SIGNAL_HOME_CHANNEL=+1234567890
```

Start the gateway:

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

Signal-specific settings (`SIGNAL_GROUP_ALLOWED_USERS`, etc.) are environment variables only.

---

## Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SIGNAL_HTTP_URL` | Yes | — | signal-cli HTTP endpoint |
| `SIGNAL_ACCOUNT` | Yes | — | Bot phone number (E.164 format, with `+`) |
| `SIGNAL_ALLOWED_USERS` | No | — | Comma-separated phone numbers/UUIDs |
| `SIGNAL_GROUP_ALLOWED_USERS` | No | — | Group IDs to monitor, or `*` for all (omit to disable groups) |
| `SIGNAL_HOME_CHANNEL` | No | — | Default delivery target for cron jobs |
| `SIGNAL_HOME_CHANNEL_NAME` | No | `Home` | Display name for the home channel |
| `SIGNAL_IGNORE_STORIES` | No | `true` | Whether to ignore Signal stories |
| `SIGNAL_ALLOW_ALL_USERS` | No | `false` | Allow all DM senders (not recommended) |

---

## Access Control

### DM Access

1. **`SIGNAL_ALLOWED_USERS` set** — only those users can message
2. **No allowlist set** — unknown users get a DM pairing code (approve via `hermes pairing approve signal CODE`)
3. **`SIGNAL_ALLOW_ALL_USERS=true`** — anyone can message (use with caution)

### Group Access

Group access is controlled by `SIGNAL_GROUP_ALLOWED_USERS`:

| Configuration | Behavior |
|---------------|----------|
| Not set (default) | All group messages are ignored. The bot only responds to DMs. |
| Set with group IDs | Only listed groups are monitored. |
| Set to `*` | The bot responds in any group it's a member of. |

---

## Platform-Specific Features

### Attachments

The adapter supports sending and receiving:
- **Images** — PNG, JPEG, GIF, WebP (auto-detected via magic bytes)
- **Audio** — MP3, OGG, WAV, M4A (voice messages transcribed if Whisper is configured)
- **Documents** — PDF, ZIP, and other file types

Attachment size limit: 100 MB.

### Message Length Limit

Signal messages are limited to 8,000 characters. Longer responses are split at natural boundaries.

### Typing Indicators

The bot sends typing indicators while processing messages, refreshing every 8 seconds.

### Phone Number Redaction

All phone numbers are automatically redacted in logs:
- `+15551234567` → `+155****4567`

This applies to both Hermes gateway logs and the global redaction system.

### Health Monitoring

The adapter monitors the SSE connection and automatically reconnects if:
- The connection drops (with exponential backoff: 2s → 60s)
- No activity is detected for 120 seconds (pings signal-cli to verify)

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Cannot reach signal-cli" during setup | Ensure signal-cli daemon is running: `signal-cli --account +YOUR_NUMBER daemon --http 127.0.0.1:8080` |
| Messages not received | Check that `SIGNAL_ALLOWED_USERS` includes the sender's number in E.164 format (with `+` prefix) |
| "signal-cli not found on PATH" | Install signal-cli and ensure it's in your PATH, or use the Docker container |
| Connection keeps dropping | Check signal-cli logs for errors. Ensure Java 17+ is installed. |
| Group messages ignored | Configure `SIGNAL_GROUP_ALLOWED_USERS` with specific group IDs, or `*` to allow all groups. |
| Bot responds to no one | Configure `SIGNAL_ALLOWED_USERS`, or use DM pairing. |
| Duplicate messages | Ensure only one signal-cli instance is listening on your phone number. |

---

## Security

Always configure access controls. The bot has terminal access by default. Without `SIGNAL_ALLOWED_USERS` or DM pairing, the gateway denies all incoming messages.

- Phone numbers are redacted in all log output
- Use DM pairing or explicit allowlists for safe onboarding of new users
- Keep groups disabled unless you specifically need group support
- Signal's end-to-end encryption protects message content in transit
- The signal-cli session data in `~/.local/share/signal-cli/` contains account credentials — protect it like a password
