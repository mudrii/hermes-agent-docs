# Nous Tool Gateway

The Nous Tool Gateway gives paid [Nous Portal](https://portal.nousresearch.com) subscribers automatic access to managed cloud tools — web search, image generation, text-to-speech, and browser automation — through their existing subscription, with no additional API keys.

**Available since:** v0.10.0 (v2026.4.16)

---

## Who Gets Access

The Tool Gateway is available to Nous Portal subscribers who are **not** on the free tier. Free-tier accounts are excluded.

When you run `hermes model` and select Nous Portal, Hermes authenticates via OAuth and stores your credentials in `~/.hermes/auth.json`. The gateway reads these credentials automatically — no extra configuration needed to get access.

---

## Covered Tools

| Tool | Vendor | Gateway vendor key |
|------|--------|--------------------|
| Web search & extraction | Firecrawl | `firecrawl` |
| Image generation | FAL (FLUX 2 Pro) | `fal-queue` |
| Text-to-speech | OpenAI Audio | `openai-audio` |
| Browser automation | Browser Use | `browser-use` |

---

## Enabling Gateway Tools

Enable per-tool via `use_gateway: true` in `~/.hermes/config.yaml`:

```yaml
web:
  use_gateway: true      # Firecrawl via Nous gateway (no FIRECRAWL_API_KEY needed)

image_gen:
  use_gateway: true      # FAL via Nous gateway (no FAL_KEY needed)

tts:
  use_gateway: true      # OpenAI TTS via Nous gateway (no OPENAI_API_KEY needed)

browser:
  use_gateway: true      # Browser Use via Nous gateway
```

You can also run `hermes model` → select Nous Portal → follow the tool setup prompts to configure this interactively.

When `use_gateway: true`, the gateway is used even if a direct API key (e.g., `FAL_KEY`) is also present.

---

## Checking Gateway Status

```bash
hermes status
hermes tools
```

`hermes status` shows gateway availability per tool. `hermes tools` shows whether each tool is using the gateway or a direct key.

---

## Priority: Gateway vs Direct Keys

| Condition | Result |
|-----------|--------|
| `use_gateway: true` + active subscription | Gateway used, direct key ignored |
| `use_gateway: false` or unset | Direct key used |
| No direct key, active subscription | Gateway used automatically |
| Free-tier subscription | Gateway not available |

---

## Environment Variable Overrides

Advanced overrides — most users do not need these.

| Variable | Purpose |
|----------|---------|
| `TOOL_GATEWAY_USER_TOKEN` | Override the Nous access token for gateway auth |
| `TOOL_GATEWAY_SCHEME` | Override URL scheme (`http` or `https`, default `https`) |
| `TOOL_GATEWAY_DOMAIN` | Override shared gateway domain (default `nousresearch.com`) |
| `FIRECRAWL_GATEWAY_URL` | Override Firecrawl gateway origin directly |
| `FAL_QUEUE_GATEWAY_URL` | Override FAL gateway origin directly |
| `OPENAI_AUDIO_GATEWAY_URL` | Override OpenAI Audio gateway origin directly |
| `BROWSER_USE_GATEWAY_URL` | Override Browser Use gateway origin directly |

---

## Troubleshooting

**Gateway shows as unavailable**
- Run `hermes model` and confirm you are logged in to Nous Portal
- Confirm your Nous subscription is not free-tier
- Run `hermes status` and look for gateway status per tool

**Token expiry**
Hermes auto-refreshes the Nous access token with a 120-second skew buffer. If you see auth errors, re-authenticate via `hermes model` → Nous Portal.

**Custom or self-hosted gateway**
Use `TOOL_GATEWAY_DOMAIN` to point at a custom domain, or per-vendor URL overrides listed above.
