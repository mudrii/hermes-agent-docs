# Feishu / Lark

Feishu (飞书) is ByteDance's enterprise collaboration platform, sold internationally as Lark. It provides messaging, video calls, documents, and calendar in a single app. The Hermes gateway adapter connects to Feishu or Lark via the official open-platform SDK, supporting both WebSocket long-connection and Webhook transport modes.

Added in **v0.6.0** ([PR #3799](https://github.com/NousResearch/hermes-agent/pull/3799), [PR #3817](https://github.com/NousResearch/hermes-agent/pull/3817), closes [#1788](https://github.com/NousResearch/hermes-agent/issues/1788)).

---

## Features

- **Event subscriptions** — receives message events via WebSocket long connection or Webhook
- **Message cards** — sends rich interactive cards as responses
- **Group chat with @mention gating** — responds in group chats only when @mentioned (configurable)
- **Image and file attachments** — downloads inbound images, audio, video, and documents for agent context; uploads outbound media natively
- **Interactive card callbacks** — button-click events on sent cards are routed as synthetic commands to the agent
- **Persistent deduplication** — tracks message IDs across restarts to prevent duplicate processing
- **ACK emoji reaction** — adds an "OK" emoji reaction to inbound messages to signal receipt

---

## Prerequisites

1. Go to the [Feishu Open Platform](https://open.feishu.cn) (or [Lark Developer Portal](https://open.larksuite.com) for international Lark)
2. Click **Create App** and select **Custom App**
3. Under **Features**, enable **Bot** (Messaging)
4. Under **Permissions**, add:
   - `im:message` — read and send messages
   - `im:message.group_at_msg` — receive group @mention messages
   - `im:resource` — download images and files
5. Under **Event Subscriptions**, subscribe to:
   - `im.message.receive_v1` — receive messages
   - `im.message.reaction.created_v1` — receive reactions (optional)
6. Set the **Callback URL** (Webhook mode) or leave blank for WebSocket mode
7. Publish the app to make it available in your organization

---

## Installation

Install the `lark-oapi` Python package:

```bash
pip install lark-oapi
```

Or install the Hermes extras:

```bash
pip install 'hermes-agent[feishu]'
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `FEISHU_APP_ID` | Yes | — | App ID from the Feishu Open Platform |
| `FEISHU_APP_SECRET` | Yes | — | App Secret from the Feishu Open Platform |
| `FEISHU_DOMAIN` | No | `feishu` | Set to `lark` for international Lark deployments |
| `FEISHU_CONNECTION_MODE` | No | `websocket` | `websocket` or `webhook` |
| `FEISHU_ENCRYPT_KEY` | No | — | Webhook payload encryption key (Webhook mode only) |
| `FEISHU_VERIFICATION_TOKEN` | No | — | Webhook verification token (Webhook mode only) |
| `FEISHU_GROUP_POLICY` | No | `allowlist` | Group access policy: `allowlist`, `open`, or `disabled` |
| `FEISHU_ALLOWED_USERS` | No | — | Comma-separated Feishu open_ids for the allowlist |
| `FEISHU_BOT_OPEN_ID` | No | — | Bot's own open_id (auto-detected if omitted) |
| `FEISHU_BOT_USER_ID` | No | — | Bot's user_id (auto-detected if omitted) |
| `FEISHU_BOT_NAME` | No | — | Bot's display name (auto-detected if omitted) |
| `FEISHU_WEBHOOK_HOST` | No | `0.0.0.0` | Bind host for Webhook mode |
| `FEISHU_WEBHOOK_PORT` | No | `8080` | Bind port for Webhook mode |
| `FEISHU_WEBHOOK_PATH` | No | `/feishu` | HTTP path for Webhook mode |

---

## config.yaml

```yaml
gateway:
  platforms:
    feishu:
      enabled: true
      extra:
        app_id: "cli_your_app_id"          # or FEISHU_APP_ID env var
        app_secret: "your_app_secret"      # or FEISHU_APP_SECRET env var
        domain: "feishu"                   # "feishu" (default) or "lark"
        connection_mode: "websocket"       # "websocket" (default) or "webhook"
        # Webhook mode only:
        # webhook_host: "0.0.0.0"
        # webhook_port: 8080
        # webhook_path: "/feishu"
```

---

## Setup

```bash
hermes gateway setup feishu
hermes gateway start
```

Or use the interactive wizard:

```bash
hermes gateway setup
```

Select **Feishu** when prompted.

---

## Connection Modes

### WebSocket (default)

The adapter establishes a long-lived WebSocket connection to Feishu's servers. No public URL is required — suitable for development and private deployments.

### Webhook

The adapter runs an HTTP server that Feishu pushes events to. Requires a public HTTPS URL. Set `FEISHU_CONNECTION_MODE=webhook` and configure `FEISHU_ENCRYPT_KEY` and `FEISHU_VERIFICATION_TOKEN` from the Feishu Open Platform.

Configure the Webhook URL in the Feishu app settings to point to your public endpoint, e.g. `https://your-server.example.com/feishu`.

---

## Group Mention Behavior

The `FEISHU_GROUP_POLICY` environment variable controls how the bot responds in group chats:

| Value | Behavior |
|-------|----------|
| `allowlist` (default) | Only responds to users listed in `FEISHU_ALLOWED_USERS` |
| `open` | Responds to all @mentions in any group |
| `disabled` | Does not respond in group chats |

In direct messages, `FEISHU_GROUP_POLICY` does not apply — the standard `FEISHU_ALLOWED_USERS` allowlist controls access.

---

## Lark (International)

To connect to the international Lark platform instead of Feishu:

```bash
FEISHU_DOMAIN=lark
```

The app must be registered on [open.larksuite.com](https://open.larksuite.com) rather than open.feishu.cn.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `lark-oapi not installed` | Run `pip install lark-oapi` |
| `FEISHU_APP_ID or FEISHU_APP_SECRET not set` | Set both env vars or `app_id`/`app_secret` in config.yaml |
| Bot not receiving group messages | Enable `im.message.group_at_msg` permission and republish the app |
| Webhook events not arriving | Verify the callback URL is publicly reachable. Check `FEISHU_VERIFICATION_TOKEN` matches the value in the Feishu app settings. |
| Messages deduplicated / dropped | The adapter tracks message IDs for 24 hours. Duplicate deliveries from Feishu are silently ignored. |

---

## Security

Set `FEISHU_ALLOWED_USERS` to restrict which Feishu users can interact with the bot. Without it, the gateway denies all users by default. Treat `FEISHU_APP_SECRET`, `FEISHU_ENCRYPT_KEY`, and `FEISHU_VERIFICATION_TOKEN` as secrets — store them in `~/.hermes/.env` with file permissions `600`.
