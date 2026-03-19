# WhatsApp

Hermes connects to WhatsApp through a built-in bridge based on **Baileys**. This works by emulating a WhatsApp Web session — not through the official WhatsApp Business API. No Meta developer account or Business verification is required.

This document covers v0.2.0 (v2026.3.12) and v0.3.0 (v2026.3.17).

---

## Important Warnings

WhatsApp does **not** officially support third-party bots outside the Business API. Using a third-party bridge carries a small risk of account restrictions:
- **Use a dedicated phone number** for the bot (not your personal number)
- **Don't send bulk or spam messages** — keep usage conversational
- **Don't automate outbound messaging** to people who haven't messaged first

WhatsApp periodically updates their Web protocol, which can temporarily break compatibility with third-party bridges. When this happens, Hermes will update the bridge dependency. If the bot stops working after a WhatsApp update, pull the latest Hermes version and re-pair.

---

## Prerequisites

- **Node.js v18+** and **npm** — the WhatsApp bridge runs as a Node.js process
- **A phone with WhatsApp** installed (for scanning the QR code)

Unlike older browser-driven bridges, the current Baileys-based bridge does not require a local Chromium or Puppeteer dependency stack.

---

## Two Modes

| Mode | How it works | Best for |
|------|-------------|----------|
| **Separate bot number** (recommended) | Dedicate a phone number to the bot. People message that number directly. | Clean UX, multiple users, lower ban risk |
| **Personal self-chat** | Use your own WhatsApp. You message yourself to talk to the agent. | Quick setup, single user, testing |

---

## Step 1: Run the Setup Wizard

```bash
hermes whatsapp
```

The wizard will:
1. Ask which mode you want (`bot` or `self-chat`)
2. Install bridge dependencies if needed
3. Display a QR code in your terminal
4. Wait for you to scan it

To scan the QR code:
1. Open WhatsApp on your phone
2. Go to **Settings → Linked Devices**
3. Tap **Link a Device**
4. Point your camera at the terminal QR code

Once paired, the wizard confirms the connection and exits. Your session is saved automatically.

If the QR code looks garbled, make sure your terminal is at least 60 columns wide and supports Unicode.

---

## Step 2: Getting a Second Phone Number (Bot Mode)

For bot mode, you need a phone number not already registered with WhatsApp:

| Option | Cost | Notes |
|--------|------|-------|
| **Google Voice** | Free | US only. Verify WhatsApp via SMS through the Google Voice app. |
| **Prepaid SIM** | $5–15 one-time | Any carrier. Number must stay active (make a call every 90 days). |
| **VoIP services** | Free–$5/month | TextNow, TextFree, or similar. Some VoIP numbers are blocked by WhatsApp. |

---

## Step 3: Configure Hermes

Add to `~/.hermes/.env`:

```bash
# Required
WHATSAPP_ENABLED=true
WHATSAPP_MODE=self-chat                    # "bot" or "self-chat" (default: self-chat)
WHATSAPP_ALLOWED_USERS=15551234567         # Comma-separated phone numbers (with country code, no +)
```

Then start the gateway:

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

The gateway starts the WhatsApp bridge automatically using the saved session.

---

## config.yaml Section

```yaml
# Global unauthorized DM behavior
unauthorized_dm_behavior: pair

# WhatsApp-specific override
whatsapp:
  unauthorized_dm_behavior: ignore

  # Optional: customize the reply header
  reply_prefix: "⚕ **Hermes Agent**\n──────\n"   # default
  # reply_prefix: ""                               # disable the header
  # reply_prefix: "🤖 *My Bot*\n──────\n"          # custom prefix (supports \n)
```

- `unauthorized_dm_behavior: ignore` makes WhatsApp stay silent for unauthorized DMs, which is usually the better choice for a private number
- `reply_prefix` controls the header prepended to agent responses (supports `\n` for newlines)

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `WHATSAPP_ENABLED` | Yes | Set to `true` to enable WhatsApp |
| `WHATSAPP_MODE` | No | `bot` or `self-chat` (default: `self-chat`) |
| `WHATSAPP_ALLOWED_USERS` | Recommended | Comma-separated phone numbers with country code, without `+` |
| `WHATSAPP_ALLOW_ALL_USERS` | No | Allow all senders (not recommended) |

---

## Session Persistence

The Baileys bridge saves its session under `~/.hermes/whatsapp/session`. Sessions survive gateway restarts — you don't need to re-scan the QR code each time. The session data includes encryption keys and device credentials.

Do not share or commit the `~/.hermes/whatsapp/session` directory. It grants full access to the WhatsApp account.

---

## Re-Pairing

If the session breaks (phone reset, WhatsApp update, manually unlinked), you'll see connection errors in the gateway logs. To fix:

```bash
hermes whatsapp
```

This generates a fresh QR code. Scan it again and the session is re-established. The gateway handles temporary disconnections (network blips, phone going offline briefly) automatically with reconnection logic.

---

## Voice Support

- **Incoming** — Voice messages (`.ogg` Opus format) are automatically transcribed using the configured STT provider: local `faster-whisper`, Groq Whisper (`GROQ_API_KEY`), or OpenAI Whisper (`VOICE_TOOLS_OPENAI_KEY`)
- **Outgoing** — TTS responses are sent as MP3 audio file attachments

---

## Message Length Limit

WhatsApp allows messages up to 65,536 characters. Longer responses are split at natural boundaries.

---

## Native Media Support

v0.2.0 added native media sending for images, videos, and documents. The adapter can send:
- Images as native WhatsApp photos
- Videos as native video messages
- Documents as file attachments

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| QR code not scanning | Ensure terminal is 60+ columns wide. Try a different terminal emulator. Make sure you're scanning from the correct WhatsApp account (bot number, not personal). |
| QR code expires | QR codes refresh every ~20 seconds. If it times out, restart `hermes whatsapp`. |
| Session not persisting | Check that `~/.hermes/whatsapp/session` exists and is writable. |
| Logged out unexpectedly | WhatsApp unlinks devices after long inactivity. Keep the phone on and connected, then re-pair. |
| Bridge crashes or reconnect loops | Restart the gateway, update Hermes, and re-pair if the session was invalidated by a WhatsApp protocol change. |
| Bot stops working after WhatsApp update | Update Hermes to get the latest bridge version, then re-pair. |
| Messages not being received | Verify `WHATSAPP_ALLOWED_USERS` includes the sender's number with country code, without `+`. |
| Bot replies to strangers with a pairing code | Set `whatsapp.unauthorized_dm_behavior: ignore` in `config.yaml`. |

---

## Security

Always set `WHATSAPP_ALLOWED_USERS` with phone numbers (country code included, without `+`). Without this setting, the gateway will deny all incoming messages by default.

- Protect the `~/.hermes/whatsapp/session` directory: `chmod 700 ~/.hermes/whatsapp/session`
- Use a **dedicated phone number** to isolate risk from your personal account
- If you suspect compromise, unlink the device from WhatsApp → Settings → Linked Devices
- Phone numbers in logs are partially redacted, but review your log retention policy
