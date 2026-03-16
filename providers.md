# LLM Providers

Hermes Agent supports multiple LLM providers via a centralized provider router. All providers are accessed through OpenAI-compatible APIs.

---

## Supported Providers

### Nous Portal (Recommended)

The official Hermes provider. Supports OAuth device code flow — no API key needed for initial auth.

```bash
hermes setup        # Login with Nous Portal via OAuth
```

**Available models via Nous Portal:**
- `anthropic/claude-opus-4.6`
- `anthropic/claude-sonnet-4.5`
- `openai/gpt-5.4-pro`
- `openai/gpt-5.4`
- `google/gemini-3-pro-preview`
- `deepseek/deepseek-v3.2`

**Endpoint:** `https://inference-api.nousresearch.com/v1`

### OpenRouter

Access to 200+ models from a single API key.

```bash
OPENROUTER_API_KEY=sk-or-...
```

**Endpoint:** `https://openrouter.ai/api/v1`

**Notable models:**
- `anthropic/claude-opus-4.6`
- `anthropic/claude-sonnet-4.5`
- `openai/gpt-5.4-pro`, `openai/gpt-5.4`
- `google/gemini-3-pro-preview`, `google/gemini-3-flash-preview`
- `qwen/qwen3.5-plus-02-15`
- `z-ai/glm-5`
- `moonshotai/kimi-k2.5`
- `minimax/minimax-m2.5`
- `stepfun/step-3.5-flash`

### Anthropic (Native)

Direct Anthropic API access.

```bash
ANTHROPIC_API_KEY=sk-ant-...
```

**Special features when using Anthropic natively:**
- Extended thinking (reasoning) support
- Prompt caching (cache_control headers)
- Thinking budget management

### OpenAI

Direct OpenAI API.

```bash
OPENAI_API_KEY=sk-...
```

**Models:** GPT-5.4, GPT-5.4-Pro, GPT-5.3-Codex variants

### z.ai (ZhipuAI / GLM)

```bash
GLM_API_KEY=...
```

**Models:** `glm-5`, `glm-4.7-coding`, `glm-4.5`

### Kimi / Moonshot

```bash
KIMI_API_KEY=...
```

**Models:** `kimi-k2.5`, `kimi-k2-thinking`, `kimi-k2-turbo`

### MiniMax

```bash
MINIMAX_API_KEY=...
```

**Models:** `minimax-m2.5`, `minimax-m2.5-highspeed`, `minimax-m2.1`

### OpenAI Codex (Experimental)

```bash
# Configured via OAuth in hermes setup
```

**Models:** GPT-5.3-5.1 Codex variants

### Custom Endpoints

Any OpenAI-compatible API:

```yaml
# ~/.hermes/config.yaml
model: "my-model"

# Override endpoint via environment
OPENAI_BASE_URL=http://localhost:11434/v1   # Ollama
OPENAI_API_KEY=ollama                        # Placeholder key
```

---

## Authentication

### API Key Auth

Set in `~/.hermes/.env`:

```bash
OPENROUTER_API_KEY=sk-or-...
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GLM_API_KEY=...
KIMI_API_KEY=...
MINIMAX_API_KEY=...
```

### OAuth Device Code (Nous Portal)

```bash
hermes setup    # Prompts for browser auth
```

1. Hermes displays a device code and URL
2. Open the URL in your browser
3. Enter the code and approve
4. Token stored in `~/.hermes/auth.json` (atomic writes + file locking)

Tokens refresh automatically (2-minute skew buffer). PKCE flow for security.

### Resolution Order

Hermes resolves credentials in this order:

1. Environment variables (`NOUS_PORTAL_ACCESS_TOKEN`, `OPENROUTER_API_KEY`, etc.)
2. Auth store (`~/.hermes/auth.json`) — OAuth tokens
3. API key env vars (fallback)

---

## Model Selection

### Interactive Selector

```bash
hermes model
```

Shows a searchable list of all available models with context length, pricing tier, and capabilities.

### Direct Set

```bash
hermes model set anthropic/claude-opus-4.6
hermes model set google/gemini-3-flash-preview
hermes model set openai/gpt-5.4
```

Model format: `provider/model-name` (OpenRouter format) or native model names.

### Mid-Session Switch

```
/model anthropic/claude-opus-4.6
/model google/gemini-3-flash-preview
```

---

## Provider Router

The centralized `call_llm()` / `async_call_llm()` functions route all LLM calls across the system:

| Consumer | Notes |
|----------|-------|
| Main agent loop | Primary model |
| Context compression | Auxiliary model (default: Gemini 3 Flash) |
| Vision analysis | Auxiliary model |
| Web content extraction | Auxiliary model |
| Session search summarization | Auxiliary model |
| Skills Hub assistance | Auxiliary model |
| MCP tool invocation | Auxiliary model |
| Cron jobs | Per-job model override supported |

### Auxiliary Models

Configure separate models for side tasks:

```yaml
auxiliary:
  compression:
    provider: "openrouter"
    model: "google/gemini-3-flash-preview"
  vision:
    provider: "openrouter"
    model: "anthropic/claude-sonnet-4.5"
  web_extract:
    provider: "auto"
    model: ""                # Auto-select based on available keys
```

### Fallback Model

```yaml
fallback_model: "google/gemini-3-flash-preview"
```

Used when the primary model fails or is rate-limited.

---

## Reasoning / Extended Thinking

For models that support it (Claude Opus, some GPT variants):

```
/reasoning xhigh    # Maximum reasoning effort
/reasoning high
/reasoning medium
/reasoning low
/reasoning minimal
/reasoning none     # Disable
```

Configure in config.yaml:

```yaml
agent:
  reasoning_effort: "high"
```

---

## Prompt Caching (Anthropic)

When using Anthropic models natively, Hermes applies `cache_control` headers to the system prompt. Cached tokens cost 10× less on repeat queries with the same system prompt.

Applied automatically. Tracked via `cached_tokens` in session stats.

---

## Context Length Handling

Hermes probes model context length from provider metadata and caches the result. When the context window fills:

1. At 50% threshold — triggers context compression (summarizes old turns)
2. At 70% — emergency trimming
3. At 90% — aggressive trimming with summary injection
4. If 413 Payload Too Large error — immediate compression triggered

---

## Per-Job Model Override (Cron)

Cron jobs can override the default model:

```json
{
  "id": "daily-brief",
  "model": "google/gemini-3-flash-preview",
  "provider": "openrouter",
  "base_url": "https://openrouter.ai/api/v1"
}
```

This allows using cheaper/faster models for routine tasks.
