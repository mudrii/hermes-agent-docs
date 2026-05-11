# DingTalk

Hermes Agent integrates with DingTalk (钉钉) as a chatbot, letting you chat with your AI assistant through direct messages or group chats. The bot connects via DingTalk's Stream Mode — a long-lived WebSocket connection that requires no public URL or webhook server — and replies using markdown-formatted messages through DingTalk's session webhook API.

Current as of Hermes Agent **v0.11.0** (`v2026.4.23`). Highlights since v0.8.0: a QR-code device-flow that prints the auth URL in your terminal, AI Cards with streaming updates, emoji reactions for processing status, and the `alibabacloud-dingtalk` SDK for media downloads.

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

Install the required Python packages. The simplest path uses the bundled extra:

```bash
pip install "hermes-agent[dingtalk]"
```

Or install them individually if you prefer:

```bash
pip install dingtalk-stream httpx alibabacloud-dingtalk
```

- `dingtalk-stream` — DingTalk's official SDK for Stream Mode (WebSocket-based real-time messaging)
- `httpx` — async HTTP client used for sending replies via session webhooks
- `alibabacloud-dingtalk` — DingTalk OpenAPI SDK for AI Cards, emoji reactions, and media downloads

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

Select **DingTalk** when prompted. The setup wizard authorizes via one of two paths:

- **QR-code device flow (recommended).** Scan the QR that prints in your terminal with the DingTalk mobile app — your Client ID and Client Secret are returned automatically and written to `~/.hermes/.env`. No developer-console trip needed.
- **Manual paste.** If you already have credentials (or QR scanning isn't convenient), paste your Client ID, Client Secret, and allowed user IDs when prompted.

:::note openClaw branding disclosure
DingTalk's `verification_uri_complete` is hardcoded to the **openClaw** identity at the API layer, so the QR currently authorizes under an `openClaw` source string until Alibaba / DingTalk-Real-AI registers a Hermes-specific template server-side. This is purely how DingTalk presents the consent screen — the bot you create is fully yours and private to your tenant.
:::

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

DingTalk auto-enables when both `DINGTALK_CLIENT_ID` and `DINGTALK_CLIENT_SECRET` are present. You can also enable it explicitly in `~/.hermes/gateway.json` or `~/.hermes/config.yaml`:

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

gateway:
  platforms:
    dingtalk:
      extra:
        # Optional chat whitelist. When set, Hermes ignores all other chats.
        allowed_chats:
          - cidXXXX==
```

`group_sessions_per_user: true` (the default) keeps each participant's context isolated inside shared group chats. Set it to `false` only if you explicitly want one shared conversation for the entire group.
`allowed_chats` under `dingtalk.extra` is a hard chat whitelist; messages from other chats are ignored.

---

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `DINGTALK_CLIENT_ID` | Yes | Client ID (AppKey) from the DingTalk Developer Console |
| `DINGTALK_CLIENT_SECRET` | Yes | Client Secret (AppSecret) from the DingTalk Developer Console |
| `DINGTALK_ALLOWED_USERS` | Recommended | Comma-separated DingTalk User IDs authorized to talk to the bot |
| `DINGTALK_ALLOW_ALL_USERS` | No | Allow all users (not recommended — anyone in your tenant can talk to the bot) |
| `DINGTALK_REQUIRE_MENTION` | No | When `true` (default), the bot only responds in group chats when directly `@mentioned`. DMs always get a response. |
| `DINGTALK_CARD_TEMPLATE_ID` | No | AI Card template ID for streaming card replies; falls back to plain markdown when unset. |

---

## Platform-Specific Features

### Stream Mode

No public URL, domain name, or webhook server is needed. The connection is initiated from your machine via WebSocket, so it works behind NAT and firewalls.

### Mention Gating

In group chats, Hermes only replies when directly `@mentioned` (controlled by `DINGTALK_REQUIRE_MENTION`, default `true`). DMs always get a response. Combine with `DINGTALK_ALLOWED_USERS` to limit who is even allowed to mention the bot — unauthorized senders are silently ignored.

### AI Cards

Hermes can reply using DingTalk AI Cards instead of plain markdown messages. Cards provide a richer, more structured display and support streaming updates as the agent generates its response.

To enable AI Cards, configure a card template ID in `~/.hermes/config.yaml`:

```yaml
platforms:
  dingtalk:
    enabled: true
    extra:
      card_template_id: "your-card-template-id"
```

You can find your card template ID in the DingTalk Developer Console under your app's AI Card settings. When AI Cards are enabled, all replies are sent as cards with streaming text updates.

### Emoji Reactions

Hermes automatically adds emoji reactions to your messages to show processing status:

- **🤔 Thinking** — added when the bot starts processing your message.
- **🥳 Done** — added when the response is complete (replaces the Thinking reaction).

These reactions work in both DMs and group chats and require the `alibabacloud-dingtalk` SDK.

### Media Handling

Images and files in incoming messages are automatically downloaded via the DingTalk OpenAPI and made available to vision tools and other media-aware tools. No special configuration is required beyond installing the `alibabacloud-dingtalk` SDK.

### Display Settings

You can customize DingTalk's display behavior independently from other platforms:

```yaml
display:
  platforms:
    dingtalk:
      show_reasoning: false   # Show model reasoning/thinking in replies
      streaming: true         # Enable streaming responses (works with AI Cards)
      tool_progress: all      # Show tool execution progress (all/new/off)
      interim_assistant_messages: true  # Show intermediate commentary messages
```

To disable tool progress and intermediate messages for a cleaner experience:

```yaml
display:
  platforms:
    dingtalk:
      tool_progress: off
      interim_assistant_messages: false
```

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
- **v0.11.0:** QR-code device-flow setup, AI Cards with streaming updates, emoji reactions for processing status, media downloads via `alibabacloud-dingtalk`, and `require_mention` gating for group chats.
