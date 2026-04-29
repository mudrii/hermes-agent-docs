---
title: "AI Providers"
sidebar_label: "AI Providers"
sidebar_position: 1
---

# AI Providers

This page covers setting up inference providers for Hermes Agent — from cloud APIs like OpenRouter and Anthropic, to self-hosted endpoints like Ollama and vLLM, to advanced routing and fallback configurations. You need at least one provider configured to use Hermes.

## Inference Providers

You need at least one way to connect to an LLM. Use `hermes model` to switch providers and models interactively, or configure directly:

| Provider | Setup |
|----------|-------|
| **Nous Portal** | `hermes model` (OAuth, subscription-based) |
| **AWS Bedrock** | `hermes model` (native Converse API, IAM/`AWS_PROFILE`/`AWS_REGION`) |
| **OpenAI Codex** | `hermes model` (ChatGPT OAuth — runs GPT-5.5 + live model discovery) |
| **GitHub Copilot** | `hermes model` (OAuth device code flow, `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, or `gh auth token`) |
| **GitHub Copilot ACP** | `hermes model` (spawns local `copilot --acp --stdio`) |
| **Anthropic** | `hermes model` (Claude Pro/Max via Claude Code auth, Anthropic API key, or manual setup-token) |
| **OpenRouter** | `OPENROUTER_API_KEY` in `~/.hermes/.env` |
| **Vercel AI Gateway** | `AI_GATEWAY_API_KEY` in `~/.hermes/.env` (provider: `ai-gateway` — pricing + dynamic discovery) |
| **NVIDIA NIM** | `NVIDIA_API_KEY` in `~/.hermes/.env` (provider: `nvidia`; cloud or local NIM via `NVIDIA_BASE_URL`) |
| **StepFun** | `STEPFUN_API_KEY` in `~/.hermes/.env` (provider: `stepfun`) |
| **Ollama Cloud** | `OLLAMA_API_KEY` in `~/.hermes/.env` (provider: `ollama-cloud` — managed Ollama models) |
| **xAI / Grok** | `XAI_API_KEY` in `~/.hermes/.env` (provider: `xai`, alias `grok` — Responses API) |
| **Qwen Portal (OAuth)** | `hermes model` → "Qwen OAuth (Portal)" (provider: `qwen-oauth`; override with `HERMES_QWEN_BASE_URL`) |
| **z.ai / GLM** | `GLM_API_KEY` in `~/.hermes/.env` (provider: `zai`) |
| **Kimi / Moonshot** | `KIMI_API_KEY` in `~/.hermes/.env` (provider: `kimi-coding`) |
| **Kimi / Moonshot (China)** | `KIMI_CN_API_KEY` in `~/.hermes/.env` (provider: `kimi-coding-cn`; aliases: `kimi-cn`, `moonshot-cn`) |
| **Arcee AI** | `ARCEEAI_API_KEY` in `~/.hermes/.env` (provider: `arcee`; aliases: `arcee-ai`, `arceeai`) |
| **MiniMax** | `MINIMAX_API_KEY` in `~/.hermes/.env` (provider: `minimax`) |
| **MiniMax China** | `MINIMAX_CN_API_KEY` in `~/.hermes/.env` (provider: `minimax-cn`) |
| **Alibaba Cloud / DashScope** | `DASHSCOPE_API_KEY` in `~/.hermes/.env` (provider: `alibaba`, aliases: `dashscope`, `qwen`) |
| **Kilo Code** | `KILOCODE_API_KEY` in `~/.hermes/.env` (provider: `kilocode`) |
| **Xiaomi MiMo** | `XIAOMI_API_KEY` in `~/.hermes/.env` (provider: `xiaomi`, aliases: `mimo`, `xiaomi-mimo`) |
| **OpenCode Zen** | `OPENCODE_ZEN_API_KEY` in `~/.hermes/.env` (provider: `opencode-zen`) |
| **OpenCode Go** | `OPENCODE_GO_API_KEY` in `~/.hermes/.env` (provider: `opencode-go`) |
| **DeepSeek** | `DEEPSEEK_API_KEY` in `~/.hermes/.env` (provider: `deepseek`) |
| **Hugging Face** | `HF_TOKEN` in `~/.hermes/.env` (provider: `huggingface`, aliases: `hf`) |
| **Google / Gemini (API key)** | `GOOGLE_API_KEY` (or `GEMINI_API_KEY`) in `~/.hermes/.env` (provider: `gemini` — native AI Studio API) |
| **Google Gemini (OAuth)** | `hermes model` → "Google Gemini (OAuth)" (provider: `google-gemini-cli`, free tier supported, browser PKCE login) |
| **Custom Endpoint** | `hermes model` → choose "Custom endpoint" (saved in `config.yaml`) |

:::tip Model key alias
In the `model:` config section, you can use either `default:` or `model:` as the key name for your model ID. Both `model: { default: my-model }` and `model: { model: my-model }` work identically.
:::

:::info v0.11.0 transport layer
Format conversion lives in `agent/transports/` (`AnthropicTransport`, `ChatCompletionsTransport`, `ResponsesApiTransport`, `BedrockTransport`). The transport layer owns format conversion only — streaming, retries, cache, and credential refresh remain on the agent loop. See [developer-guide/provider-runtime.md](/docs/developer-guide/provider-runtime) for the full picture.
:::

:::info Codex Note
The OpenAI Codex provider authenticates via device code (open a URL, enter a code). Hermes stores credentials under `~/.hermes/auth.json` and can import existing Codex CLI credentials from `~/.codex/auth.json` when present. v0.11.0 adds **GPT-5.5 over Codex OAuth** with **live model discovery** in the picker — new OpenAI releases show up without catalog updates.
:::

:::warning
Even when using Nous Portal, Codex, or a custom endpoint, some background tasks use a separate "auxiliary" model. v0.11.0 ships **9 aux tasks** — `vision`, `compression`, `web_extract`, `session_search`, `approval`, `mcp`, `flush_memories`, `title_generation`, and `skills_hub` — by default routing through Gemini Flash via OpenRouter. An `OPENROUTER_API_KEY` enables these automatically. You can also configure each task individually — see [Auxiliary Models](/docs/user-guide/configuration#auxiliary-models). In v0.11.0, `auto` routing for auxiliary tasks defaults to the **main model** for all users.
:::

### Two Commands for Model Management

Hermes has **two** model commands that serve different purposes:

| Command | Where to run | What it does |
|---------|-------------|--------------|
| **`hermes model`** | Your terminal (outside any session) | Full setup wizard — add providers, run OAuth, enter API keys, configure endpoints |
| **`/model`** | Inside a Hermes chat session | Quick switch between **already-configured** providers and models |

If you're trying to switch to a provider you haven't set up yet, you need `hermes model`, not `/model`. Exit your session first (`Ctrl+C` or `/quit`), run `hermes model`, complete the provider setup, then start a new session.

## Nous Tool Gateway

Paid Nous Portal subscribers can also use the [Tool Gateway](tool-gateway.md) to route web search, image generation (FAL backend), text-to-speech, and browser automation through their subscription instead of separate direct-provider API keys.

This is normally enabled from `hermes model` or `hermes tools`. The image-generation gateway routes only the FAL backend — direct provider keys are still required for Recraft V4 Pro, Nano Banana Pro, GPT Image 2, xAI grok-imagine, and Codex-OAuth-backed gpt-image-2.

### AWS Bedrock (Native Converse API)

Anthropic Claude (including Claude Opus 4.7), Amazon Nova, DeepSeek v3.2, Meta Llama 4, and other models via AWS Bedrock. v0.11.0 ships native Bedrock support through the **Converse API** on top of the new transport ABC — no API key, just standard AWS auth.

```bash
# Simplest — named profile in ~/.aws/credentials
hermes chat --provider bedrock --model us.anthropic.claude-opus-4.7

# Or with explicit env vars
AWS_PROFILE=myprofile AWS_REGION=us-east-1 hermes chat --provider bedrock --model us.anthropic.claude-opus-4.7
```

Or permanently in `config.yaml`:

```yaml
model:
  provider: "bedrock"
  default: "us.anthropic.claude-opus-4.7"
bedrock:
  region: "us-east-1"          # or set AWS_REGION
  # profile: "myprofile"       # or set AWS_PROFILE
  # discovery: true            # auto-discover region from IAM
  # guardrail:                 # optional Bedrock Guardrails
  #   guardrail_identifier: "your-guardrail-id"
  #   guardrail_version: "DRAFT"
  #   stream_processing_mode: "sync"   # or "async"
  #   trace: "enabled"                  # or "disabled"
```

Authentication uses the standard boto3 chain: explicit `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`, `AWS_PROFILE` from `~/.aws/credentials`, IAM role on EC2/ECS/Lambda, IMDS, or SSO. Inference profiles use the `us.` and `global.` prefixes; cross-region inference is supported through region-aware discovery.

Set `BEDROCK_BASE_URL` only if you're calling a non-default regional endpoint. boto3's connect/read timeouts apply on top of Hermes's `request_timeout_seconds`.

See the [AWS Bedrock guide](/docs/guides/aws-bedrock) for a walkthrough of IAM setup, region selection, and cross-region inference.

### Anthropic (Native)

Use Claude models directly through the Anthropic API — no OpenRouter proxy needed. Supports three auth methods:

```bash
# With an API key (pay-per-token)
export ANTHROPIC_API_KEY=***
hermes chat --provider anthropic --model claude-opus-4.7

# Preferred: authenticate through `hermes model`
# Hermes will use Claude Code's credential store directly when available
hermes model

# Manual override with a setup-token (fallback / legacy)
export ANTHROPIC_TOKEN=***  # setup-token or manual OAuth token
hermes chat --provider anthropic
```

When you choose Anthropic OAuth through `hermes model`, Hermes prefers Claude Code's own credential store over copying the token into `~/.hermes/.env`. That keeps refreshable Claude credentials refreshable.

Or set it permanently:
```yaml
model:
  provider: "anthropic"
  default: "claude-opus-4.7"
```

:::tip Aliases
`--provider claude` and `--provider claude-code` also work as shorthand for `--provider anthropic`.
:::

### GitHub Copilot

Hermes supports GitHub Copilot as a first-class provider with two modes:

**`copilot` — Direct Copilot API** (recommended). Uses your GitHub Copilot subscription to access GPT-5.5, Claude, Gemini, and other models through the Copilot API.

```bash
hermes chat --provider copilot --model gpt-5.5
```

**Authentication options** (checked in this order):

1. `COPILOT_GITHUB_TOKEN` environment variable
2. `GH_TOKEN` environment variable
3. `GITHUB_TOKEN` environment variable
4. `gh auth token` CLI fallback

If no token is found, `hermes model` offers an **OAuth device code login** — the same flow used by the Copilot CLI and opencode.

:::warning Token types
The Copilot API does **not** support classic Personal Access Tokens (`ghp_*`). Supported token types:

| Type | Prefix | How to get |
|------|--------|------------|
| OAuth token | `gho_` | `hermes model` → GitHub Copilot → Login with GitHub |
| Fine-grained PAT | `github_pat_` | GitHub Settings → Developer settings → Fine-grained tokens (needs **Copilot Requests** permission) |
| GitHub App token | `ghu_` | Via GitHub App installation |

If your `gh auth token` returns a `ghp_*` token, use `hermes model` to authenticate via OAuth instead.
:::

**API routing**: GPT-5+ models (except `gpt-5-mini`) automatically use the Responses API. All other models (GPT-4o, Claude, Gemini, etc.) use Chat Completions. Models are auto-detected from the live Copilot catalog.

**`copilot-acp` — Copilot ACP agent backend**. Spawns the local Copilot CLI as a subprocess:

```bash
hermes chat --provider copilot-acp --model copilot-acp
# Requires the GitHub Copilot CLI in PATH and an existing `copilot login` session
```

**Permanent config:**
```yaml
model:
  provider: "copilot"
  default: "gpt-5.5"
```

| Environment variable | Description |
|---------------------|-------------|
| `COPILOT_GITHUB_TOKEN` | GitHub token for Copilot API (first priority) |
| `HERMES_COPILOT_ACP_COMMAND` | Override the Copilot CLI binary path (default: `copilot`) |
| `HERMES_COPILOT_ACP_ARGS` | Override ACP args (default: `--acp --stdio`) |

### NVIDIA NIM

Nemotron and other open source models via [build.nvidia.com](https://build.nvidia.com) (free API key) or a local NIM endpoint. Native v0.11.0 provider.

```bash
# Cloud (build.nvidia.com)
hermes chat --provider nvidia --model nvidia/nemotron-3-super-120b-a12b
# Requires: NVIDIA_API_KEY in ~/.hermes/.env

# Local NIM endpoint — override base URL
NVIDIA_BASE_URL=http://localhost:8000/v1 hermes chat --provider nvidia --model nvidia/nemotron-3-super-120b-a12b
```

Or set it permanently in `config.yaml`:
```yaml
model:
  provider: "nvidia"
  default: "nvidia/nemotron-3-super-120b-a12b"
```

:::tip Local NIM
For on-prem deployments (DGX Spark, local GPU), set `NVIDIA_BASE_URL=http://localhost:8000/v1`. NIM exposes the same OpenAI-compatible chat completions API as build.nvidia.com, so switching between cloud and local is a one-line env-var change.
:::

### Step Plan

StepFun's hosted Step models, salvaged from #6005 in v0.11.0.

```bash
hermes chat --provider step --model step-2-mini
# Requires: STEP_API_KEY in ~/.hermes/.env
```

Or in `config.yaml`:
```yaml
model:
  provider: "step"
  default: "step-2-mini"
```

### Vercel AI Gateway

[AI Gateway](https://vercel.com/docs/ai-gateway) routes through Vercel's catalog with pricing metadata, attribution, and dynamic model discovery — added in v0.11.0.

```bash
hermes chat --provider ai-gateway --model openai/gpt-5.5
# Requires: AI_GATEWAY_API_KEY in ~/.hermes/.env
```

Or in `config.yaml`:
```yaml
model:
  provider: "ai-gateway"
  default: "openai/gpt-5.5"
```

The catalog is fetched dynamically from the gateway and includes per-token pricing — so `/usage` shows true gateway-attributed cost.

### Google Gemini via OAuth (`google-gemini-cli`)

The `google-gemini-cli` provider uses Google's Cloud Code Assist backend — the same API that Google's own `gemini-cli` tool uses. This supports both the **free tier** (generous daily quota for personal accounts) and **paid tiers** (Standard/Enterprise via a GCP project).

**Quick start:**

```bash
hermes model
# → pick "Google Gemini (OAuth)"
# → see policy warning, confirm
# → browser opens to accounts.google.com, sign in
# → done — Hermes auto-provisions your free tier on first request
```

Hermes ships Google's **public** `gemini-cli` desktop OAuth client by default — the same credentials Google includes in their open-source `gemini-cli`. Desktop OAuth clients are not confidential (PKCE provides the security). You do not need to install `gemini-cli` or register your own GCP OAuth client.

**Tiers & project IDs:**

| Your situation | What to do |
|---|---|
| Personal Google account, want free tier | Nothing — sign in, start chatting |
| Workspace / Standard / Enterprise account | Set `HERMES_GEMINI_PROJECT_ID` or `GOOGLE_CLOUD_PROJECT` to your GCP project ID |
| VPC-SC-protected org | Hermes detects `SECURITY_POLICY_VIOLATED` and forces `standard-tier` automatically |

Free tier auto-provisions a Google-managed project on first use. No GCP setup required.

**Quota monitoring:**

```
/gquota
```

Shows remaining Code Assist quota per model with progress bars.

:::warning Policy risk
Google considers using the Gemini CLI OAuth client with third-party software a policy violation. Some users have reported account restrictions. For the lowest-risk experience, use your own API key via the `gemini` provider instead. Hermes shows an upfront warning and requires explicit confirmation before OAuth begins.
:::

### Google Gemini (Native AI Studio API)

The `gemini` provider routes through the native AI Studio endpoint (`generativelanguage.googleapis.com`) for better performance than the OpenAI-compatible shim.

```bash
hermes chat --provider gemini --model gemini-2.5-pro
# Requires: GOOGLE_API_KEY or GEMINI_API_KEY in ~/.hermes/.env
```

### Ollama Cloud — Managed Ollama Models, OAuth + API Key

[Ollama Cloud](https://ollama.com/cloud) hosts the same open-weight catalog as local Ollama but without the GPU requirement. Pick it in `hermes model` as **Ollama Cloud**, paste your API key from [ollama.com/settings/keys](https://ollama.com/settings/keys), and Hermes auto-discovers the available models.

```bash
hermes model
# → pick "Ollama Cloud"
# → paste your OLLAMA_API_KEY
# → select from discovered models (gpt-oss:120b, glm-4.6:cloud, qwen3-coder:480b-cloud, etc.)
```

Or `config.yaml` directly:
```yaml
model:
  provider: "ollama-cloud"
  default: "gpt-oss:120b"
```

The model catalog is fetched dynamically from `ollama.com/v1/models` and cached for one hour. `model:tag` notation (e.g. `qwen3-coder:480b-cloud`) is preserved through normalization — don't use dashes.

:::tip Ollama Cloud vs local Ollama
Both speak the same OpenAI-compatible API. Cloud is a first-class provider (`--provider ollama-cloud`, `OLLAMA_API_KEY`); local Ollama is reached via the Custom Endpoint flow (base URL `http://localhost:11434/v1`, no key). Use cloud for large models you can't run locally; use local for privacy or offline work.
:::

### xAI (Grok) — Responses API + Prompt Caching

xAI is wired through the Responses API (`ResponsesApiTransport`) for automatic reasoning support on Grok 4 models — no `reasoning_effort` parameter needed, the server reasons by default. Set `XAI_API_KEY` in `~/.hermes/.env` and pick xAI in `hermes model`, or use the `grok` alias as a shortcut into `/model grok-4-1-fast-reasoning`.

When using xAI as a provider (any base URL containing `x.ai`), Hermes automatically enables prompt caching by sending the `x-grok-conv-id` header with every API request. This routes requests to the same server within a conversation session, allowing xAI's infrastructure to reuse cached system prompts and conversation history.

xAI also ships a dedicated TTS endpoint (`/v1/tts`). Select **xAI TTS** in `hermes tools` → Voice & TTS, or see the [TTS](tts.md) page for config.

### Qwen Portal (OAuth)

Alibaba's Qwen Portal with browser-based OAuth login. Pick **Qwen OAuth (Portal)** in `hermes model`, sign in through the browser, and Hermes persists the refresh token.

```bash
hermes model
# → pick "Qwen OAuth (Portal)"
# → browser opens; sign in with your Alibaba account
# → confirm — credentials are saved to ~/.hermes/auth.json

hermes chat   # uses portal.qwen.ai/v1 endpoint
```

Or configure `config.yaml`:
```yaml
model:
  provider: "qwen-oauth"
  default: "qwen3-coder-plus"
```

Set `HERMES_QWEN_BASE_URL` only if the portal endpoint relocates (default: `https://portal.qwen.ai/v1`).

:::tip Qwen OAuth vs DashScope (Alibaba)
`qwen-oauth` uses the consumer-facing Qwen Portal with OAuth login — ideal for individual users. The `alibaba` provider uses DashScope's enterprise API with a `DASHSCOPE_API_KEY` — ideal for programmatic / production workloads. Both route to Qwen-family models but live at different endpoints.
:::

### First-Class Chinese AI Providers

These providers have built-in support with dedicated provider IDs. Set the API key and use `--provider` to select:

```bash
# z.ai / ZhipuAI GLM
hermes chat --provider zai --model glm-5
# Requires: GLM_API_KEY in ~/.hermes/.env

# Kimi / Moonshot AI (international: api.moonshot.ai) — Kimi K2.5/K2.6
hermes chat --provider kimi-coding --model kimi-k2.6
# Requires: KIMI_API_KEY in ~/.hermes/.env

# Kimi / Moonshot AI (China: api.moonshot.cn)
hermes chat --provider kimi-coding-cn --model kimi-k2.6
# Requires: KIMI_CN_API_KEY in ~/.hermes/.env

# MiniMax (global endpoint)
hermes chat --provider minimax --model MiniMax-M2.7
# Requires: MINIMAX_API_KEY in ~/.hermes/.env

# MiniMax (China endpoint)
hermes chat --provider minimax-cn --model MiniMax-M2.7
# Requires: MINIMAX_CN_API_KEY in ~/.hermes/.env

# Alibaba Cloud / DashScope (Qwen models)
hermes chat --provider alibaba --model qwen3.5-plus
# Requires: DASHSCOPE_API_KEY in ~/.hermes/.env

# Xiaomi MiMo v2.5 / v2.5-pro
hermes chat --provider xiaomi --model mimo-v2.5-pro
# Requires: XIAOMI_API_KEY in ~/.hermes/.env

# Arcee AI (Trinity models)
hermes chat --provider arcee --model trinity-large-thinking
# Requires: ARCEEAI_API_KEY in ~/.hermes/.env
```

Or set the provider permanently in `config.yaml`:
```yaml
model:
  provider: "zai"       # or: kimi-coding, kimi-coding-cn, minimax, minimax-cn, alibaba, xiaomi, arcee
  default: "glm-5"
```

Base URLs can be overridden with `GLM_BASE_URL`, `KIMI_BASE_URL`, `MINIMAX_BASE_URL`, `MINIMAX_CN_BASE_URL`, `DASHSCOPE_BASE_URL`, or `XIAOMI_BASE_URL` environment variables.

:::note Z.AI Endpoint Auto-Detection
When using the Z.AI / GLM provider, Hermes automatically probes multiple endpoints (global, China, coding variants) to find one that accepts your API key. You don't need to set `GLM_BASE_URL` manually — the working endpoint is detected and cached automatically.
:::

### Hugging Face Inference Providers

[Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers) routes to 20+ open models through a unified OpenAI-compatible endpoint (`router.huggingface.co/v1`). Requests are automatically routed to the fastest available backend (Groq, Together, SambaNova, etc.) with automatic failover.

```bash
# Use any available model
hermes chat --provider huggingface --model Qwen/Qwen3-235B-A22B-Thinking-2507
# Requires: HF_TOKEN in ~/.hermes/.env

# Short alias
hermes chat --provider hf --model deepseek-ai/DeepSeek-V3.2
```

Or set it permanently in `config.yaml`:
```yaml
model:
  provider: "huggingface"
  default: "Qwen/Qwen3-235B-A22B-Thinking-2507"
```

Get your token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) — make sure to enable the "Make calls to Inference Providers" permission. Free tier included ($0.10/month credit, no markup on provider rates).

You can append routing suffixes to model names: `:fastest` (default), `:cheapest`, or `:provider_name` to force a specific backend.

The base URL can be overridden with `HF_BASE_URL`.

## Custom & Self-Hosted LLM Providers

Hermes Agent works with **any OpenAI-compatible API endpoint**. If a server implements `/v1/chat/completions`, you can point Hermes at it. This means you can use local models, GPU inference servers, multi-provider routers, or any third-party API.

### General Setup

Three ways to configure a custom endpoint:

**Interactive setup (recommended):**
```bash
hermes model
# Select "Custom endpoint (self-hosted / VLLM / etc.)"
# Enter: API base URL, API key, Model name
```

**Manual config (`config.yaml`):**
```yaml
# In ~/.hermes/config.yaml
model:
  default: your-model-name
  provider: custom
  base_url: http://localhost:8000/v1
  api_key: your-key-or-leave-empty-for-local
```

:::warning Legacy env vars
`OPENAI_BASE_URL` and `LLM_MODEL` in `.env` are **removed**. Neither is read by any part of Hermes — `config.yaml` is the single source of truth for model and endpoint configuration. If you have stale entries in your `.env`, they are automatically cleared on the next `hermes setup` or config migration. Use `hermes model` or edit `config.yaml` directly.
:::

Both approaches persist to `config.yaml`, which is the source of truth for model, provider, and base URL.

### Switching Models with `/model`

Once you have at least one custom endpoint configured, you can switch models mid-session:

```
/model custom:qwen-2.5          # Switch to a model on your custom endpoint
/model custom                    # Auto-detect the model from the endpoint
/model openrouter:claude-sonnet-4 # Switch back to a cloud provider
```

If you have **named custom providers** configured (see below), use the triple syntax:

```
/model custom:local:qwen-2.5    # Use the "local" custom provider with model qwen-2.5
/model custom:work:llama3       # Use the "work" custom provider with llama3
```

When switching providers, Hermes persists the base URL and provider to config so the change survives restarts. When switching away from a custom endpoint to a built-in provider, the stale base URL is automatically cleared.

:::tip
`/model custom` (bare, no model name) queries your endpoint's `/models` API and auto-selects the model if exactly one is loaded. Useful for local servers running a single model.
:::

Everything below follows this same pattern — just change the URL, key, and model name.

---

### Ollama — Local Models, Zero Config

[Ollama](https://ollama.com/) runs open-weight models locally with one command. Best for: quick local experimentation, privacy-sensitive work, offline use. Supports tool calling via the OpenAI-compatible API.

```bash
# Install and run a model
ollama pull qwen2.5-coder:32b
ollama serve   # Starts on port 11434
```

Then configure Hermes:

```bash
hermes model
# Select "Custom endpoint (self-hosted / VLLM / etc.)"
# Enter URL: http://localhost:11434/v1
# Skip API key (Ollama doesn't need one)
# Enter model name (e.g. qwen2.5-coder:32b)
```

Or configure `config.yaml` directly:

```yaml
model:
  default: qwen2.5-coder:32b
  provider: custom
  base_url: http://localhost:11434/v1
  context_length: 32768   # See warning below
```

:::caution Ollama defaults to very low context lengths
Ollama does **not** use your model's full context window by default. Depending on your VRAM, the default is:

| Available VRAM | Default context |
|----------------|----------------|
| Less than 24 GB | **4,096 tokens** |
| 24–48 GB | 32,768 tokens |
| 48+ GB | 256,000 tokens |

For agent use with tools, **you need at least 16k–32k context**. At 4k, the system prompt + tool schemas alone can fill the window, leaving no room for conversation.

**How to increase it** (pick one):

```bash
# Option 1: Set server-wide via environment variable (recommended)
OLLAMA_CONTEXT_LENGTH=32768 ollama serve

# Option 2: For systemd-managed Ollama
sudo systemctl edit ollama.service
# Add: Environment="OLLAMA_CONTEXT_LENGTH=32768"
# Then: sudo systemctl daemon-reload && sudo systemctl restart ollama

# Option 3: Bake it into a custom model (persistent per-model)
echo -e "FROM qwen2.5-coder:32b\nPARAMETER num_ctx 32768" > Modelfile
ollama create qwen2.5-coder-32k -f Modelfile
```

**You cannot set context length through the OpenAI-compatible API** (`/v1/chat/completions`). It must be configured server-side or via a Modelfile. This is the #1 source of confusion when integrating Ollama with tools like Hermes.
:::

**Verify your context is set correctly:**

```bash
ollama ps
# Look at the CONTEXT column — it should show your configured value
```

:::tip
List available models with `ollama list`. Pull any model from the [Ollama library](https://ollama.com/library) with `ollama pull <model>`. Ollama handles GPU offloading automatically — no configuration needed for most setups.
:::

---

### vLLM — High-Performance GPU Inference

[vLLM](https://docs.vllm.ai/) is the standard for production LLM serving. Best for: maximum throughput on GPU hardware, serving large models, continuous batching.

```bash
pip install vllm
vllm serve meta-llama/Llama-3.1-70B-Instruct \
  --port 8000 \
  --max-model-len 65536 \
  --tensor-parallel-size 2 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

Then configure Hermes:

```bash
hermes model
# Select "Custom endpoint (self-hosted / VLLM / etc.)"
# Enter URL: http://localhost:8000/v1
# Skip API key (or enter one if you configured vLLM with --api-key)
# Enter model name: meta-llama/Llama-3.1-70B-Instruct
```

**Context length:** vLLM reads the model's `max_position_embeddings` by default. If that exceeds your GPU memory, it errors and asks you to set `--max-model-len` lower. You can also use `--max-model-len auto` to automatically find the maximum that fits. Set `--gpu-memory-utilization 0.95` (default 0.9) to squeeze more context into VRAM.

**Tool calling requires explicit flags:**

| Flag | Purpose |
|------|---------|
| `--enable-auto-tool-choice` | Required for `tool_choice: "auto"` (the default in Hermes) |
| `--tool-call-parser <name>` | Parser for the model's tool call format |

Supported parsers: `hermes` (Qwen 2.5, Hermes 2/3), `llama3_json` (Llama 3.x), `mistral`, `deepseek_v3`, `deepseek_v31`, `xlam`, `pythonic`. Without these flags, tool calls won't work — the model will output tool calls as text.

:::tip
vLLM supports human-readable sizes: `--max-model-len 64k` (lowercase k = 1000, uppercase K = 1024).
:::

---

### SGLang — Fast Serving with RadixAttention

[SGLang](https://github.com/sgl-project/sglang) is an alternative to vLLM with RadixAttention for KV cache reuse. Best for: multi-turn conversations (prefix caching), constrained decoding, structured output.

```bash
pip install "sglang[all]"
python -m sglang.launch_server \
  --model meta-llama/Llama-3.1-70B-Instruct \
  --port 30000 \
  --context-length 65536 \
  --tp 2 \
  --tool-call-parser qwen
```

Then configure Hermes:

```bash
hermes model
# Select "Custom endpoint (self-hosted / VLLM / etc.)"
# Enter URL: http://localhost:30000/v1
# Enter model name: meta-llama/Llama-3.1-70B-Instruct
```

**Context length:** SGLang reads from the model's config by default. Use `--context-length` to override. If you need to exceed the model's declared maximum, set `SGLANG_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN=1`.

**Tool calling:** Use `--tool-call-parser` with the appropriate parser for your model family: `qwen` (Qwen 2.5), `llama3`, `llama4`, `deepseekv3`, `mistral`, `glm`. Without this flag, tool calls come back as plain text.

:::caution SGLang defaults to 128 max output tokens
If responses seem truncated, add `max_tokens` to your requests or set `--default-max-tokens` on the server. SGLang's default is only 128 tokens per response if not specified in the request.
:::

---

### llama.cpp / llama-server — CPU & Metal Inference

[llama.cpp](https://github.com/ggml-org/llama.cpp) runs quantized models on CPU, Apple Silicon (Metal), and consumer GPUs. Best for: running models without a datacenter GPU, Mac users, edge deployment.

```bash
# Build and start llama-server
cmake -B build && cmake --build build --config Release
./build/bin/llama-server \
  --jinja -fa \
  -c 32768 \
  -ngl 99 \
  -m models/qwen2.5-coder-32b-instruct-Q4_K_M.gguf \
  --port 8080 --host 0.0.0.0
```

Then configure Hermes to point at it. Tool calling requires `--jinja`. See `gateway.md` and the [llama.cpp function calling docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/function-calling.md) for parser details.

---

### LM Studio — Desktop App with Local Models

[LM Studio](https://lmstudio.ai/) is a desktop app for running local models with a GUI. Best for: users who prefer a visual interface, quick model testing, developers on macOS/Windows/Linux.

Start the server from the LM Studio app (Developer tab → Start Server), or use the CLI:

```bash
lms server start                        # Starts on port 1234
lms load qwen2.5-coder --context-length 32768
```

Then configure Hermes:

```bash
hermes model
# Select "Custom endpoint (self-hosted / VLLM / etc.)"
# Enter URL: http://localhost:1234/v1
# Skip API key (LM Studio doesn't require one)
# Enter model name
```

Tool calling supported since LM Studio 0.3.6. Set context length explicitly in LM Studio model settings (defaults are often 2048).

---

### Named Custom Providers

If you work with multiple custom endpoints (e.g., a local dev server and a remote GPU server), you can define them as named custom providers in `config.yaml`:

```yaml
custom_providers:
  - name: local
    base_url: http://localhost:8080/v1
    # api_key omitted — Hermes uses "no-key-required" for keyless local servers
  - name: work
    base_url: https://gpu-server.internal.corp/v1
    key_env: CORP_API_KEY
    api_mode: chat_completions   # optional, auto-detected from URL
  - name: anthropic-proxy
    base_url: https://proxy.example.com/anthropic
    key_env: ANTHROPIC_PROXY_KEY
    api_mode: anthropic_messages  # for Anthropic-compatible proxies
```

Switch between them mid-session with the triple syntax:

```
/model custom:local:qwen-2.5       # Use the "local" endpoint with qwen-2.5
/model custom:work:llama3-70b      # Use the "work" endpoint with llama3-70b
/model custom:anthropic-proxy:claude-sonnet-4  # Use the proxy
```

You can also select named custom providers from the interactive `hermes model` menu.

:::note `key_env` (formerly `api_key_env`)
v0.11.0 renamed `api_key_env` to `key_env` for custom endpoints, fallback models, and smart routing. The shorter name is consistent with `base_url`/`api_mode`. Old configs using `api_key_env` are still parsed but the canonical key is `key_env`.
:::

---

### Other Compatible Providers

Any service with an OpenAI-compatible API works. Some popular options:

| Provider | Base URL | Notes |
|----------|----------|-------|
| [Together AI](https://together.ai) | `https://api.together.xyz/v1` | Cloud-hosted open models |
| [Groq](https://groq.com) | `https://api.groq.com/openai/v1` | Ultra-fast inference |
| [DeepSeek](https://deepseek.com) | `https://api.deepseek.com/v1` | DeepSeek models |
| [Fireworks AI](https://fireworks.ai) | `https://api.fireworks.ai/inference/v1` | Fast open model hosting |
| [GMI Cloud](https://www.gmicloud.ai/) | `https://api.gmi-serving.com/v1` | Managed OpenAI-compatible inference |
| [Cerebras](https://cerebras.ai) | `https://api.cerebras.ai/v1` | Wafer-scale chip inference |
| [Mistral AI](https://mistral.ai) | `https://api.mistral.ai/v1` | OpenAI-compatible — set via Custom Endpoint |
| [OpenAI](https://openai.com) | `https://api.openai.com/v1` | Direct OpenAI access |
| [Azure OpenAI](https://azure.microsoft.com) | `https://YOUR.openai.azure.com/` | Enterprise OpenAI via Custom Endpoint |
| [LocalAI](https://localai.io) | `http://localhost:8080/v1` | Self-hosted, multi-model |
| [Jan](https://jan.ai) | `http://localhost:1337/v1` | Desktop app with local models |

Configure any of these with `hermes model` → Custom endpoint, or in `config.yaml`:

```yaml
model:
  default: meta-llama/Llama-3.1-70B-Instruct-Turbo
  provider: custom
  base_url: https://api.together.xyz/v1
  api_key: your-together-key
```

---

## Timeouts and Retries

v0.11.0 exposes per-provider and per-model timeout knobs plus an explicit retry count. Defaults are sensible for hosted providers; tune for slow self-hosted setups or aggressive failover.

```yaml
agent:
  api_max_retries: 3                # default 3; raise for flaky upstreams

model:
  provider: openrouter
  default: anthropic/claude-opus-4.7
  request_timeout_seconds: 120      # full request budget (per-model override)
  stale_timeout_seconds: 60         # kill if no streaming bytes for this long
  timeout_seconds: 600              # hard ceiling (per-model)

# Per-provider defaults (under config.yaml top level)
providers:
  bedrock:
    request_timeout_seconds: 240
  ollama-cloud:
    stale_timeout_seconds: 90
```

Use `HERMES_API_CALL_STALE_TIMEOUT` env var to override `stale_timeout_seconds` globally for a single run.

## Choosing the Right Setup

| Use Case | Recommended |
|----------|-------------|
| **Just want it to work** | OpenRouter (default) or Nous Portal |
| **Local models, easy setup** | Ollama |
| **Production GPU serving** | vLLM or SGLang |
| **Mac / no GPU** | Ollama or llama.cpp |
| **Multi-provider routing** | LiteLLM, Vercel AI Gateway, OpenRouter |
| **Cost optimization** | OpenRouter with `sort: "price"` or Vercel ai-gateway |
| **Maximum privacy** | Ollama, vLLM, or llama.cpp (fully local) |
| **Enterprise / Azure** | Azure OpenAI via Custom Endpoint |
| **AWS-native** | AWS Bedrock (native Converse API) |
| **NVIDIA stack** | NVIDIA NIM (cloud or local) |
| **Chinese AI models** | z.ai (GLM), Kimi/Moonshot, MiniMax, Xiaomi MiMo, Qwen Portal OAuth |

:::tip
You can switch between providers at any time with `hermes model` — no restart required. Your conversation history, memory, and skills carry over regardless of which provider you use.
:::

## Optional API Keys

| Feature | Provider | Env Variable |
|---------|----------|--------------|
| Web scraping | [Firecrawl](https://firecrawl.dev/) | `FIRECRAWL_API_KEY`, `FIRECRAWL_API_URL` |
| Browser automation | [Browserbase](https://browserbase.com/) | `BROWSERBASE_API_KEY`, `BROWSERBASE_PROJECT_ID` |
| Image generation | [FAL](https://fal.ai/) | `FAL_KEY` |
| Premium TTS voices | [ElevenLabs](https://elevenlabs.io/) | `ELEVENLABS_API_KEY` |
| OpenAI TTS + voice transcription | [OpenAI](https://platform.openai.com/api-keys) | `VOICE_TOOLS_OPENAI_KEY` |
| Mistral TTS + voice transcription | [Mistral](https://console.mistral.ai/) | `MISTRAL_API_KEY` |
| RL Training | [Tinker](https://tinker-console.thinkingmachines.ai/) + [WandB](https://wandb.ai/) | `TINKER_API_KEY`, `WANDB_API_KEY` |
| Cross-session user modeling | [Honcho](https://honcho.dev/) | `HONCHO_API_KEY` |
| Semantic long-term memory | [Supermemory](https://supermemory.ai) | `SUPERMEMORY_API_KEY` |

## OpenRouter Provider Routing

When using OpenRouter, you can control how requests are routed across providers. Add a `provider_routing` section to `~/.hermes/config.yaml`:

```yaml
provider_routing:
  sort: "throughput"          # "price" (default), "throughput", or "latency"
  # only: ["anthropic"]      # Only use these providers
  # ignore: ["deepinfra"]    # Skip these providers
  # order: ["anthropic", "google"]  # Try providers in this order
  # require_parameters: true  # Only use providers that support all request params
  # data_collection: "deny"   # Exclude providers that may store/train on data
```

**Shortcuts:** Append `:nitro` to any model name for throughput sorting (e.g., `anthropic/claude-sonnet-4:nitro`), or `:floor` for price sorting.

## Fallback Model

Configure a backup provider:model that Hermes switches to automatically when your primary model fails (rate limits, server errors, auth failures):

```yaml
fallback_model:
  provider: openrouter                    # required
  model: anthropic/claude-opus-4.7        # required
  # base_url: http://localhost:8000/v1    # optional, for custom endpoints
  # key_env: MY_CUSTOM_KEY                # optional, env var name for custom endpoint API key
```

In v0.11.0, fallback semantics are **per-turn** — the primary provider is restored each turn, so a transient outage doesn't permanently shift the session off your preferred model.

Supported providers: `openrouter`, `nous`, `openai-codex`, `copilot`, `copilot-acp`, `anthropic`, `gemini`, `google-gemini-cli`, `qwen-oauth`, `huggingface`, `zai`, `kimi-coding`, `kimi-coding-cn`, `minimax`, `minimax-cn`, `deepseek`, `nvidia`, `xai`, `ollama-cloud`, `bedrock`, `ai-gateway`, `stepfun`, `opencode-zen`, `opencode-go`, `kilocode`, `xiaomi`, `arcee`, `alibaba`, `custom`.

:::tip
Fallback is configured exclusively through `config.yaml` — there are no environment variables for it. For full details on when it triggers, supported providers, and how it interacts with auxiliary tasks and delegation, see [Fallback Providers](fallback-providers.md).
:::

## Smart Model Routing

Optional cheap-vs-strong routing lets Hermes keep your main model for complex work while sending very short/simple turns to a cheaper model.

```yaml
smart_model_routing:
  enabled: true
  max_simple_chars: 160
  max_simple_words: 28
  cheap_model:
    provider: openrouter
    model: google/gemini-2.5-flash
    # base_url: http://localhost:8000/v1  # optional custom endpoint
    # key_env: MY_CUSTOM_KEY              # optional env var name for that endpoint's API key
```

How it works:
- If a turn is short, single-line, and does not look code/tool/debug heavy, Hermes may route it to `cheap_model`
- If the turn looks complex, Hermes stays on your primary model/provider
- If the cheap route cannot be resolved cleanly, Hermes falls back to the primary model automatically

This is intentionally conservative. It is meant for quick, low-stakes turns like:
- short factual questions
- quick rewrites
- lightweight summaries

It will avoid routing prompts that look like:
- coding/debugging work
- tool-heavy requests
- long or multi-line analysis asks

Use this when you want lower latency or cost without fully changing your default model.

---

## See Also

- [Configuration](configuration.md) — General configuration (directory structure, config precedence, terminal backends, memory, compression, and more)
- [Environment Variables](reference/environment-variables.md) — Complete reference of all environment variables
- [Fallback Providers](fallback-providers.md) — Per-turn fallback semantics and supported providers
- [Provider Routing](provider-routing.md) — OpenRouter routing keys and shortcuts
- [Developer: Provider Runtime](developer-guide/provider-runtime.md) — Transport ABC and how providers plug in
