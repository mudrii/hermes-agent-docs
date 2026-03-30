# SMS (Twilio)

Hermes connects to SMS through the [Twilio](https://www.twilio.com/) API. People text your Twilio phone number and get AI responses back — same conversational experience as Telegram or Discord, but over standard text messages.

The SMS gateway shares credentials with the optional telephony skill. If you have already set up Twilio for voice calls or one-off SMS, the gateway works with the same `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, and `TWILIO_PHONE_NUMBER`.

This document covers v0.2.0 through v0.5.0 (v2026.3.28).

---

## Prerequisites

- **Twilio account** — [Sign up at twilio.com](https://www.twilio.com/try-twilio) (free trial available)
- **A Twilio phone number** with SMS capability
- **A publicly accessible server** — Twilio sends webhooks to your server when SMS arrives
- **aiohttp** — `pip install 'hermes-agent[sms]'`

Unlike most other Hermes platforms, SMS requires a publicly accessible URL. The webhook port defaults to `8080`. You can use a tunnel service (cloudflared, ngrok) when running locally.

---

## Step 1: Get Your Twilio Credentials

1. Go to the [Twilio Console](https://console.twilio.com/)
2. Copy your **Account SID** and **Auth Token** from the dashboard
3. Go to **Phone Numbers → Manage → Active Numbers** — note your phone number in E.164 format (e.g., `+15551234567`)

---

## Step 2: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **SMS (Twilio)** from the platform list. The wizard will prompt for your credentials.

### Option B: Manual Setup

Add to `~/.hermes/.env`:

```bash
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token_here
TWILIO_PHONE_NUMBER=+15551234567

# Security: restrict to specific phone numbers (recommended)
SMS_ALLOWED_USERS=+15559876543,+15551112222

# Optional
SMS_HOME_CHANNEL=+15559876543
SMS_WEBHOOK_PORT=8080    # Default: 8080
```

---

## Step 3: Configure Twilio Webhook

Twilio needs to know where to send incoming messages. In the [Twilio Console](https://console.twilio.com/):

1. Go to **Phone Numbers → Manage → Active Numbers**
2. Click your phone number
3. Under **Messaging → A MESSAGE COMES IN**, set:
   - **Webhook**: `https://your-server:8080/webhooks/twilio`
   - **HTTP Method**: `POST`

If you are running Hermes locally, use a tunnel to expose the webhook:

```bash
# Using cloudflared
cloudflared tunnel --url http://localhost:8080

# Using ngrok
ngrok http 8080
```

Set the resulting public URL as your Twilio webhook. Update the URL in Twilio Console whenever the tunnel URL changes.

---

## Step 4: Start the Gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

You should see:

```
[sms] Twilio webhook server listening on port 8080, from: +1555***4567
```

Text your Twilio number — Hermes will respond via SMS.

---

## Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `TWILIO_ACCOUNT_SID` | Yes | — | Twilio Account SID (starts with `AC`) |
| `TWILIO_AUTH_TOKEN` | Yes | — | Twilio Auth Token |
| `TWILIO_PHONE_NUMBER` | Yes | — | Your Twilio phone number (E.164 format) |
| `SMS_WEBHOOK_PORT` | No | `8080` | Webhook listener port |
| `SMS_ALLOWED_USERS` | No | — | Comma-separated E.164 phone numbers allowed to chat |
| `SMS_ALLOW_ALL_USERS` | No | `false` | Allow anyone to chat (not recommended) |
| `SMS_HOME_CHANNEL` | No | — | Phone number for cron job / notification delivery |
| `SMS_HOME_CHANNEL_NAME` | No | `Home` | Display name for the home channel |

---

## SMS-Specific Behavior

### Plain Text Only

Markdown is automatically stripped since SMS renders it as literal characters. Responses are always delivered as plain text.

### 1600 Character Limit

Longer responses are split across multiple messages at natural boundaries (newlines, then spaces).

### Echo Prevention

Messages from your own Twilio number are ignored to prevent loops.

### Phone Number Redaction

Phone numbers are redacted in logs for privacy (e.g., `+15551234567` → `+155****4567`).

---

## Access Control

The gateway denies all users by default. Configure an allowlist:

```bash
# Recommended: restrict to specific phone numbers
SMS_ALLOWED_USERS=+15559876543,+15551112222

# Or allow all (NOT recommended for agents with terminal access)
SMS_ALLOW_ALL_USERS=true
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Messages not arriving | Check your Twilio webhook URL is correct and publicly accessible. Verify `TWILIO_ACCOUNT_SID` and `TWILIO_AUTH_TOKEN` are correct. Check Twilio Console → **Monitor → Logs → Messaging** for delivery errors. Ensure your phone number is in `SMS_ALLOWED_USERS`. |
| Replies not sending | Check `TWILIO_PHONE_NUMBER` is set correctly (E.164 format with `+`). Verify your Twilio account has SMS-capable numbers. Check Hermes gateway logs for Twilio API errors. |
| Webhook port conflicts | If port 8080 is already in use, set `SMS_WEBHOOK_PORT=3001` and update the webhook URL in Twilio Console to match. |
| Tunnel URL changes | If using a tunnel for local development, update the Twilio webhook URL in the Console whenever the public URL changes. Use a stable tunnel URL if possible. |

---

## Security

SMS has no built-in encryption. Do not use SMS for sensitive operations unless you understand the security implications. For sensitive use cases, prefer Signal or Telegram.

- Always set `SMS_ALLOWED_USERS` with the phone numbers of authorized users
- Store Twilio credentials in `~/.hermes/.env` with file permissions `600`
- Phone numbers are partially redacted in logs, but review your log retention policy
- The Twilio Auth Token grants full access to your Twilio account — protect it accordingly

---

## Changelog

- **v0.4.0:** SMS (Twilio) adapter added ([PR #1688](https://github.com/NousResearch/hermes-agent/pull/1688)).
- **v0.5.0:** Request timeouts added ([PR #3258](https://github.com/NousResearch/hermes-agent/pull/3258)).
