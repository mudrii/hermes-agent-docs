# Email

Hermes can receive and reply to emails using standard IMAP and SMTP protocols. Send an email to the agent's address and it replies in-thread — no special client or bot API needed. Works with Gmail, Outlook, Yahoo, Fastmail, or any provider that supports IMAP/SMTP.

The Email adapter uses Python's built-in `imaplib`, `smtplib`, and `email` modules. No additional Python packages are required.

This document covers the released Email adapter surface through v0.8.0 (`v2026.4.8`). No Email-specific changes were made in v0.8.0.

---

## Prerequisites

- **A dedicated email account** for your Hermes agent (do not use your personal email)
- **IMAP enabled** on the email account
- **An app password** if using Gmail or another provider with 2FA

### Gmail Setup

1. Enable 2-Factor Authentication on your Google Account
2. Go to [App Passwords](https://myaccount.google.com/apppasswords)
3. Create a new App Password (select "Mail" or "Other")
4. Copy the 16-character password — you will use this instead of your regular password

### Outlook / Microsoft 365

1. Go to [Security Settings](https://account.microsoft.com/security)
2. Enable 2FA if not already active
3. Create an App Password under "Additional security options"
4. IMAP host: `outlook.office365.com`, SMTP host: `smtp.office365.com`

### Other Providers

Most email providers support IMAP/SMTP. Check your provider's documentation for:
- IMAP host and port (usually port 993 with SSL)
- SMTP host and port (usually port 587 with STARTTLS)
- Whether app passwords are required

---

## Step 1: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **Email** from the platform menu. The wizard prompts for your email address, password, IMAP/SMTP hosts, and allowed senders.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

```bash
# Required
EMAIL_ADDRESS=hermes@gmail.com
EMAIL_PASSWORD=abcd efgh ijkl mnop    # App password (not your regular password)
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com

# Security (recommended)
EMAIL_ALLOWED_USERS=your@email.com,colleague@work.com

# Optional
EMAIL_IMAP_PORT=993                    # Default: 993 (IMAP SSL)
EMAIL_SMTP_PORT=587                    # Default: 587 (SMTP STARTTLS)
EMAIL_POLL_INTERVAL=15                 # Seconds between inbox checks (default: 15)
EMAIL_HOME_ADDRESS=your@email.com      # Default delivery target for cron jobs
```

---

## Step 2: Start the Gateway

```bash
hermes gateway              # Run in foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: boot-time system service
```

On startup, the adapter:
1. Tests IMAP and SMTP connections
2. Marks all existing inbox messages as "seen" (only processes new emails)
3. Starts polling for new messages

---

## How It Works

### Receiving Messages

The adapter polls the IMAP inbox for UNSEEN messages at a configurable interval (default: 15 seconds). For each new email:

- **Subject line** is included as context (e.g., `[Subject: Deploy to production]`)
- **Reply emails** (subject starting with `Re:`) skip the subject prefix — the thread context is already established
- **Attachments** are cached locally:
  - Images (JPEG, PNG, GIF, WebP) are available to the vision tool
  - Documents (PDF, ZIP, etc.) are available for file access
- **HTML-only emails** have tags stripped for plain text extraction
- **Automated sender filtering** — emails from addresses matching common no-reply patterns (`noreply@`, `no-reply@`, `do-not-reply@`, `mailer-daemon@`, `postmaster@`) are automatically skipped to prevent the agent from responding to automated notifications, newsletters, and bounce messages
- **Self-messages** are filtered out to prevent reply loops

### Sending Replies

Replies are sent via SMTP with proper email threading:

- **In-Reply-To** and **References** headers maintain the thread in email clients
- **Subject line** preserved with `Re:` prefix (no double `Re: Re:`)
- **Message-ID** generated with the agent's domain
- Responses are sent as plain text (UTF-8)

### Message Length Limit

Email responses are limited to 50,000 characters per email body. This is a Gmail-safe maximum; other providers may have different limits.

### File Attachments (Outgoing)

The agent can send file attachments in replies. Include `MEDIA:/path/to/file` in the response and the file is attached to the outgoing email.

### Skipping Incoming Attachments

Added in v0.3.0. To ignore all incoming attachments (for malware protection or bandwidth savings), add to `~/.hermes/config.yaml`:

```yaml
platforms:
  email:
    skip_attachments: true
```

When enabled, attachment and inline parts are skipped before payload decoding. The email body text is still processed normally.

---

## config.yaml Section

```yaml
platforms:
  email:
    skip_attachments: false    # true to ignore all incoming attachments (v0.3.0)
```

---

## Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `EMAIL_ADDRESS` | Yes | — | Agent's email address |
| `EMAIL_PASSWORD` | Yes | — | Email password or app password |
| `EMAIL_IMAP_HOST` | Yes | — | IMAP server host (e.g., `imap.gmail.com`) |
| `EMAIL_SMTP_HOST` | Yes | — | SMTP server host (e.g., `smtp.gmail.com`) |
| `EMAIL_IMAP_PORT` | No | `993` | IMAP server port |
| `EMAIL_SMTP_PORT` | No | `587` | SMTP server port |
| `EMAIL_POLL_INTERVAL` | No | `15` | Seconds between inbox checks |
| `EMAIL_ALLOWED_USERS` | No | — | Comma-separated allowed sender addresses |
| `EMAIL_HOME_ADDRESS` | No | — | Default delivery target for cron jobs |
| `EMAIL_HOME_ADDRESS_NAME` | No | `Home` | Display name for the home address |
| `EMAIL_ALLOW_ALL_USERS` | No | `false` | Allow all senders (not recommended) |

---

## Access Control

Email access follows the same pattern as all other Hermes platforms:

1. **`EMAIL_ALLOWED_USERS` set** — only emails from those addresses are processed
2. **No allowlist set** — unknown senders get a pairing code
3. **`EMAIL_ALLOW_ALL_USERS=true`** — any sender is accepted (use with caution)

Always configure `EMAIL_ALLOWED_USERS`. Without it, anyone who knows the agent's email address could send commands. The agent has terminal access by default.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "IMAP connection failed" at startup | Verify `EMAIL_IMAP_HOST` and `EMAIL_IMAP_PORT`. Ensure IMAP is enabled on the account. For Gmail, enable it in Settings → Forwarding and POP/IMAP. |
| "SMTP connection failed" at startup | Verify `EMAIL_SMTP_HOST` and `EMAIL_SMTP_PORT`. Check that your password is correct (use App Password for Gmail). |
| Messages not received | Check `EMAIL_ALLOWED_USERS` includes the sender's email. Check spam folder — some providers flag automated replies. |
| "Authentication failed" | For Gmail, you must use an App Password, not your regular password. Ensure 2FA is enabled first. |
| Duplicate replies | Ensure only one gateway instance is running. Check `hermes gateway status`. |
| Slow response | The default poll interval is 15 seconds. Reduce with `EMAIL_POLL_INTERVAL=5` for faster responses (but more IMAP connections). |
| Replies not threading | The adapter uses In-Reply-To headers. Some email clients (especially web-based) may not thread correctly with automated messages. |

---

## Security

Use a dedicated email account. Do not use your personal email — the agent stores the password in `.env` and has full inbox access via IMAP.

- Use **App Passwords** instead of your main password (required for Gmail with 2FA)
- Set `EMAIL_ALLOWED_USERS` to restrict who can interact with the agent
- The password is stored in `~/.hermes/.env` — protect this file (`chmod 600`)
- IMAP uses SSL (port 993) and SMTP uses STARTTLS (port 587) by default — connections are encrypted

---

## v0.6.0 Changes

### Close SMTP/IMAP connections on failure ([PR #3804](https://github.com/NousResearch/hermes-agent/pull/3804))

The adapter now explicitly closes SMTP and IMAP connections when an error occurs during a send or receive operation. Previously, failed connections could remain open indefinitely, leaking socket file descriptors. Over time, especially on long-running gateway instances, this could exhaust the connection pool and prevent new emails from being processed.

No configuration change is required — the fix is applied automatically.

---

## Changelog

- **v0.6.0:** SMTP/IMAP connections closed on error to prevent resource leaks ([PR #3804](https://github.com/NousResearch/hermes-agent/pull/3804)).
- **v0.5.0:** Unbounded growth of the `_seen_uids` set (which tracks processed messages) is now capped, preventing memory growth on high-volume inboxes ([PR #3490](https://github.com/NousResearch/hermes-agent/pull/3490)).
- **v0.5.0:** Request timeouts added ([PR #3258](https://github.com/NousResearch/hermes-agent/pull/3258)).
