# DingTalk

Hermes Agent integrates with DingTalk (钉钉) as a chatbot, letting you chat with your AI assistant through direct messages or group chats. The bot connects via DingTalk's Stream Mode — a long-lived WebSocket connection that requires no public URL or webhook server — and replies using markdown-formatted messages through DingTalk's session webhook API.

This document covers v0.2.0 through v0.8.0 (v2026.4.8). No DingTalk-specific changes were made after v0.5.0.

---

## How Hermes Behaves

| Context | Behavior |
|---------|----------|
| **DMs (1:1 chat)** | Hermes responds to every message. No `@mention` needed. Each DM has its own session. |
| **Group chats** | Hermes responds when you `@mention` it. Without a mention, Hermes ignores the message. |
| **Shared groups with multiple users** | By default, Hermes isolates session history per user inside the group. Two people talking in the same group do not share one transcript. |

---

## Session Model

By default:
- Each DM gets its own session
- Each user in a shared group chat gets their own session inside that group

Controlled by `config.yaml`:

```yaml
group_sessions_per_user: true
```

Set to `false` only if you explicitly want one shared conversation for the entire group.

---

## Prerequisites

Install the required Python packages:

```bash
pip install dingtalk-stream httpx
```

- `dingtalk-stream` — DingTalk's official SDK for Stream Mode (WebSocket-based real-time messaging)
- `httpx` — async HTTP client used for sending replies via session webhooks

---

## Step 1: Create a DingTalk App

1. Go to the [DingTalk Developer Console](https://open-dev.dingtalk.com/)
2. Log in with your DingTalk admin account
3. Click **Application Development** → **Custom Apps** → **Create App via H5 Micro-App** (or **Robot** depending on your console version)
4. Fill in the **App Name** (e.g., `Hermes Agent`) and optional description
5. After creating, navigate to **Credentials & Basic Info** to find your **Client ID** (AppKey) and **Client Secret** (AppSecret) — copy both

The Client Secret is only displayed once when you create the app. If you lose it, you will need to regenerate it. Never share these credentials publicly or commit them to Git.

---

## Step 2: Enable the Robot Capability

1. In your app's settings page, go to **Add Capability** → **Robot**
2. Enable the robot capability
3. Under **Message Reception Mode**, select **Stream Mode**

Stream Mode is recommended. It uses a long-lived WebSocket connection initiated from your machine, so you do not need a public IP, domain name, or webhook endpoint. This works behind NAT, firewalls, and on local machines.

---

## Step 3: Find Your DingTalk User ID

Hermes uses your DingTalk User ID to control access. DingTalk User IDs are alphanumeric strings set by your organization's admin.

To find yours:
- Ask your DingTalk organization admin — User IDs are configured in the DingTalk admin console under **Contacts** → **Members**
- Alternatively, start the gateway, send the bot a message, then check the logs for the `sender_id` value

---

## Step 4: Configure Hermes

### Option A: Interactive Setup (Recommended)

```bash
hermes gateway setup
```

Select **DingTalk** when prompted, then paste your Client ID, Client Secret, and allowed user IDs.

### Option B: Manual Configuration

Add to `~/.hermes/.env`:

```bash
# Required
DINGTALK_CLIENT_ID=your-app-key
DINGTALK_CLIENT_SECRET=your-app-secret

# Security: restrict who can interact with the bot
DINGTALK_ALLOWED_USERS=user-id-1

# Multiple allowed users (comma-separated)
# DINGTALK_ALLOWED_USERS=user-id-1,user-id-2
```

### Start the Gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

---

## config.yaml Section

Unlike platforms such as Telegram or Discord, DingTalk does not auto-enable from environment variables alone. You must enable it in `~/.hermes/gateway.json` or `~/.hermes/config.yaml`:

```json
{
  "platforms": {
    "dingtalk": {
      "enabled": true,
      "extra": {
        "client_id": "your-app-key",
        "client_secret": "your-secret"
      }
    }
  }
}
```

Or use `hermes gateway setup`, which writes the configuration automatically.

Global settings in `config.yaml`:

```yaml
group_sessions_per_user: true

unauthorized_dm_behavior: pair    # pair | ignore
```

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `DINGTALK_CLIENT_ID` | Yes | Client ID (AppKey) from the DingTalk Developer Console |
| `DINGTALK_CLIENT_SECRET` | Yes | Client Secret (AppSecret) from the DingTalk Developer Console |
| `DINGTALK_ALLOWED_USERS` | Recommended | Comma-separated DingTalk User IDs |
| `DINGTALK_ALLOW_ALL_USERS` | No | Allow all users (not recommended) |

---

## Platform-Specific Features

### Stream Mode

No public URL, domain name, or webhook server is needed. The connection is initiated from your machine via WebSocket, so it works behind NAT and firewalls.

### Markdown Responses

Replies are formatted in DingTalk's markdown format for rich text display.

### Message Deduplication

The adapter deduplicates messages with a 5-minute window to prevent processing the same message twice. This handles cases where the DingTalk platform retries delivery.

### Message Length Limit

Responses are capped at 20,000 characters per message. Longer responses are truncated.

### Auto-Reconnection

If the stream connection drops, the adapter automatically reconnects with exponential backoff (2s → 5s → 10s → 30s → 60s).

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot not responding to messages | Verify the robot capability is enabled in your app settings and that Stream Mode is selected. Check that your User ID is in `DINGTALK_ALLOWED_USERS`. Restart the gateway. |
| "dingtalk-stream not installed" error | Run `pip install dingtalk-stream httpx` |
| "DINGTALK_CLIENT_ID and DINGTALK_CLIENT_SECRET required" | Verify both are set correctly in `~/.hermes/.env`. The Client ID is your AppKey, the Client Secret is your AppSecret. |
| Stream disconnects / reconnection loops | The adapter automatically reconnects with exponential backoff. Check that your credentials are valid and your app has not been deactivated. Verify your network allows outbound WebSocket connections. |
| Bot is offline | Check `hermes gateway` is running. Common causes: wrong credentials, app deactivated, `dingtalk-stream` or `httpx` not installed. |
| "No session_webhook available" | The bot tried to reply but the webhook expired. Send a new message to the bot — each incoming message provides a fresh session webhook. This is a normal DingTalk platform limitation. |

---

## Security

Always set `DINGTALK_ALLOWED_USERS` to restrict who can interact with the bot. Without it, the gateway denies all users by default. Only add User IDs of people you trust — authorized users have full access to the agent's capabilities, including tool use and system access.

- Store credentials in `~/.hermes/.env` with file permissions `600`
- The Client Secret grants control of your DingTalk app — protect it like a password

---

## Changelog

- **v0.4.0:** DingTalk adapter added ([PR #1685](https://github.com/NousResearch/hermes-agent/pull/1685), [#1690](https://github.com/NousResearch/hermes-agent/pull/1690), [#1692](https://github.com/NousResearch/hermes-agent/pull/1692)).
- **v0.5.0:** Request timeouts added to keep the adapter responsive during network issues ([PR #3258](https://github.com/NousResearch/hermes-agent/pull/3258)).
