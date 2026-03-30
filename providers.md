# Providers

This document covers every inference provider supported by Hermes Agent, how credential resolution works, provider routing, smart model routing, fallback providers, and how to add a custom provider.

---

## Supported Providers

All providers are registered in `PROVIDER_REGISTRY` in `hermes_cli/auth.py`. The canonical provider ID is used in `--provider` flags, `config.yaml`, and environment variables.

| Provider ID | Name | Auth Type | Base URL |
|-------------|------|-----------|----------|
| `openrouter` | OpenRouter | API key | `https://openrouter.ai/api/v1` |
| `nous` | Nous Portal | OAuth device code | `https://portal.nousresearch.com` -> `https://inference-api.nousresearch.com/v1` |
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
| `huggingface` | Hugging Face Inference API | API key | `https://api-inference.huggingface.co/v1` |
| custom | Any OpenAI-compatible endpoint | API key | User-configured |

### Provider Aliases

The following aliases are recognized in `--provider`, `/model provider:model`, and `config.yaml`:

| Alias | Maps to |
|-------|---------|
| `glm`, `z-ai`, `z.ai`, `zhipu` | `zai` |
| `github`, `github-copilot`, `github-models`, `github-model` | `copilot` |
| `github-copilot-acp`, `copilot-acp-agent` | `copilot-acp` |
| `kimi`, `moonshot` | `kimi-coding` |
| `minimax-china`, `minimax_cn` | `minimax-cn` |
| `claude`, `claude-code` | `anthropic` |
| `deep-seek` | `deepseek` |
| `opencode`, `zen` | `opencode-zen` |
| `go`, `opencode-go-sub` | `opencode-go` |
| `aigateway`, `vercel`, `vercel-ai-gateway` | `ai-gateway` |
| `kilo`, `kilo-code`, `kilo-gateway` | `kilocode` |
| `dashscope`, `aliyun`, `qwen`, `alibaba-cloud` | `alibaba` |
| `hf`, `hf-inference`, `hugging-face` | `huggingface` |

---

## Provider Detail

### OpenRouter

Recommended for flexibility -- routes to hundreds of models through a single API key.

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
stepfun/step-3.5-flash
minimax/minimax-m2.5
z-ai/glm-5
z-ai/glm-5-turbo
moonshotai/kimi-k2.5
x-ai/grok-4.20-beta
nvidia/nemotron-3-super-120b-a12b:free  (free)
arcee-ai/trinity-large-preview:free     (free)
openai/gpt-5.4-pro
openai/gpt-5.4-nano
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
| `HERMES_NOUS_TIMEOUT_SECONDS` | HTTP timeout for Nous credential/token flows. Default: `15`. |

**Static model catalog:**

```
claude-opus-4-6
claude-sonnet-4-6
gpt-5.4
gemini-3-flash
gemini-3.0-pro-preview
deepseek-v3.2
```

The Nous Portal now supports 400+ models (v0.5.0). **v0.5.0:** Portal model slugs are aligned with OpenRouter naming conventions ([PR #3253](https://github.com/NousResearch/hermes-agent/pull/3253)).

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
gpt-5.1-codex-max
gpt-5.1-codex-mini
```

Forward-compatibility: when the live API returns `gpt-5.2-codex` or `gpt-5.3-codex`, Hermes also surfaces `gpt-5.3-codex-spark` and `gpt-5.4` as synthetic entries.

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
| `gho_` | OAuth token | `hermes model` -> GitHub Copilot -> Login with GitHub |
| `github_pat_` | Fine-grained PAT | GitHub Settings -> Developer settings -> Fine-grained tokens (needs Copilot Requests permission) |
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
| `COPILOT_CLI_PATH` | -- | Alias for `HERMES_COPILOT_ACP_COMMAND` |
| `HERMES_COPILOT_ACP_ARGS` | `--acp --stdio` | Override ACP arguments |
| `COPILOT_ACP_BASE_URL` | `acp://copilot` | Override ACP base URL |

---

### Anthropic (Native)

Native Anthropic Messages API -- no OpenRouter proxy needed. Supports native prompt caching.

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
# Auto-detect Claude Code credentials -- no env var needed
hermes chat --provider anthropic
```

**3. Manual token override:**

```bash
export ANTHROPIC_TOKEN=your-setup-token
hermes chat --provider anthropic
```

**Credential resolution order** (inside `agent/anthropic_adapter.py`):

1. Claude Code credential files (`~/.claude/.credentials.json`) -- refreshable, preferred.
2. `ANTHROPIC_TOKEN` environment variable -- manual or legacy OAuth token.
3. `ANTHROPIC_API_KEY` environment variable -- console API key.
4. `CLAUDE_CODE_OAUTH_TOKEN` environment variable -- explicit override.

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
- **v0.5.0:** Per-model native output limits replace the previously hardcoded 16K `max_tokens` cap. Opus 4.6 uses a 128K output limit; Sonnet 4.6 uses 64K ([PR #3426](https://github.com/NousResearch/hermes-agent/pull/3426), [#3444](https://github.com/NousResearch/hermes-agent/pull/3444)). This fixes "Response truncated" errors and thinking-budget exhaustion on long tasks.

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

**Static model catalog:**

```
anthropic/claude-opus-4.6
anthropic/claude-sonnet-4.6
anthropic/claude-sonnet-4.5
anthropic/claude-haiku-4.5
openai/gpt-5
openai/gpt-4.1
openai/gpt-4.1-mini
google/gemini-3-pro-preview
google/gemini-3-flash
google/gemini-2.5-pro
google/gemini-2.5-flash
deepseek/deepseek-v3.2
```

---

### Z.AI / ZhipuAI GLM

```bash
hermes chat --provider zai --model glm-5
```

**Endpoint auto-detection at setup:** Z.AI has separate billing for general vs coding plans, and global vs China endpoints. During `hermes model` setup, Hermes probes all four endpoint combinations to find one that accepts your API key:

| Endpoint ID | Base URL | Default Model | Label |
|-------------|----------|---------------|-------|
| `global` | `https://api.z.ai/api/paas/v4` | `glm-5` | Global |
| `cn` | `https://open.bigmodel.cn/api/paas/v4` | `glm-5` | China |
| `coding-global` | `https://api.z.ai/api/coding/paas/v4` | `glm-4.7` | Global (Coding Plan) |
| `coding-cn` | `https://open.bigmodel.cn/api/coding/paas/v4` | `glm-4.7` | China (Coding Plan) |

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `GLM_API_KEY` | Z.AI / ZhipuAI API key (primary) |
| `ZAI_API_KEY` | Alias for `GLM_API_KEY` |
| `Z_AI_API_KEY` | Alias for `GLM_API_KEY` |
| `GLM_BASE_URL` | Override Z.AI base URL. Default: `https://api.z.ai/api/paas/v4`. |

**Static model catalog:** `glm-5`, `glm-5-turbo` (added v0.5.0, [PR #3095](https://github.com/NousResearch/hermes-agent/pull/3095)), `glm-4.7`, `glm-4.5`, `glm-4.5-flash`

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
| `MINIMAX_API_KEY` | MiniMax API key -- global endpoint |
| `MINIMAX_BASE_URL` | Override. Default: `https://api.minimax.io/v1`. |

**China endpoint:**

```bash
hermes chat --provider minimax-cn --model MiniMax-M2.7
```

| Variable | Description |
|----------|-------------|
| `MINIMAX_CN_API_KEY` | MiniMax API key -- China endpoint |
| `MINIMAX_CN_BASE_URL` | Override. Default: `https://api.minimaxi.com/v1`. |

**Static model catalog:** `MiniMax-M2.7`, `MiniMax-M2.7-highspeed`, `MiniMax-M2.5`, `MiniMax-M2.5-highspeed`, `MiniMax-M2.1`

**v0.5.0:** The automatic `/v1` → `/anthropic` URL correction that MiniMax endpoints require can now be disabled by setting `MINIMAX_SKIP_PATH_REWRITE=true` ([PR #3553](https://github.com/NousResearch/hermes-agent/pull/3553)).

---

### Alibaba Cloud (DashScope)

```bash
hermes chat --provider alibaba --model qwen3.5-plus
```

**API mode:** `anthropic_messages` (routes through Anthropic-compatible DashScope endpoint).

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `DASHSCOPE_API_KEY` | Alibaba Cloud DashScope API key (required) |
| `DASHSCOPE_BASE_URL` | Override DashScope base URL. Default: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`. |

**Aliases:** `dashscope`, `qwen`, `aliyun`, `alibaba-cloud` also accepted.

**Static model catalog:** `qwen3.5-plus`, `qwen3-max`, `qwen3-coder-plus`, `qwen3-coder-next`, `qwen-plus-latest`, `qwen3.5-flash`, `qwen-vl-max`

**v0.5.0:** Default endpoint and model list corrected ([PR #3484](https://github.com/NousResearch/hermes-agent/pull/3484)).

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
hermes chat --provider kilocode --model anthropic/claude-opus-4.6
```

| Variable | Description |
|----------|-------------|
| `KILOCODE_API_KEY` | Kilo Code API key |
| `KILOCODE_BASE_URL` | Override. Default: `https://api.kilo.ai/api/gateway`. |

**Static model catalog:** `anthropic/claude-opus-4.6`, `anthropic/claude-sonnet-4.6`, `openai/gpt-5.4`, `google/gemini-3-pro-preview`, `google/gemini-3-flash-preview`

---

### OpenCode Zen and OpenCode Go

**OpenCode Zen** (pay-as-you-go, curated models):

```bash
hermes chat --provider opencode-zen --model gpt-5.4
```

| Variable | Description |
|----------|-------------|
| `OPENCODE_ZEN_API_KEY` | OpenCode Zen API key |
| `OPENCODE_ZEN_BASE_URL` | Override base URL. Default: `https://opencode.ai/zen/v1`. |

**OpenCode Go** ($10/month subscription for open models):

```bash
hermes chat --provider opencode-go --model glm-5
```

| Variable | Description |
|----------|-------------|
| `OPENCODE_GO_API_KEY` | OpenCode Go API key |
| `OPENCODE_GO_BASE_URL` | Override base URL. Default: `https://opencode.ai/zen/go/v1`. |

**OpenCode Go model catalog:** `glm-5`, `kimi-k2.5`, `minimax-m2.5`

---

### Hugging Face

First-class Hugging Face Inference API integration added in v0.5.0 ([PR #3419](https://github.com/NousResearch/hermes-agent/pull/3419), [#3440](https://github.com/NousResearch/hermes-agent/pull/3440)).

```bash
hermes chat --provider huggingface --model meta-llama/Llama-3.3-70B-Instruct
```

**Setup:** Run `hermes model` and select Hugging Face. The setup wizard prompts for your HF token and opens an interactive model picker.

**Model picker:** The wizard presents a curated list of agentic-capable models mapped to their OpenRouter equivalents. These are the same models used on OpenRouter but served directly from HF Inference. When a provider has 8 or more curated model entries, the wizard skips the live `/models` API probe for speed — the curated list is used directly.

**Authentication:**

```bash
hermes config set HF_TOKEN hf_...
```

Or set the standard HuggingFace environment variable (the Hermes variable takes precedence):

```bash
export HUGGING_FACE_HUB_TOKEN=hf_...
```

**Environment variables:**

| Variable | Description |
|----------|-------------|
| `HF_TOKEN` | HuggingFace API token (primary) |
| `HUGGING_FACE_HUB_TOKEN` | Standard HF Hub token (fallback) |
| `HF_BASE_URL` | Override Inference API base URL. Default: `https://api-inference.huggingface.co/v1`. |

**Permanent config:**

```yaml
model:
  provider: "huggingface"
  default: "meta-llama/Llama-3.3-70B-Instruct"
```

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

**v0.5.0:** The `custom` provider is now preserved exactly as configured — it is no longer silently remapped to `openrouter` when no explicit API key is detected ([PR #2792](https://github.com/NousResearch/hermes-agent/pull/2792)).

**v0.5.0:** Root-level `provider` and `base_url` keys in `config.yaml` are now read directly into the model config, so you can specify them at the top level without nesting under `model:` ([PR #3112](https://github.com/NousResearch/hermes-agent/pull/3112)):

```yaml
provider: custom
base_url: http://localhost:8000/v1
```

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

### Auto-Detection Order (when provider is `auto`)

1. Active OAuth provider in `~/.hermes/auth.json` with valid credentials (Nous Portal, OpenAI Codex).
2. Explicit CLI `api_key`/`base_url` -> `openrouter`.
3. `OPENAI_API_KEY` or `OPENROUTER_API_KEY` env vars -> `openrouter`.
4. Provider-specific API keys checked in `PROVIDER_REGISTRY` order (Z.AI, Kimi, MiniMax, DeepSeek, etc.) -> that provider. Note: `copilot` is intentionally skipped in auto-detection because `GITHUB_TOKEN` is commonly present for repo/tool access.
5. Fallback: `openrouter`.

### Output of Runtime Resolution

`resolve_runtime_provider()` returns a dict containing:

| Key | Description |
|-----|-------------|
| `provider` | Canonical provider ID |
| `api_mode` | `chat_completions`, `codex_responses`, or `anthropic_messages` |
| `base_url` | API endpoint base URL |
| `api_key` | Resolved API key or token |
| `source` | Where the credentials came from (`env`, `portal`, `auth-store`, `explicit`, `env/config`) |
| `requested_provider` | The provider ID that was originally requested |
| Provider-specific | `expires_at`, `last_refresh`, `command`, `args` for relevant providers |

### API Modes

| Mode | Used by |
|------|---------|
| `chat_completions` | OpenRouter, Nous, custom endpoints, most API-key providers |
| `codex_responses` | OpenAI Codex (`openai-codex`), GPT-5+ models on Copilot |
| `anthropic_messages` | Anthropic native, Alibaba Cloud DashScope, any endpoint whose URL ends in `/anthropic` |

### Key Scoping (Anti-Leakage)

Hermes enforces strict API key scoping to prevent credentials from leaking to unrelated endpoints:

- `OPENROUTER_API_KEY` is only sent to `openrouter.ai` endpoints.
- `AI_GATEWAY_API_KEY` is only sent to `ai-gateway.vercel.sh` endpoints.
- `OPENAI_API_KEY` is used for custom endpoints and as a fallback.
- `OPENROUTER_API_KEY` is not reused for auxiliary task custom endpoints.

When the resolved `base_url` targets `openrouter.ai`, the key priority is `OPENROUTER_API_KEY` -> `OPENAI_API_KEY`. For all other endpoints, the priority is `OPENAI_API_KEY` -> `OPENROUTER_API_KEY`.

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

Provider routing only applies when using OpenRouter. It has no effect with direct provider connections.

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
```

---

## Fallback Providers

### Primary Model Fallback

Automatically switches to a backup provider:model when the main model fails with rate limits (429), overload (529), server errors (503), or connection failures.

**Configuration in `config.yaml`:**

```yaml
fallback_model:
  provider: openrouter       # required
  model: anthropic/claude-sonnet-4  # required
```

Fires at most once per session. Configured exclusively through `config.yaml`.

### Auxiliary Task Fallback

Auxiliary tasks each have their own independent provider resolution chain. All tasks fall back to `openrouter:google/gemini-3-flash-preview` if the configured provider is unavailable.

---

## Adding a Custom Provider

Key files to update:

| File | Purpose |
|------|---------|
| `hermes_cli/auth.py` | `ProviderConfig` entry in `PROVIDER_REGISTRY` |
| `hermes_cli/models.py` | Model catalog in `_PROVIDER_MODELS`, labels in `_PROVIDER_LABELS`, aliases in `_PROVIDER_ALIASES` |
| `hermes_cli/runtime_provider.py` | Runtime resolution branch |
| `hermes_cli/main.py` | `--provider` argument choices, `model` command dispatch |
| `hermes_cli/setup.py` | Setup wizard provider selection |
| `agent/auxiliary_client.py` | Default auxiliary model |
| `agent/model_metadata.py` | Context lengths |
