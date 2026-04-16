# Nous Tool Gateway

The Tool Gateway lets paid [Nous Portal](https://portal.nousresearch.com) subscribers use web search, image generation, text-to-speech, and browser automation through their existing subscription. You do not need separate Firecrawl, FAL, OpenAI TTS, or Browser Use API keys when these tools are routed through the gateway.

## Included tools

| Tool | Runtime provider through the gateway | Typical direct-key alternative |
|------|--------------------------------------|--------------------------------|
| Web search and extract | Firecrawl | `FIRECRAWL_API_KEY`, `EXA_API_KEY`, `PARALLEL_API_KEY`, `TAVILY_API_KEY` |
| Image generation | FAL / FLUX 2 Pro | `FAL_KEY` |
| Text-to-speech | OpenAI TTS | `VOICE_TOOLS_OPENAI_KEY`, `ELEVENLABS_API_KEY` |
| Browser automation | Browser Use | `BROWSER_USE_API_KEY`, `BROWSERBASE_API_KEY` |

## Eligibility

The Tool Gateway is available to paid Nous Portal subscribers. Free-tier accounts can still use the same tools with direct API keys.

To inspect your current state:

```bash
hermes status
```

Look for the `Nous Tool Gateway` section. It reports which tools are active through the gateway, which are using direct credentials, and which are still unconfigured.

## Enabling it

### During `hermes model`

When you run `hermes model` and choose `Nous Portal`, Hermes offers to enable the Tool Gateway automatically for supported tools.

### Through `hermes tools`

You can also enable it per tool:

```bash
hermes tools
```

Choose a tool category such as Web, Browser, Image Generation, or TTS, then select the Nous subscription-backed provider.

### Manual config

Set `use_gateway: true` for each tool you want to route through the subscription:

```yaml
web:
  backend: firecrawl
  use_gateway: true

image_gen:
  use_gateway: true

tts:
  provider: openai
  use_gateway: true

browser:
  cloud_provider: browser-use
  use_gateway: true
```

## Routing behavior

The gateway is checked first when `use_gateway: true` is set:

- `use_gateway: true` routes through the Tool Gateway even if a direct API key is also present.
- `use_gateway: false` or an absent flag uses direct provider credentials first.

This means you can mix and match. For example, you can use the Tool Gateway for web search and image generation while keeping ElevenLabs for TTS.

## Disabling it

To switch a tool back to direct credentials, either select a direct provider in `hermes tools` or set `use_gateway: false` in `config.yaml`.

```yaml
web:
  backend: firecrawl
  use_gateway: false
```

## Advanced environment variables

Most users do not need to set these manually, but they are available for custom gateway deployments:

| Variable | Description |
|----------|-------------|
| `TOOL_GATEWAY_DOMAIN` | Base domain for Tool Gateway routing |
| `TOOL_GATEWAY_SCHEME` | HTTP or HTTPS scheme |
| `TOOL_GATEWAY_USER_TOKEN` | Auth token for the Tool Gateway, normally populated automatically |
| `FIRECRAWL_GATEWAY_URL` | Override the Firecrawl gateway endpoint specifically |

## Notes

- Existing direct API keys do not need to be removed. They remain in `.env` and are simply bypassed while `use_gateway: true` is active.
- The Tool Gateway works the same way in the CLI, messaging gateway platforms, cron jobs, and the dashboard because routing happens at the tool-runtime level.
- Modal remains separate. It is not enabled by the Tool Gateway prompt.

