# Webhook

The webhook adapter lets external services trigger Hermes Agent runs by sending HTTP POST requests. When a webhook fires — from GitHub, GitLab, JIRA, Stripe, or any other service — the adapter validates the HMAC signature, extracts the payload, formats a prompt, runs the agent, and routes the response to a configured destination (a GitHub comment, a Telegram message, a Discord channel, etc.).

Added in v0.4.0 ([PR #2166](https://github.com/NousResearch/hermes-agent/pull/2166)).

---

## How It Works

1. The adapter starts an `aiohttp` HTTP server listening on a configured host and port (default `0.0.0.0:8644`).
2. External services POST to `http://your-server:8644/webhooks/{route_name}`.
3. The adapter validates the HMAC-SHA256 signature from the request header.
4. Idempotency cache (1-hour TTL) skips duplicate deliveries on webhook retries.
5. Rate limiting (configurable, default 30 requests per minute per route) protects against abuse.
6. The payload is parsed (JSON or form-encoded) and a prompt is rendered from a template.
7. The agent runs in a new isolated session keyed to `webhook:{route}:{delivery_id}`.
8. The response is delivered to the configured destination.

A `GET /health` endpoint returns `{"status": "ok", "platform": "webhook"}` for health checks.

---

## Prerequisites

```bash
pip install 'hermes-agent[webhook]'
# or
pip install aiohttp
```

The webhook adapter requires `aiohttp`. It is an optional dependency.

---

## Step 1: Configure Routes in config.yaml

The webhook adapter is configured entirely in `~/.hermes/config.yaml` under `platforms.webhook.extra.routes`. Each route is a named entry with the following fields:

```yaml
platforms:
  webhook:
    enabled: true
    extra:
      host: "0.0.0.0"         # Bind address (default: 0.0.0.0)
      port: 8644               # Listen port (default: 8644)
      secret: ""               # Global HMAC secret (used if route has no secret)
      rate_limit: 30           # Max requests per minute per route (default: 30)
      max_body_bytes: 1048576  # Max payload size in bytes (default: 1 MB)
      routes:
        github-pr-review:
          secret: "your-github-webhook-secret"    # REQUIRED
          events:                                  # Accept only these event types
            - pull_request
          prompt: |
            Review this pull request:
            Repository: {repository[full_name]}
            PR #: {number}
            Title: {pull_request[title]}
            Author: {pull_request[user][login]}
            URL: {pull_request[html_url]}
            Body: {pull_request[body]}
          skills:
            - code-review
          deliver: github_comment
          deliver_extra:
            repo: "{repository[full_name]}"
            pr_number: "{number}"

        stripe-payment:
          secret: "your-stripe-webhook-secret"
          events:
            - payment_intent.succeeded
          prompt: |
            Payment received: {amount} {currency} from {customer}.
          deliver: telegram
          deliver_extra:
            chat_id: "-1001234567890"
```

---

## Route Configuration Reference

| Field | Required | Description |
|-------|----------|-------------|
| `secret` | Yes | HMAC-SHA256 secret for signature validation. Set to `INSECURE_NO_AUTH` to skip (testing only). |
| `events` | No | List of accepted event types. Filtered from `X-GitHub-Event`, `X-GitLab-Event`, or `event_type` in the payload. Omit to accept all events. |
| `prompt` | No | Template string rendered with the parsed payload using Python `str.format_map()`. Nested keys use `{outer[inner]}` syntax. |
| `skills` | No | List of skill names to load for the agent. The first matching installed skill is loaded. |
| `deliver` | No | Where to send the response. See delivery targets below. Default: `log` (log to gateway output only). |
| `deliver_extra` | No | Additional delivery config. Supports `{key}` substitution from the payload. |

### Delivery Targets

| Value | Behavior |
|-------|----------|
| `log` | Response is logged to gateway output only. No message is sent. |
| `github_comment` | Posts a comment on a GitHub PR or issue. Requires `deliver_extra.repo` and `deliver_extra.pr_number`. |
| `telegram` | Sends to a Telegram chat. Requires `deliver_extra.chat_id`. The Telegram platform must be configured. |
| `discord` | Sends to a Discord channel. Requires `deliver_extra.chat_id`. The Discord platform must be configured. |
| `slack` | Sends to a Slack channel. Requires `deliver_extra.chat_id`. The Slack platform must be configured. |
| `signal` | Sends via Signal. Requires `deliver_extra.chat_id`. The Signal platform must be configured. |
| `sms` | Sends an SMS. Requires `deliver_extra.chat_id` (E.164 phone number). The SMS platform must be configured. |

---

## Step 2: Configure the External Service

Set the webhook URL in your external service:

```
http://your-server:8644/webhooks/github-pr-review
```

For HTTPS, place a reverse proxy (nginx, Caddy) in front of the aiohttp server.

### GitHub Webhooks

In your repository settings:
1. Go to **Settings → Webhooks → Add webhook**
2. Set **Payload URL** to `https://your-server/webhooks/github-pr-review`
3. Set **Content type** to `application/json`
4. Set **Secret** to the same value as `secret` in your route config
5. Choose which events to send

The adapter validates GitHub's `X-Hub-Signature-256` header automatically.

### GitLab Webhooks

In your project settings under **Webhooks**:
1. Set **URL** to `https://your-server/webhooks/your-route`
2. Set **Secret token** to your route's `secret`
3. Select the events to trigger the webhook

The adapter reads `X-GitLab-Token` for GitLab signature validation.

---

## Step 3: Start the Gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

On startup you should see:

```
[webhook] Listening on 0.0.0.0:8644 — routes: github-pr-review, stripe-payment
```

---

## Security

HMAC signature validation is **required** for every route. The adapter enforces this at startup — a route with no `secret` raises an error and prevents the gateway from starting. The only way to skip validation is to set `secret: INSECURE_NO_AUTH`, which should only be used for local testing.

Additional protections:
- **Rate limiting**: configurable per-minute limit per route (default 30 requests)
- **Idempotency**: `X-GitHub-Delivery` / `X-Request-ID` headers are cached for 1 hour to skip retried deliveries
- **Body size limit**: payloads over `max_body_bytes` (default 1 MB) are rejected with HTTP 413 before the body is read
- **Auth-before-body**: `Content-Length` is checked before reading the full payload

Store any HMAC secrets in `~/.hermes/.env` or directly in `config.yaml` with file permissions `600`.

---

## Dynamic Routes (Agent-Created Subscriptions)

The agent can create webhook subscriptions at runtime by writing to `~/.hermes/webhook_subscriptions.json`. The adapter hot-reloads this file on each incoming request (mtime-gated, so it is cheap). Dynamic routes are merged with static routes; static routes take precedence over dynamic ones with the same name.

This is used internally by the `cronjob` tool when it creates webhook-triggered jobs. You do not normally need to edit this file directly.

---

## Prompt Template Syntax

Prompts are rendered using Python's `str.format_map()` with the full parsed payload as the context. Use `{key}` for top-level keys and `{outer[inner]}` for nested keys.

**Example GitHub push payload template:**

```yaml
prompt: |
  New push to {repository[full_name]} on branch {ref}.
  Pushed by: {pusher[name]}
  Commits: {commits}
```

If a key is missing from the payload, the substitution is left as-is (no error).

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `[webhook] Route 'name' has no HMAC secret` on startup | Add a `secret` field to the route in `config.yaml`, or set a global `secret` under `extra`. |
| HTTP 401 Invalid signature | The `secret` in your route config does not match the secret configured in the external service. Copy the value exactly. |
| HTTP 404 Unknown route | The route name in the URL does not match any configured route name. Check for typos. |
| HTTP 429 Rate limit exceeded | The route is receiving more than `rate_limit` requests per minute. Increase `rate_limit` in config, or reduce retry frequency from the sender. |
| HTTP 413 Payload too large | The payload exceeds `max_body_bytes`. Increase the limit or reduce payload size. |
| Response not delivered | Check `deliver` and `deliver_extra` values. For cross-platform delivery (telegram, discord, etc.), the target platform must be enabled and connected. |
| Port already in use | Set a different port: `platforms.webhook.extra.port: 8645`. |
