# WeCom (Enterprise WeChat)

WeCom (企业微信, also known as Enterprise WeChat) is Tencent's enterprise communication platform, deeply integrated with WeChat. The Hermes gateway adapter connects to WeCom using the official AI Bot WebSocket gateway — no public URL is required.

Added in **v0.6.0** ([PR #3847](https://github.com/NousResearch/hermes-agent/pull/3847)).

---

## Features

- **Text, image, and voice messages** — sends and receives all common message types
- **Group chats** — responds in WeCom group chats with configurable access control
- **Callback verification** — validates inbound WeCom events to prevent spoofing
- **Native media upload** — uploads images, video, and files through the WeCom API before sending
- **Configurable DM and group policies** — `open`, `allowlist`, `disabled`, or `pairing` per chat type

---

## Prerequisites

1. Log in to the [WeCom Admin Console](https://work.weixin.qq.com/wework_admin/loginpage_wx)
2. Go to **Apps** → **Self-Built** → **Create App**
3. Select **AI Bot** as the app type
4. Note the **Bot ID** and **Secret** displayed after creation
5. The adapter connects outbound to WeCom's WebSocket gateway — no callback URL setup is needed for standard operation

---

## Installation

The WeCom adapter requires `aiohttp` (already installed with Hermes by default) and optionally `httpx` for media downloads.

```bash
pip install aiohttp httpx
```

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `WECOM_BOT_ID` | Yes | — | Bot ID from the WeCom Admin Console |
| `WECOM_SECRET` | Yes | — | Bot Secret from the WeCom Admin Console |
| `WECOM_WEBSOCKET_URL` | No | `wss://openws.work.weixin.qq.com` | WeCom AI Bot WebSocket endpoint |
| `WECOM_DM_POLICY` | No | `open` | DM access policy: `open`, `allowlist`, `disabled`, or `pairing` |
| `WECOM_GROUP_POLICY` | No | `open` | Group access policy: `open`, `allowlist`, or `disabled` |

---

## config.yaml

```yaml
gateway:
  platforms:
    wecom:
      enabled: true
      extra:
        bot_id: "your-bot-id"          # or WECOM_BOT_ID env var
        secret: "your-secret"          # or WECOM_SECRET env var
        websocket_url: "wss://openws.work.weixin.qq.com"
        dm_policy: "open"              # open | allowlist | disabled | pairing
        allow_from:
          - "user_id_1"
        group_policy: "open"           # open | allowlist | disabled
        group_allow_from:
          - "group_id_1"
        groups:
          group_id_1:
            allow_from:
              - "user_id_1"
```

---

## Setup

```bash
hermes gateway setup
```

Select **WeCom** when prompted, then paste your Bot ID and Secret.

Start the gateway:

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
```

---

## Access Policies

### DM Policy (`WECOM_DM_POLICY`)

| Value | Behavior |
|-------|----------|
| `open` (default) | Any WeCom user can message the bot |
| `allowlist` | Only users listed in `allow_from` can message |
| `disabled` | Bot ignores all DMs |
| `pairing` | Unknown users receive a pairing code |

### Group Policy (`WECOM_GROUP_POLICY`)

| Value | Behavior |
|-------|----------|
| `open` (default) | Bot responds in any group it's added to |
| `allowlist` | Only listed groups are monitored |
| `disabled` | Bot ignores all group messages |

Per-group `allow_from` lists let you restrict which users can trigger the bot within specific groups, even when `group_policy` is `open`.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `WECOM_BOT_ID and WECOM_SECRET are required` | Set both env vars or configure `bot_id`/`secret` in config.yaml |
| Bot connects but does not respond | Check `dm_policy` — it defaults to `open`. Verify the bot app is published in WeCom Admin Console. |
| Group messages ignored | Check `group_policy` and ensure the bot has been added to the group. |
| Media upload failures | Verify the bot has the necessary permissions in the WeCom Admin Console. |
| WebSocket disconnects | The adapter reconnects automatically with exponential backoff. Check WeCom service status if disconnects are persistent. |

---

## Security

Treat `WECOM_BOT_ID` and `WECOM_SECRET` as credentials — store them in `~/.hermes/.env` with file permissions `600`. Use `dm_policy: allowlist` and `group_policy: allowlist` in production to restrict access to known users.
