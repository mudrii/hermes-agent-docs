# Providers

This document covers every inference provider supported by Hermes Agent, how credential resolution works, provider routing, smart model routing, fallback providers, and how to add a custom provider.

---

## Supported Providers

All providers are registered in `PROVIDER_REGISTRY` in `hermes_cli/auth.py`. The canonical provider ID is used in `--provider` flags, `config.yaml`, and environment variables.

| Provider ID | Name | Auth Type | Base URL |
|-------------|------|-----------|----------|
| `openrouter` | OpenRouter | API key | `https://openrouter.ai/api/v1` |
| `nous` | Nous Portal | OAuth device code | `https://portal.nousresearch.com` → `https://inference-api.nousresearch.com/v1` |
| `openai-codex` | OpenAI Codex | OAuth external (ChatGPT) | `https://chatgpt.com/backend-api/codex` |
| `copilot` | GitHub Copilot | API key (GitHub token) | `https://api.githubcopilot.com` |
| `copilot-acp` | GitHub Copilot ACP | External process | `acp://copilot` |
| `anthropic` | Anthropic | API key / Claude Code OAuth | `https://api.anthropic.com` |
| `ai-gateway` | Vercel AI Gateway | API key | `https://ai-gateway.vercel.sh/v1` |
| `zai` | Z.AI / ZhipuAI GLM | API key | `https://api.z.ai/api/paas/v4` |
| `kimi-coding` | Kimi / Moonshot AI | API key | `https://api.moonshot.ai/v1` (or `https://api.kimi.com/coding/v1` for `sk-kimi-` keys) |
| `minimax` | MiniMax (global) | API key | `https://api.minimax.io/v1` |
| `minimax-cn` | MiniMax (China) | API key | `https://api.minimaxi.com/v1` |
| `alibaba` | Alibaba Cloud (DashScope) | API key | `https://dashscope-intl.aliyuncs.com/apps/anthropic` |
| `deepseek` | DeepSeek | API key | `https://api.deepseek.com/v1` |
| `kilocode` | Kilo Code | API key | `https://api.kilo.ai/api/gateway` |
| `opencode-zen` | OpenCode Zen | API key | `https://opencode.ai/zen/v1` |
| `opencode-go` | OpenCode Go | API key | `https://opencode.ai/zen/go/v1` |
| custom | Any OpenAI-compatible endpoint | API key | User-configured |

---

## Provider Detail

### OpenRouter

Recommended for flexibility — routes to hundreds of models through a single API key.

**Setup:**

```bash
hermes config set OPENROUTER_API_KEY sk-or-...
```

**Key environment variables:**

| Variable | Description |
|----------|-------------|
| `OPENROUTER_API_KEY` | OpenRouter API key (required) |
| `OPENROUTER_BASE_URL` | Override the OpenRouter-compatible base URL |

**Model syntax:**

```bash
hermes chat --provider openrouter --model anthropic/claude-sonnet-4.6
hermes chat --model anthropic/claude-sonnet-4:nitro     # throughput sorting shortcut
hermes chat --model anthropic/claude-sonnet-4:floor     # price sorting shortcut
```

**Static model catalog (from `hermes_cli/models.py`):**

```
anthropic/claude-opus-4.6       (recommended)
anthropic/claude-sonnet-4.5
anthropic/claude-haiku-4.5
openai/gpt-5.4
openai/gpt-5.4-mini
openrouter/hunter-alpha         (free)
openrouter/healer-alpha         (free)
openai/gpt-5.3-codex
google/gemini-3-pro-preview
google/gemini-3-flash-preview
qwen/qwen3.5-plus-02-15
qwen/qwen3.5-35b-a3b
moonshotai/kimi-k2.5
x-ai/grok-4.20-beta
```

**Key scoping:** `OPENROUTER_API_KEY` is only sent to `openrouter.ai` endpoints. It is never forwarded to custom or other provider endpoints.

---

### Nous Portal

Nous Research's own inference endpoint with subscription-based access.

**Setup:** Run `hermes model` and select Nous Portal. This triggers an OAuth device code flow.

**Credential flow:**

1. Hermes requests a device code from `https://portal.nousresearch.com`.
2. You open a URL and enter a code.
3. A Portal access token is stored in `~/.hermes/auth.json`.
4. At runtime, Hermes exchanges the Portal token for short-lived agent API keys.
5. Keys are automatically re-minted before expiry (`HERMES_NOUS_MIN_KEY_TTL_SECONDS`, default 1800 seconds / 30 minutes).

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `HERMES_PORTAL_BASE_URL` | Override Nous Portal URL (development/testing) |
| `NOUS_INFERENCE_BASE_URL` | Override Nous inference API URL |
| `HERMES_NOUS_MIN_KEY_TTL_SECONDS` | Minimum agent key TTL before re-mint. Default: `1800`. |
| `HERMES_NOUS_TIMEOUT_SECONDS` | HTTP timeout for Nous credential/token flows. |

**Static model catalog:**

```
claude-opus-4-6
claude-sonnet-4-6
gpt-5.4
gemini-3-flash
gemini-3.0-pro-preview
deepseek-v3.2
```

---

### OpenAI Codex

ChatGPT OAuth-backed provider for Codex models. No separate OpenAI API key required.

**Setup:** Run `hermes model` and select OpenAI Codex. This triggers the OAuth device code flow against `https://auth.openai.com/oauth/token` using client ID `app_EMoamEEZ73f0CkXaXp7hrann`.

**API mode:** `codex_responses` (separate Responses API path, not Chat Completions).

Hermes stores credentials in `~/.hermes/auth.json` and can import existing Codex CLI credentials from `~/.codex/auth.json` when present. No Codex CLI installation is required.

**Static model catalog:**

```
gpt-5.3-codex
gpt-5.2-codex
gpt-5.1-codex-mini
gpt-5.1-codex-max
```

---

### GitHub Copilot

First-class provider using your GitHub Copilot subscription. Provides access to GPT-5.x, Claude, Gemini, and other models.

**Setup:** Run `hermes model` and select GitHub Copilot, or set a token environment variable directly.

**Credential resolution order** (implemented in `hermes_cli/copilot_auth.py`):

1. `COPILOT_GITHUB_TOKEN` environment variable
2. `GH_TOKEN` environment variable
3. `GITHUB_TOKEN` environment variable
4. `gh auth token` CLI fallback

**Supported token types:**

| Prefix | Type | How to obtain |
|--------|------|---------------|
| `gho_` | OAuth token | `hermes model` → GitHub Copilot → Login with GitHub |
| `github_pat_` | Fine-grained PAT | GitHub Settings → Developer settings → Fine-grained tokens (needs Copilot Requests permission) |
| `ghu_` | GitHub App token | Via GitHub App installation |

Classic PATs (`ghp_*`) are **not supported** by the Copilot API and are explicitly rejected.

**OAuth device code flow:** When no environment variable provides a usable token, `hermes model` initiates an OAuth device code flow using client ID `Ov23li8tweQw6odWQebz` against `https://github.com/login/device/code`. The flow polls `https://github.com/login/oauth/access_token` at 5-second intervals until a `gho_*` token is returned or a 300-second timeout expires.

**API mode routing:** Determined by `copilot_runtime_api_mode()` in `hermes_cli/runtime_provider.py`. GPT-5+ models (except `gpt-5-mini`) use the Responses API (`codex_responses`). All other models use Chat Completions.

**Static model catalog:**

```
gpt-5.4
gpt-5.4-mini
gpt-5-mini
gpt-5.3-codex
gpt-5.2-codex
gpt-4.1
gpt-4o
gpt-4o-mini
claude-opus-4.6
claude-sonnet-4.6
claude-sonnet-4.5
claude-haiku-4.5
gemini-2.5-pro
grok-code-fast-1
```

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `COPILOT_GITHUB_TOKEN` | GitHub token for Copilot API (first priority) |
| `GH_TOKEN` | GitHub token (second priority) |
| `GITHUB_TOKEN` | GitHub token (third priority) |

**Permanent config:**

```yaml
model:
  provider: "copilot"
  default: "gpt-5.4"
```

---

### GitHub Copilot ACP

Spawns the local Copilot CLI as a subprocess. Requires the GitHub Copilot CLI installed in `$PATH` and an existing `copilot login` session.

```bash
hermes chat --provider copilot-acp --model copilot-acp
```

**Environment variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `HERMES_COPILOT_ACP_COMMAND` | `copilot` | Override the Copilot CLI binary path |
| `COPILOT_CLI_PATH` | — | Alias for `HERMES_COPILOT_ACP_COMMAND` |
| `HERMES_COPILOT_ACP_ARGS` | `--acp --stdio` | Override ACP arguments |
| `COPILOT_ACP_BASE_URL` | `acp://copilot` | Override ACP base URL |

---

### Anthropic (Native)

Native Anthropic Messages API — no OpenRouter proxy needed. Supports native prompt caching.

**API mode:** `anthropic_messages` (uses `agent/anthropic_adapter.py` for translation).

**Three auth methods:**

**1. API key (pay-per-token):**

```bash
export ANTHROPIC_API_KEY=sk-ant-...
hermes chat --provider anthropic --model claude-sonnet-4-6
```

**2. Claude Code credential auto-discovery (preferred):**

Hermes reads Claude Code's own credential files at `~/.claude/.credentials.json`. When valid refreshable credentials exist, Hermes uses them directly, keeping them refreshable instead of copying a static token. The startup gate in `hermes_cli/main.py` explicitly checks for Claude Code credentials via `agent.anthropic_adapter.read_claude_code_credentials()` and `is_claude_code_token_valid()`.

```bash
# Auto-detect Claude Code credentials — no env var needed
hermes chat --provider anthropic
```

**3. Manual token override:**

```bash
export ANTHROPIC_TOKEN=your-setup-token
hermes chat --provider anthropic
```

**Credential resolution order** (inside `agent/anthropic_adapter.py`):

1. Claude Code credential files (`~/.claude/.credentials.json`) — refreshable, preferred.
2. `ANTHROPIC_TOKEN` environment variable — manual or legacy OAuth token.
3. `ANTHROPIC_API_KEY` environment variable — console API key.
4. `CLAUDE_CODE_OAUTH_TOKEN` environment variable — explicit override.

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Anthropic Console API key |
| `ANTHROPIC_TOKEN` | Manual or legacy Anthropic OAuth/setup-token override |
| `CLAUDE_CODE_OAUTH_TOKEN` | Explicit Claude Code token override |

**Runtime behavior:**
- Hermes preflights Anthropic credential refresh before native Messages API calls.
- Hermes retries once on a `401` after rebuilding the Anthropic client as a fallback path.
- Base URL can be overridden via `model.base_url` in `config.yaml`.

**Aliases:** `--provider claude` and `--provider claude-code` both map to `--provider anthropic`.

**Static model catalog:**

```
claude-opus-4-6
claude-sonnet-4-6
claude-opus-4-5-20251101
claude-sonnet-4-5-20250929
claude-opus-4-20250514
claude-sonnet-4-20250514
claude-haiku-4-5-20251001
```

**Permanent config:**

```yaml
model:
  provider: "anthropic"
  default: "claude-sonnet-4-6"
```

---

### Vercel AI Gateway

```bash
hermes chat --provider ai-gateway --model <model>
```

**Setup:**

```bash
hermes config set AI_GATEWAY_API_KEY your-key
```

Hermes fetches available models from the gateway's `/models` endpoint, filtering to language models with tool-use support.

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `AI_GATEWAY_API_KEY` | Vercel AI Gateway API key (required) |
| `AI_GATEWAY_BASE_URL` | Override AI Gateway base URL. Default: `https://ai-gateway.vercel.sh/v1`. |

---

### Z.AI / ZhipuAI GLM

```bash
hermes chat --provider zai --model glm-5
```

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `GLM_API_KEY` | Z.AI / ZhipuAI API key (primary) |
| `ZAI_API_KEY` | Alias for `GLM_API_KEY` |
| `Z_AI_API_KEY` | Alias for `GLM_API_KEY` |
| `GLM_BASE_URL` | Override Z.AI base URL. Default: `https://api.z.ai/api/paas/v4`. |

**Static model catalog:** `glm-5`, `glm-4.7`, `glm-4.5`, `glm-4.5-flash`

---

### Kimi / Moonshot AI

```bash
hermes chat --provider kimi-coding --model kimi-k2.5
```

**Endpoint auto-detection:** Keys prefixed `sk-kimi-` automatically route to `https://api.kimi.com/coding/v1`. Legacy keys from `platform.moonshot.ai` use the default `https://api.moonshot.ai/v1`. Override with `KIMI_BASE_URL`.

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `KIMI_API_KEY` | Kimi / Moonshot AI API key (required) |
| `KIMI_BASE_URL` | Override Kimi base URL |

**Static model catalog:** `kimi-for-coding`, `kimi-k2.5`, `kimi-k2-thinking`, `kimi-k2-thinking-turbo`, `kimi-k2-turbo-preview`, `kimi-k2-0905-preview`

---

### MiniMax

**Global endpoint:**

```bash
hermes chat --provider minimax --model MiniMax-M2.7
```

| Variable | Description |
|----------|-------------|
| `MINIMAX_API_KEY` | MiniMax API key — global endpoint |
| `MINIMAX_BASE_URL` | Override. Default: `https://api.minimax.io/v1`. |

**China endpoint:**

```bash
hermes chat --provider minimax-cn --model MiniMax-M2.7
```

| Variable | Description |
|----------|-------------|
| `MINIMAX_CN_API_KEY` | MiniMax API key — China endpoint |
| `MINIMAX_CN_BASE_URL` | Override. Default: `https://api.minimaxi.com/v1`. |

**Static model catalog:** `MiniMax-M2.7`, `MiniMax-M2.7-highspeed`, `MiniMax-M2.5`, `MiniMax-M2.5-highspeed`, `MiniMax-M2.1`

---

### Alibaba Cloud (DashScope)

```bash
hermes chat --provider alibaba --model qwen-plus
```

**API mode:** `anthropic_messages` (routes through Anthropic-compatible DashScope endpoint).

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `DASHSCOPE_API_KEY` | Alibaba Cloud DashScope API key (required) |
| `DASHSCOPE_BASE_URL` | Override DashScope base URL |

**Aliases:** `dashscope`, `qwen` also accepted in some contexts.

---

### DeepSeek

```bash
hermes chat --provider deepseek --model deepseek-chat
```

| Variable | Description |
|----------|-------------|
| `DEEPSEEK_API_KEY` | DeepSeek API key |
| `DEEPSEEK_BASE_URL` | Custom base URL. Default: `https://api.deepseek.com/v1`. |

**Static model catalog:** `deepseek-chat`, `deepseek-reasoner`

---

### Kilo Code

```bash
hermes chat --provider kilocode --model <model>
```

| Variable | Description |
|----------|-------------|
| `KILOCODE_API_KEY` | Kilo Code API key |
| `KILOCODE_BASE_URL` | Override. Default: `https://api.kilo.ai/api/gateway`. |

---

### OpenCode Zen and OpenCode Go

**OpenCode Zen** (pay-as-you-go, curated models):

```bash
hermes chat --provider opencode-zen --model gpt-5.4
```

| Variable | Description |
|----------|-------------|
| `OPENCODE_ZEN_API_KEY` | OpenCode Zen API key |
| `OPENCODE_ZEN_BASE_URL` | Override base URL |

**OpenCode Go** ($10/month subscription for open models):

```bash
hermes chat --provider opencode-go --model <model>
```

| Variable | Description |
|----------|-------------|
| `OPENCODE_GO_API_KEY` | OpenCode Go API key |
| `OPENCODE_GO_BASE_URL` | Override base URL |

---

### Custom / Self-Hosted Endpoints

Any server implementing `/v1/chat/completions` works.

**Interactive setup (recommended):**

```bash
hermes model
# Select "Custom endpoint (self-hosted / VLLM / etc.)"
# Enter: API base URL, API key, Model name
```

**Manual `.env` setup:**

```bash
OPENAI_BASE_URL=http://localhost:8000/v1
OPENAI_API_KEY=your-key
LLM_MODEL=your-model-name
```

When saved via `hermes model`, the endpoint is persisted to `config.yaml` under `model.base_url` and `model.provider: custom`. It continues to work even when `OPENAI_BASE_URL` is not exported in the current shell.

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | API key for custom OpenAI-compatible endpoints |
| `OPENAI_BASE_URL` | Base URL for custom endpoint (vLLM, SGLang, Ollama, etc.) |

**Key scoping:** `OPENAI_API_KEY` is used for custom endpoints and as a fallback. It is not forwarded to OpenRouter endpoints (which use `OPENROUTER_API_KEY`) or to AI Gateway (which uses `AI_GATEWAY_API_KEY`).

**Named custom providers in `config.yaml`:**

```yaml
custom_providers:
  - name: my-local
    base_url: http://localhost:8000/v1
    api_key: my-key
    api_mode: chat_completions   # optional
```

Select with `--provider my-local` or `--provider custom:my-local`.

**Common OpenAI-compatible servers:**

| Server | URL | Best for |
|--------|-----|----------|
| Ollama | `http://localhost:11434/v1` | Local models, zero GPU required |
| vLLM | `http://localhost:8000/v1` | High-throughput GPU serving |
| SGLang | `http://localhost:8000/v1` | Multi-turn with prefix caching |
| llama.cpp server | `http://localhost:8080/v1` | CPU and Apple Silicon inference |
| LiteLLM | `http://localhost:4000/v1` | Multi-provider proxy |
| Together AI | `https://api.together.xyz/v1` | Cloud open models |
| Groq | `https://api.groq.com/openai/v1` | Ultra-fast inference |
| Fireworks AI | `https://api.fireworks.ai/inference/v1` | Fast open model hosting |
| Cerebras | `https://api.cerebras.ai/v1` | Wafer-scale chip inference |
| Mistral AI | `https://api.mistral.ai/v1` | Mistral models |
| OpenAI | `https://api.openai.com/v1` | Direct OpenAI access |

---

## Credential Resolution

The shared resolution path is implemented in `hermes_cli/runtime_provider.py` (`resolve_runtime_provider()`). This same function is called by the CLI, gateway, cron jobs, ACP sessions, and auxiliary model calls.

### Resolution Precedence

1. Explicit `--provider` CLI argument or `requested` parameter.
2. `model.provider` in `~/.hermes/config.yaml`.
3. `HERMES_INFERENCE_PROVIDER` environment variable.
4. Auto-detection (`auto`).

This ordering prevents a stale shell export from silently overriding the provider a user last selected in `hermes model`.

### Output of Runtime Resolution

`resolve_runtime_provider()` returns a dict containing:

| Key | Description |
|-----|-------------|
| `provider` | Canonical provider ID |
| `api_mode` | `chat_completions`, `codex_responses`, or `anthropic_messages` |
| `base_url` | API endpoint base URL |
| `api_key` | Resolved API key or token |
| `source` | Where the credentials came from (`env`, `portal`, `auth-store`, `explicit`) |
| `requested_provider` | The provider ID that was originally requested |
| Provider-specific | `expires_at`, `last_refresh`, `command`, `args` for relevant providers |

### API Modes

| Mode | Used by |
|------|---------|
| `chat_completions` | OpenRouter, Nous, custom endpoints, most API-key providers |
| `codex_responses` | OpenAI Codex (`openai-codex`), GPT-5+ models on Copilot |
| `anthropic_messages` | Anthropic native, Alibaba Cloud DashScope |

### Key Scoping (Anti-Leakage)

Hermes enforces strict API key scoping to prevent credentials from leaking to unrelated endpoints:

- `OPENROUTER_API_KEY` is only sent to `openrouter.ai` endpoints.
- `AI_GATEWAY_API_KEY` is only sent to `ai-gateway.vercel.sh` endpoints.
- `OPENAI_API_KEY` is used for custom endpoints and as a fallback.
- `OPENROUTER_API_KEY` is not reused for auxiliary task custom endpoints.

When the resolved `base_url` targets `openrouter.ai`, the key priority is `OPENROUTER_API_KEY` → `OPENAI_API_KEY`. For all other endpoints, the priority is `OPENAI_API_KEY` → `OPENROUTER_API_KEY`.

---

## Provider Routing (OpenRouter)

When using OpenRouter, you can control which underlying sub-providers handle your requests. Add a `provider_routing` section to `~/.hermes/config.yaml`:

```yaml
provider_routing:
  sort: "price"              # "price" (default), "throughput", or "latency"
  only: []                   # Whitelist: only use these providers
  ignore: []                 # Blacklist: never use these providers
  order: []                  # Explicit priority order
  require_parameters: false  # Only use providers supporting all request params
  data_collection: null      # "allow" or "deny"
```

This applies to both CLI and gateway modes. The config is passed to `AIAgent` as separate parameters that are forwarded to OpenRouter via the `extra_body.provider` field.

Provider routing only applies when using OpenRouter. It has no effect with direct provider connections.

### Options

**`sort`** — How OpenRouter ranks available providers:

| Value | Description |
|-------|-------------|
| `"price"` | Cheapest provider first (default) |
| `"throughput"` | Fastest tokens-per-second first |
| `"latency"` | Lowest time-to-first-token first |

**`only`** — Whitelist of provider names. Only these providers will be used:

```yaml
provider_routing:
  only: ["Anthropic", "Google"]
```

**`ignore`** — Blacklist of provider names. These providers will never be used:

```yaml
provider_routing:
  ignore: ["Together", "DeepInfra"]
```

**`order`** — Explicit priority order. Unlisted providers are used as fallbacks:

```yaml
provider_routing:
  order: ["Anthropic", "Google", "AWS Bedrock"]
```

**`require_parameters`** — When `true`, only routes to providers that support all request parameters (like `temperature`, `top_p`, `tools`). Prevents silent parameter drops.

**`data_collection`** — `"allow"` or `"deny"`. `"deny"` excludes providers that may store or train on your data.

### Shortcuts

Append to any model name:

- `:nitro` — throughput sorting (e.g., `anthropic/claude-sonnet-4:nitro`)
- `:floor` — price sorting (e.g., `anthropic/claude-sonnet-4:floor`)

### Examples

```yaml
# Cost optimization
provider_routing:
  sort: "price"
  ignore: ["Together"]
  require_parameters: true
  data_collection: "deny"

# Lock to Anthropic
provider_routing:
  only: ["Anthropic"]

# Preferred order with fallbacks
provider_routing:
  order: ["Anthropic", "Google"]
  require_parameters: true
```

---

## Smart Model Routing

Smart model routing sends short, simple turns to a cheap model while routing complex turns to your primary model.

**Configuration in `config.yaml`:**

```yaml
smart_model_routing:
  enabled: true
  max_simple_chars: 160
  max_simple_words: 28
  cheap_model:
    provider: openrouter
    model: google/gemini-2.5-flash
    # base_url: http://localhost:8000/v1   # optional custom endpoint
    # api_key_env: MY_CUSTOM_KEY           # optional env var for that endpoint
```

**When it activates:**

- The turn is short (within `max_simple_chars` and `max_simple_words`).
- The turn is single-line.
- The turn does not look code/tool/debug-heavy.

**When it stays on the primary model:**

- Coding or debugging work.
- Tool-heavy requests.
- Long or multi-line analysis.

**Fallback:** If the cheap route cannot be resolved cleanly, Hermes falls back to the primary model automatically.

This is intentionally conservative. Use it when you want lower latency or cost without fully changing your default model.

---

## Fallback Providers

Hermes has two independent fallback systems:

### Primary Model Fallback

Automatically switches to a backup provider:model when the main model fails with rate limits, server errors, auth failures, or invalid responses.

**Configuration in `config.yaml`:**

```yaml
fallback_model:
  provider: openrouter       # required
  model: anthropic/claude-sonnet-4  # required
  # base_url: http://localhost:8000/v1   # optional, for custom endpoints
  # api_key_env: MY_CUSTOM_KEY           # optional, env var name for the API key
```

Both `provider` and `model` are required. If either is missing, fallback is disabled.

**Trigger conditions:**

| Trigger | Details |
|---------|---------|
| Rate limits (`429`) | After exhausting retry attempts |
| Server errors (`500`, `502`, `503`) | After exhausting retry attempts |
| Auth failures (`401`, `403`) | Immediately — no retries |
| Not found (`404`) | Immediately |
| Invalid responses | When the API returns malformed or empty responses repeatedly |

**Activation flow** (from `hermes_cli/runtime_provider.py` via `AIAgent._try_activate_fallback()`):

1. Returns `False` immediately if already activated or not configured.
2. Calls `resolve_provider_client()` from `agent/auxiliary_client.py` to build a new client.
3. Determines `api_mode`: `codex_responses` for `openai-codex`, `anthropic_messages` for `anthropic`, `chat_completions` for everything else.
4. Swaps in-place: `self.model`, `self.provider`, `self.base_url`, `self.api_mode`, `self.client`.
5. For Anthropic fallback: builds a native Anthropic client.
6. Re-evaluates prompt caching (enabled for Claude models on OpenRouter).
7. Sets `_fallback_activated = True` — fires **at most once** per session.
8. Resets retry count to 0 and continues the agent loop.

**Supported fallback providers:**

| Provider value | Requirements |
|---------------|-------------|
| `openrouter` | `OPENROUTER_API_KEY` |
| `nous` | `hermes login` (OAuth) |
| `openai-codex` | `hermes model` (ChatGPT OAuth) |
| `copilot` | GitHub token |
| `anthropic` | `ANTHROPIC_API_KEY` or Claude Code credentials |
| `zai` | `GLM_API_KEY` |
| `kimi-coding` | `KIMI_API_KEY` |
| `minimax` | `MINIMAX_API_KEY` |
| `minimax-cn` | `MINIMAX_CN_API_KEY` |
| `kilocode` | `KILOCODE_API_KEY` |
| `ai-gateway` | `AI_GATEWAY_API_KEY` |
| `custom` | `base_url` + optional `api_key_env` |

**Where fallback works:**

| Context | Supported |
|---------|-----------|
| CLI sessions | Yes |
| Messaging gateway | Yes |
| Subagent delegation | No — subagents do not inherit fallback config |
| Cron jobs | No — run with a fixed provider |
| Auxiliary tasks | No — use their own provider chain |

Fallback is configured exclusively through `config.yaml`. There are no environment variables for it.

**Examples:**

```yaml
# OpenRouter fallback for Anthropic native
model:
  provider: anthropic
  default: claude-sonnet-4-6
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4

# Local model as fallback for cloud
fallback_model:
  provider: custom
  model: llama-3.1-70b
  base_url: http://localhost:8000/v1
  api_key_env: LOCAL_API_KEY
```

### Auxiliary Task Fallback

Auxiliary tasks (vision, web extraction, compression, memory flush, etc.) each have their own independent provider resolution chain.

**Auto-detection chain for text tasks:**

```
OpenRouter → Nous Portal → Custom endpoint → Codex OAuth →
API-key providers (z.ai, Kimi, MiniMax, Anthropic) → give up
```

**Auto-detection chain for vision tasks:**

```
Main provider (if vision-capable) → OpenRouter → Nous Portal →
Codex OAuth → Anthropic → Custom endpoint → give up
```

If the resolved provider fails at call time, Hermes tries OpenRouter as a last-resort fallback (unless the provider is already OpenRouter or an explicit `base_url` is set).

**Auxiliary tasks with independent provider resolution:**

| Task | Config key |
|------|-----------|
| Vision (image analysis, browser screenshots) | `auxiliary.vision` |
| Web extraction (page summarization) | `auxiliary.web_extract` |
| Context compression | `auxiliary.compression` or `compression.summary_provider` |
| Session search | `auxiliary.session_search` |
| Skills hub | `auxiliary.skills_hub` |
| MCP helpers | `auxiliary.mcp` |
| Memory flush | `auxiliary.flush_memories` |

**Configuration:**

```yaml
auxiliary:
  vision:
    provider: "auto"              # auto | openrouter | nous | codex | main | anthropic
    model: ""                     # e.g. "openai/gpt-4o"
    base_url: ""                  # direct endpoint (takes precedence over provider)
    api_key: ""                   # API key for base_url
  web_extract:
    provider: "auto"
    model: ""
  compression:
    provider: "auto"
    model: ""
```

When `base_url` is set, it takes precedence over `provider`. Authentication uses the configured `api_key`, falling back to `OPENAI_API_KEY`. `OPENROUTER_API_KEY` is never reused for custom endpoints.

**Provider options for auxiliary tasks:**

| Value | Description | Requirements |
|-------|-------------|-------------|
| `"auto"` | Try providers in order (default) | At least one provider configured |
| `"openrouter"` | Force OpenRouter | `OPENROUTER_API_KEY` |
| `"nous"` | Force Nous Portal | `hermes login` |
| `"codex"` | Force Codex OAuth | `hermes model` → Codex |
| `"main"` | Use the main agent's provider | Active main provider |
| `"anthropic"` | Force Anthropic native | `ANTHROPIC_API_KEY` or Claude Code credentials |

When `provider: "main"` is set, the auxiliary task routes through the same custom endpoint as the main agent — whether that comes from `OPENAI_BASE_URL`, a saved `hermes model` endpoint, or `config.yaml`.

---

## Adding a Custom Provider

Hermes can already talk to any OpenAI-compatible endpoint through the custom provider path (`OPENAI_BASE_URL` + `OPENAI_API_KEY`). A built-in first-class provider is only needed when you want:

- Provider-specific auth or token refresh.
- A curated model catalog shown in menus.
- Setup / `hermes model` menu entries.
- Provider aliases for `provider:model` syntax.
- A non-OpenAI API shape requiring a new adapter.

### Implementation Layers

A built-in provider must be consistent across these files (from `website/docs/developer-guide/adding-providers.md`):

| File | Purpose |
|------|---------|
| `hermes_cli/auth.py` | `ProviderConfig` entry in `PROVIDER_REGISTRY`, aliases in `_PROVIDER_ALIASES` |
| `hermes_cli/models.py` | Model catalog in `_PROVIDER_MODELS`, display labels in `_PROVIDER_LABELS`, aliases in `_PROVIDER_ALIASES` |
| `hermes_cli/runtime_provider.py` | Runtime resolution branch returning `provider`, `api_mode`, `base_url`, `api_key`, `source` |
| `hermes_cli/main.py` | `provider_labels`, dispatch inside the `model` command, `--provider` argument choices |
| `hermes_cli/setup.py` | `provider_choices`, auth branch, model-selection branch |
| `agent/auxiliary_client.py` | Default aux model in `_API_KEY_PROVIDER_AUX_MODELS` |
| `agent/model_metadata.py` | Context lengths for the provider's models |

For a native provider with a non-OpenAI protocol (like `anthropic_messages` or `codex_responses`):

- Add `agent/<provider>_adapter.py` — handles client construction, message conversion, response normalization, usage extraction.
- Update `run_agent.py` — add the new `api_mode` to every switch point: `__init__`, `_build_api_kwargs()`, `_api_call_with_interrupt()`, interrupt/rebuild paths, response validation, finish-reason extraction, token-usage extraction, fallback activation.

### Two Implementation Paths

**Path A — OpenAI-compatible:** The provider accepts standard chat-completions requests. You do not need a new adapter or `api_mode`. Add auth metadata, model catalog, runtime resolution, and CLI wiring.

**Path B — Native:** The provider uses a different protocol. Add everything from Path A plus a provider adapter and `run_agent.py` branches.

### Step-by-Step Summary

1. **Pick a canonical provider ID** — e.g., `kimi-coding`. Use it everywhere: `PROVIDER_REGISTRY`, `_PROVIDER_LABELS`, `_PROVIDER_ALIASES`, `--provider` choices, tests.

2. **Add `ProviderConfig` in `hermes_cli/auth.py`:**

   ```python
   "my-provider": ProviderConfig(
       id="my-provider",
       name="My Provider",
       auth_type="api_key",
       inference_base_url="https://api.myprovider.com/v1",
       api_key_env_vars=("MY_PROVIDER_API_KEY",),
       base_url_env_var="MY_PROVIDER_BASE_URL",
   ),
   ```

3. **Add model catalog in `hermes_cli/models.py`:** Update `_PROVIDER_MODELS`, `_PROVIDER_LABELS`, `_PROVIDER_ALIASES`, and `list_available_providers()`. If the provider exposes a live `/models` endpoint, prefer that over a static list.

4. **Add runtime resolution in `hermes_cli/runtime_provider.py`:** Return at minimum `{"provider", "api_mode", "base_url", "api_key", "source", "requested_provider"}`. For OpenAI-compatible providers, `api_mode` is `chat_completions`.

5. **Wire the CLI in `hermes_cli/main.py` and `hermes_cli/setup.py`:** Both must know about the provider — only updating one causes `hermes model` and `hermes setup` to drift.

6. **Add auxiliary defaults in `agent/auxiliary_client.py`:** Add the provider's cheap/fast aux model to `_API_KEY_PROVIDER_AUX_MODELS`.

7. **Add context lengths in `agent/model_metadata.py`:** Required for token budgeting and compression thresholds.

8. **For native providers:** Add `agent/<provider>_adapter.py` and update all `api_mode` switch points in `run_agent.py`. Search for `api_mode` and `self.client.` — every path that assumes the standard OpenAI client can break.

9. **Add tests:** Cover auth resolution, CLI menu/provider selection, runtime provider resolution, agent execution path, `provider:model` parsing, and any adapter-specific message conversion.

10. **Update user docs:** `website/docs/user-guide/configuration.md` and `website/docs/reference/environment-variables.md`.

### Common Pitfalls

- Adding the provider to `auth.py` but not `models.py` — credentials resolve but `/model` and `provider:model` inputs fail.
- Forgetting that `config["model"]` can be a string or a dict — normalize both forms.
- Not updating `hermes setup` when you update `hermes model` — both flows must stay in sync.
- Missing auxiliary paths — the main chat path works while vision helpers or compression fail silently.
- Sending OpenRouter-only fields (like provider routing) to other providers.
- Native-provider branches hiding in `run_agent.py` — search exhaustively for `api_mode` and `self.client.`.
