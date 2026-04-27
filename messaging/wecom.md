# WeCom (Enterprise WeChat)

WeCom (企业微信, also known as Enterprise WeChat) is Tencent's enterprise communication platform, deeply integrated with WeChat. The Hermes gateway adapter connects to WeCom using the official AI Bot WebSocket gateway — no public URL is required.

Current as of Hermes Agent **v0.11.0** (`v2026.4.23`). Originally added in **v0.6.0** ([PR #3847](https://github.com/NousResearch/hermes-agent/pull/3847)); v0.11.0 introduces the **Scan-to-Create** setup wizard, which provisions a brand-new WeCom AI Bot directly from your terminal via a QR scan.

---

## Features

- **Text, image, and voice messages** — sends and receives all common message types
- **Group chats** — responds in WeCom group chats with configurable access control
- **Callback verification** — validates inbound WeCom events to prevent spoofing
- **Native media upload** — uploads images, video, and files through the WeCom API before sending
- **Configurable DM and group policies** — `open`, `allowlist`, `disabled`, or `pairing` per chat type

---

## Prerequisites

- A WeCom organization account with permission to create apps
- The Hermes Agent gateway installed (`pip install "hermes-agent[messaging]"` already pulls in `aiohttp` + `httpx`)
- The adapter connects outbound to WeCom's WebSocket gateway, so no public URL, callback endpoint, or firewall rule is required

```bash
pip install aiohttp httpx
```

---

## Setup

### Step 1: Create the AI Bot

#### Recommended: Scan-to-Create (one command)

In v0.11.0 the gateway can provision a brand-new WeCom AI Bot for you from a single command:

```bash
hermes gateway setup
```

Select **WeCom** and scan the QR code that prints in your terminal with the WeCom mobile app. The wizard will:

1. Display a QR code in your terminal.
2. Wait for you to scan it with the WeCom mobile app.
3. Automatically create the AI Bot application with the correct permissions in your tenant.
4. Retrieve the **Bot ID** and **Secret** and write them to `~/.hermes/.env`.
5. Walk you through access-control configuration (`dm_policy`, `group_policy`, allowlists, home channel).

No round-trip to the WeCom Admin Console is required — the bot exists, is named, has its credentials saved, and is ready for traffic by the time the wizard exits.

#### Alternative: Manual Setup

If scan-to-create is unavailable (older WeCom mobile clients, or you prefer the web console), the wizard falls back to manual input:

1. Log in to the [WeCom Admin Console](https://work.weixin.qq.com/wework_admin/frame).
2. Go to **Apps** → **Self-Built** → **Create App** and select **AI Bot** as the app type.
3. Note the **Bot ID** and **Secret** shown after creation.
4. Run `hermes gateway setup`, select **WeCom**, choose **Manual entry**, and paste the Bot ID and Secret when prompted.

:::warning
Keep the **Bot Secret** private — anyone with it can impersonate your bot. Store it in `~/.hermes/.env` with file permissions `600` and never commit it to Git.
:::

### Step 2: Start the gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
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

### Per-Group User Restrictions

Within the `groups` config block, each group can define its own `allow_from` list of user IDs. Only those users can trigger the bot in that specific group, regardless of the global `group_policy`:

```yaml
groups:
  group_id_1:
    allow_from:
      - "user_id_1"
      - "user_id_2"
  group_id_2:
    allow_from:
      - "user_id_3"
```

---

## AI Bot WebSocket Transport

The adapter connects outbound to WeCom's AI Bot WebSocket gateway — no public URL, callback endpoint, or firewall rule is required. The connection is initiated by Hermes, so the bot works behind NAT, firewalls, and private networks without any tunnel or port forwarding.

### Heartbeat

The adapter sends a WebSocket **ping frame every 30 seconds** to keep the connection alive. If no pong is received within the timeout window, the connection is considered dead and the adapter reconnects with exponential backoff.

---

## Media Limits

| Media type | Maximum size |
|------------|-------------|
| Images | 10 MB |
| Video | 10 MB |
| Voice | 2 MB |

Files exceeding these limits are rejected by the WeCom API. The adapter checks file size before uploading and logs a warning if the limit is exceeded.

---

## Deduplication

The adapter maintains an in-memory dedup cache to prevent processing the same message twice (WeCom may retry delivery on transient failures). The cache uses a **300-second (5-minute) window** with a maximum of **1,000 entries**. When the cache reaches capacity, the oldest entries are evicted. Message IDs that appear in the cache within the window are silently dropped.

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
