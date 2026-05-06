# Provider Runtime Resolution

Hermes has a shared provider runtime resolver used across CLI, gateway, cron jobs, ACP, and auxiliary model calls.

Primary implementation files:

- `hermes_cli/runtime_provider.py` -- credential resolution, `_resolve_custom_runtime()`
- `hermes_cli/auth.py` -- provider registry, `resolve_provider()`
- `hermes_cli/model_switch.py` -- shared `/model` switch pipeline (CLI + gateway)
- `agent/auxiliary_client.py` -- auxiliary model routing
- `agent/model_metadata.py` -- context length resolution, token estimation
- `providers/` -- ABC + registry entry points (`ProviderProfile`, `register_provider`, `get_provider_profile`, `list_providers`)
- `plugins/model-providers/<name>/` -- per-provider plugins (bundled) that declare `api_mode`, `base_url`, `env_vars`, `fallback_models` and register themselves into the registry on first access. User plugins at `$HERMES_HOME/plugins/model-providers/<name>/` override bundled ones of the same name.

`get_provider_profile()` in `providers/` returns a `ProviderProfile` for a given provider id. `runtime_provider.py` calls this at resolution time to get the canonical `base_url`, `env_vars` priority list, `api_mode`, and `fallback_models` without needing to duplicate that data in multiple files. Adding a new plugin under `plugins/model-providers/<your-provider>/` (or `$HERMES_HOME/plugins/model-providers/<your-provider>/`) that calls `register_provider()` is enough for `runtime_provider.py` to pick it up — no branch needed in the resolver itself.

## Resolution Precedence

At a high level, provider resolution uses:

1. Explicit CLI/runtime request
2. `config.yaml` model/provider config
3. Environment variables
4. Provider-specific defaults or auto resolution

That ordering matters because Hermes treats the saved model/provider choice as the source of truth for normal runs. This prevents a stale shell export from silently overriding the endpoint a user last selected in `hermes model`.

## Providers

Current provider families include:

- AI Gateway (Vercel) -- with pricing, attribution, and dynamic model discovery (v0.11.0)
- OpenRouter
- Nous Portal
- OpenAI Codex (Responses API; GPT-5 and GPT-5.5 over Codex OAuth in v0.11.0)
- Anthropic (native)
- Google AI Studio (`gemini`) -- added v0.8.0
- Google Gemini CLI OAuth (`google-gemini-cli`) -- added v0.11.0; uses `HERMES_GEMINI_PROJECT_ID`
- AWS Bedrock (`bedrock`) -- native Converse API path, IAM auth, region-aware discovery, inference profiles, Guardrails (v0.11.0)
- NVIDIA NIM (`nvidia`) -- added v0.11.0
- Step Plan -- added v0.11.0
- Arcee AI -- added v0.11.0
- Azure AI Foundry (`azure-foundry`) -- first-class provider in v0.12.0
- Qwen Portal OAuth (`qwen-oauth`) -- v0.11.0 flavor
- Z.AI
- Kimi / Moonshot
- MiniMax
- MiniMax China
- MiniMax OAuth (`minimax-oauth`) -- browser OAuth provider in v0.12.0
- GMI Cloud (`gmi`, aliases `gmi-cloud` / `gmicloud`) -- first-class provider in v0.12.0
- LM Studio (`lmstudio`) -- first-class local provider in v0.12.0
- Tencent Tokenhub (`tencent-tokenhub`) -- first-class provider in v0.12.0
- xAI / Grok (Responses API)
- Mistral
- Ollama Cloud (`ollama-cloud`)
- Custom (`provider: custom`) -- first-class provider for any OpenAI-compatible endpoint
- Named custom providers (`custom_providers` list in config.yaml)

## Output of Runtime Resolution

The runtime resolver returns a dict with:

- `provider` -- canonical provider id
- `api_mode` -- `"chat_completions"`, `"codex_responses"`, `"anthropic_messages"`, or `"bedrock_converse"`
- `base_url` -- target endpoint
- `api_key` -- resolved credential (or IAM-scoped boto3 session for Bedrock)
- `source` -- where the credential came from (`env`, `portal`, `auth-store`, `explicit`)
- Provider-specific metadata like expiry/refresh info

## API Modes

The important abstraction is `api_mode`. In v0.11.0 each `api_mode` is owned by a class under `agent/transports/` (see [Transport Layer](./transport-layer.md)):

| `api_mode` | Used by | Transport class |
|------------|---------|-----------------|
| `chat_completions` | OpenRouter, Nous Portal, NVIDIA NIM, Step Plan, Arcee, Vercel ai-gateway, Ollama Cloud, Mistral, Qwen, DeepSeek, Z.AI, Kimi, GMI Cloud, Tencent Tokenhub, LM Studio, Azure OpenAI-style Foundry endpoints, custom OpenAI-compatible endpoints | `ChatCompletionsTransport` |
| `codex_responses` | OpenAI Codex (GPT-5 and GPT-5.5 over Codex OAuth), xAI Grok in Responses mode | `ResponsesApiTransport` |
| `anthropic_messages` | Native Anthropic, MiniMax, MiniMax OAuth, Azure Anthropic-style Foundry endpoints | `AnthropicTransport` |
| `bedrock_converse` | AWS Bedrock | `BedrockTransport` |

Adding a new non-OpenAI protocol means adding a new transport subclass and a new `api_mode` string. See [Transport Layer](./transport-layer.md) for the ABC contract and [Adding Providers](./adding-providers.md) for the full provider-side checklist.

## API Key Scoping

Hermes contains logic to avoid leaking the wrong API key to a custom endpoint when multiple provider keys exist. Each provider's API key is scoped to its own base URL:

- `OPENROUTER_API_KEY` is only sent to `openrouter.ai` endpoints
- `AI_GATEWAY_API_KEY` is only sent to `ai-gateway.vercel.sh` endpoints
- `OPENAI_API_KEY` is used for custom endpoints and as a fallback

## Native Anthropic Path

When provider resolution selects `anthropic`, Hermes uses the native Anthropic Messages API via `agent/anthropic_adapter.py`, dispatched through `AnthropicTransport`. Credential resolution prefers refreshable Claude Code credentials over copied env tokens when both are present.

## OpenAI Codex / Responses API Path

Codex uses the OpenAI Responses API with `api_mode = codex_responses`, dispatched through `ResponsesApiTransport`. v0.11.0 adds GPT-5.5 over Codex OAuth and live model discovery. xAI Grok in Responses mode shares this transport.

## AWS Bedrock Path (v0.11.0)

Native AWS Bedrock support ships in v0.11.0 with `api_mode = bedrock_converse`, dispatched through `BedrockTransport`. The transport wraps the boto3 Converse API; runtime resolution returns IAM-scoped credentials (env vars, `AWS_PROFILE`, or instance metadata) rather than a plain `api_key`. Region-aware model discovery picks the right inference profile prefix (`us.` for US-region profiles, `global.` for global profiles), and Bedrock Guardrails are configured per-call. Boto3 timeouts are set explicitly to avoid the default 60s socket timeout silently truncating long completions.

## NVIDIA NIM Path (v0.11.0)

`nvidia` uses `NVIDIA_API_KEY` (env) and `NVIDIA_BASE_URL` (override; defaults to NVIDIA's hosted NIM). It is OpenAI-compatible, so it dispatches through `ChatCompletionsTransport`. No new transport is required.

## Step Plan Path (v0.11.0)

Step Plan is added as an OpenAI-compatible provider; it dispatches through `ChatCompletionsTransport`.

## Google Gemini CLI OAuth Path (v0.11.0)

`google-gemini-cli` reuses Gemini CLI's locally cached OAuth credentials. Runtime resolution requires `HERMES_GEMINI_PROJECT_ID` and falls back to chat-completions dispatch. (Google AI Studio via API key continues to use the OpenAI-compatible `/v1beta/openai` endpoint.)

## Vercel ai-gateway Path (v0.11.0)

`ai-gateway` dispatches through `ChatCompletionsTransport` and additionally pulls live pricing, attribution metadata, and dynamic model discovery from the gateway's catalog endpoint. Aggregator-aware `/model` resolution stays on `ai-gateway` when the requested model is available there.

## Azure AI Foundry Path (v0.12.0)

`azure-foundry` supports Azure OpenAI-style `/openai/v1` endpoints and Anthropic-style Azure endpoints. Hermes normalizes the configured URL, can detect the endpoint shape, and dispatches through either `ChatCompletionsTransport` or the Anthropic Messages path.

## LM Studio Path (v0.12.0)

`lmstudio` is a first-class local provider instead of a generic custom endpoint. It supports optional `LM_API_KEY`, `LM_BASE_URL`, live `/models` probing, `hermes doctor` checks, and LM Studio reasoning-effort handling.

## MiniMax OAuth Path (v0.12.0)

`minimax-oauth` uses MiniMax's browser OAuth flow and stores refreshable credentials in Hermes auth state. It shares the Anthropic Messages-compatible runtime path with API-key MiniMax providers.

## GMI Cloud and Tencent Tokenhub (v0.12.0)

`gmi` / `gmi-cloud` and `tencent-tokenhub` are first-class provider IDs in the registry. They resolve through the shared runtime and model-selection path instead of requiring a custom-provider entry.

## Qwen Portal OAuth Path (v0.11.0)

`qwen-oauth` reuses Qwen Portal OAuth credentials with `HERMES_QWEN_BASE_URL` for endpoint override. Dispatch is via `ChatCompletionsTransport`.

## Auxiliary Model Routing

Auxiliary tasks can use their own provider/model routing rather than the main conversational model. The v0.12.0 canonical slots are: `vision`, `web_extract`, `compression`, `session_search`, `skills_hub`, `approval`, `mcp`, `title_generation`, and `curator`.

### Default Auxiliary Models Per Provider

| Provider | Default Auxiliary Model |
|----------|----------------------|
| OpenRouter | `google/gemini-3-flash-preview` |
| Nous Portal | `google/gemini-3-flash-preview` |
| Anthropic | `claude-haiku-4-5-20251001` |
| Z.AI | `glm-4.5-flash` |
| Kimi/Moonshot | `kimi-k2-turbo-preview` |
| MiniMax | `MiniMax-M2.7` |
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
- Cross-provider fallback when the model is not available on OpenRouter or Nous Portal
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
| MiniMax | `MiniMax-M2.7` (rewritten to `/v1` endpoint) |
| AI Gateway | `google/gemini-3-flash` |
| Codex | `gpt-5.2-codex` |
