# Provider Runtime Resolution

Hermes has a shared provider runtime resolver used across CLI, gateway, cron jobs, ACP, and auxiliary model calls.

Primary implementation files:

- `hermes_cli/runtime_provider.py` -- credential resolution, `_resolve_custom_runtime()`
- `hermes_cli/auth.py` -- provider registry, `resolve_provider()`
- `hermes_cli/model_switch.py` -- shared `/model` switch pipeline (CLI + gateway)
- `agent/auxiliary_client.py` -- auxiliary model routing
- `agent/model_metadata.py` -- context length resolution, token estimation

## Resolution Precedence

At a high level, provider resolution uses:

1. Explicit CLI/runtime request
2. `config.yaml` model/provider config
3. Environment variables
4. Provider-specific defaults or auto resolution

That ordering matters because Hermes treats the saved model/provider choice as the source of truth for normal runs. This prevents a stale shell export from silently overriding the endpoint a user last selected in `hermes model`.

## Providers

Current provider families include:

- AI Gateway (Vercel)
- OpenRouter
- Nous Portal
- OpenAI Codex
- Anthropic (native)
- Google AI Studio (`gemini`) -- added v0.8.0
- Z.AI
- Kimi / Moonshot
- MiniMax
- MiniMax China
- Custom (`provider: custom`) -- first-class provider for any OpenAI-compatible endpoint
- Named custom providers (`custom_providers` list in config.yaml)

## Output of Runtime Resolution

The runtime resolver returns a dict with:

- `provider` -- canonical provider id
- `api_mode` -- `"chat_completions"`, `"codex_responses"`, or `"anthropic_messages"`
- `base_url` -- target endpoint
- `api_key` -- resolved credential
- `source` -- where the credential came from (`env`, `portal`, `auth-store`, `explicit`)
- Provider-specific metadata like expiry/refresh info

## API Modes

The important abstraction is `api_mode`:

- Most providers use `chat_completions`
- Codex uses `codex_responses`
- Anthropic uses `anthropic_messages`
- A new non-OpenAI protocol means adding a new adapter and a new `api_mode` branch

## API Key Scoping

Hermes contains logic to avoid leaking the wrong API key to a custom endpoint when multiple provider keys exist. Each provider's API key is scoped to its own base URL:

- `OPENROUTER_API_KEY` is only sent to `openrouter.ai` endpoints
- `AI_GATEWAY_API_KEY` is only sent to `ai-gateway.vercel.sh` endpoints
- `OPENAI_API_KEY` is used for custom endpoints and as a fallback

## Native Anthropic Path

When provider resolution selects `anthropic`, Hermes uses the native Anthropic Messages API via `agent/anthropic_adapter.py`. Credential resolution prefers refreshable Claude Code credentials over copied env tokens when both are present.

## OpenAI Codex Path

Codex uses a separate Responses API path with `api_mode = codex_responses` and dedicated credential resolution.

## Auxiliary Model Routing

Auxiliary tasks (vision, web extraction, context compression summaries, session search summarization, memory flushes) can use their own provider/model routing rather than the main conversational model.

### Default Auxiliary Models Per Provider

| Provider | Default Auxiliary Model |
|----------|----------------------|
| OpenRouter | `google/gemini-3-flash-preview` |
| Nous Portal | `gemini-3-flash` |
| Anthropic | `claude-haiku-4-5-20251001` |
| Z.AI | `glm-4.5-flash` |
| Kimi/Moonshot | `kimi-k2-turbo-preview` |
| MiniMax | `MiniMax-M2.7-highspeed` |
| AI Gateway | `google/gemini-3-flash` |
| Codex | `gpt-5.2-codex` |

### Per-Task Override Configuration

Each auxiliary task supports per-task provider and model overrides via:

- Environment variables: `AUXILIARY_{TASK}_PROVIDER`, `AUXILIARY_{TASK}_MODEL`, `AUXILIARY_{TASK}_BASE_URL`, `AUXILIARY_{TASK}_API_KEY`
- Config file: `auxiliary.{task}.provider`, `auxiliary.{task}.model`, `auxiliary.{task}.base_url`
- Legacy: `compression.summary_provider`, `compression.summary_model`

Resolution priority: explicit args > env vars > config file > auto-detection.

## Fallback Models

Hermes supports a configured fallback model/provider pair, allowing runtime failover when the primary model encounters errors.

### How Fallback Works

1. `AIAgent.__init__` stores the `fallback_model` dict and sets `_fallback_activated = False`.
2. `_try_activate_fallback()` is called from three places in the main retry loop: after max retries on invalid API responses, on non-retryable client errors (HTTP 401, 403, 404), and after max retries on transient errors (HTTP 429, 500, 502, 503).
3. On activation, the agent swaps model, provider, base_url, api_mode, and client in-place. It re-evaluates prompt caching, sets `_fallback_activated = True` (one-shot), and resets retry count to 0.
4. Configuration flows through `CLI_CONFIG["fallback_model"]` (CLI) or `gateway/run.py._load_fallback_model()` (gateway). Both `provider` and `model` keys must be non-empty, or fallback is disabled.

### What Does NOT Support Fallback

- Subagent delegation: subagents inherit the parent's provider but not the fallback config
- Cron jobs: run with a fixed provider, no fallback mechanism
- Auxiliary tasks: use their own independent provider auto-detection chain

## Model Metadata

`agent/model_metadata.py` handles context length resolution, token estimation, and pricing extraction. It strips provider prefixes from model names (while preserving Ollama-style `model:tag` colons), caches context lengths, and provides rough token estimation functions used for pre-flight compression checks.

### models.dev Integration (v0.8.0)

`agent/models_dev.py` integrates the [models.dev](https://models.dev) registry as the primary source for provider and model metadata. The registry covers 4000+ models across 109+ providers and is fetched at most once per hour (background refresh), with a bundled offline snapshot for airgapped installs.

The `_infer_provider_from_url()` function in `model_metadata.py` maps well-known base URLs to models.dev provider IDs. This enables automatic context length resolution for any built-in or custom provider whose base URL is recognized:

```python
# _URL_TO_PROVIDER (partial)
"generativelanguage.googleapis.com": "gemini",
"inference-api.nousresearch.com": "nous",
"api.minimax": "minimax",
"openrouter.ai": "openrouter",
```

When adding a new provider, adding an entry to `_URL_TO_PROVIDER` is the recommended way to enable models.dev context length lookups for that provider's base URL.

### `/model` Command and Live Switching

The `/model` command pipeline is implemented in `hermes_cli/model_switch.py` (`switch_model()`) and shared between the CLI and all gateway platforms. It handles:

- `--provider` flag for explicit provider switching
- `--global` flag to persist the switch to `config.yaml`
- Alias resolution against the live models.dev catalog
- Aggregator-aware resolution: stays on OpenRouter or Nous Portal when the requested model is available there
- Cross-provider fallback when the model is not available on the current aggregator
- Model name normalization per provider (via `hermes_cli/model_normalize.py`)

The gateway handler in `gateway/run.py` (`_handle_model_command()`) adds platform-specific interactive pickers on Telegram and Discord when no model name is given, falling back to a text list on other platforms.

Model switches made with `/model` (without `--global`) persist for the session in `_session_model_overrides` and are **not** written to `config.yaml`. The agent receives a system note about the switch so it can update its self-identification.

Default auxiliary models per provider (used by `agent/auxiliary_client.py`):

| Provider | Default Auxiliary Model |
|----------|----------------------|
| OpenRouter | `google/gemini-3-flash-preview` |
| Nous Portal (paid) | `google/gemini-3-flash-preview` |
| Nous Portal (free tier) | `xiaomi/mimo-v2-pro` (text), `xiaomi/mimo-v2-omni` (vision) |
| Anthropic | `claude-haiku-4-5-20251001` |
| Google AI Studio | `gemini-3-flash-preview` |
| Z.AI | `glm-4.5-flash` |
| Kimi/Moonshot | `kimi-k2-turbo-preview` |
| MiniMax | `MiniMax-M2.7-highspeed` (rewritten to `/v1` endpoint) |
| AI Gateway | `google/gemini-3-flash` |
| Codex | `gpt-5.2-codex` |
