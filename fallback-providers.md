# Fallback Providers

Hermes Agent has three layers of resilience that keep your sessions running when providers hit issues:

1. **[Credential pools](./credential-pools.md)** -- rotate across multiple API keys for the *same* provider (tried first)
2. **Primary model fallback** -- automatically switches to a *different* provider:model when your main model fails
3. **Auxiliary task fallback** -- independent provider resolution for side tasks like vision, compression, and web extraction

Credential pools handle same-provider rotation (e.g., multiple OpenRouter keys). This page covers cross-provider fallback. Both are optional and work independently.

Primary model fallback was introduced in v0.3.0. The ordered fallback provider chain was added in v0.6.0 ([PR #3813](https://github.com/NousResearch/hermes-agent/pull/3813)). In v0.8.0 the auxiliary auto-detection chain was simplified and vision fallback now tries the main provider first before OpenRouter and Nous Portal ([PR #6041](https://github.com/NousResearch/hermes-agent/pull/6041)).

---

## Primary Model Fallback

When your main LLM provider encounters errors -- rate limits, server overload, auth failures, connection drops -- Hermes can automatically switch to a backup provider:model pair mid-session without losing your conversation.

### Configuration

Add a `fallback_model` section to `~/.hermes/config.yaml`:

```yaml
fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

Both `provider` and `model` are required. If either is missing, the fallback is disabled.

### When Fallback Triggers

The fallback activates automatically when the primary model fails with:

- **Rate limits** (HTTP 429) -- after exhausting retry attempts
- **Server errors** (HTTP 500, 502, 503) -- after exhausting retry attempts
- **Auth failures** (HTTP 401, 403) -- immediately (no point retrying)
- **Not found** (HTTP 404) -- immediately
- **Invalid responses** -- when the API returns malformed or empty responses repeatedly

When triggered, Hermes resolves credentials for the fallback provider, builds a new API client, swaps the model/provider/client in-place, resets the retry counter, and continues the conversation. Your conversation history, tool calls, and context are preserved.

Fallback activates **at most once** per session. If the fallback provider also fails, normal error handling takes over (retries, then error message). This prevents cascading failover loops.

### Custom Endpoint Fallback

For a custom OpenAI-compatible endpoint, add `base_url` and optionally `api_key_env`:

```yaml
fallback_model:
  provider: custom
  model: my-local-model
  base_url: http://localhost:8000/v1
  api_key_env: MY_LOCAL_KEY
```

### Examples

**OpenRouter as fallback for Anthropic native:**
```yaml
model:
  provider: anthropic
  default: claude-sonnet-4-6

fallback_model:
  provider: openrouter
  model: anthropic/claude-sonnet-4
```

**Local model as fallback for cloud:**
```yaml
fallback_model:
  provider: custom
  model: llama-3.1-70b
  base_url: http://localhost:8000/v1
  api_key_env: LOCAL_API_KEY
```

### Where Fallback Works

| Context | Fallback Supported |
|---------|-------------------|
| CLI sessions | Yes |
| Messaging gateway (Telegram, Discord, etc.) | Yes |
| Subagent delegation | No (subagents do not inherit fallback config) |
| Cron jobs | No (run with a fixed provider) |
| Auxiliary tasks (vision, compression) | No (use their own provider chain) |

There are no environment variables for `fallback_model` -- it is configured exclusively through `config.yaml`. This is intentional: fallback configuration is a deliberate choice, not something a stale shell export should override.

---

## Fallback Provider Chain

Added in v0.6.0 ([PR #3813](https://github.com/NousResearch/hermes-agent/pull/3813), closes [#1734](https://github.com/NousResearch/hermes-agent/issues/1734)).

The ordered fallback provider chain extends the legacy single-entry `fallback_model` to support any number of backup providers. When the primary provider fails, Hermes automatically advances through the chain in order until one succeeds or all are exhausted.

### Configuration

```yaml
fallback_providers:
  - provider: openrouter
    model: anthropic/claude-sonnet-4.6
  - provider: anthropic
    model: claude-sonnet-4-6
  - provider: deepseek
    model: deepseek-chat
```

### Trigger Conditions

Fallback is attempted on any of these failures:

| Condition | Example |
|-----------|---------|
| Rate limit | HTTP 429, "too many requests", quota exhausted |
| Server error | HTTP 500, 503, 529 (Anthropic overload) |
| Network failure | Connection reset, DNS failure, timeout |
| Empty/malformed response | Blank streaming response (common rate-limit symptom) |
| Max retries exhausted | Provider repeatedly returns non-fatal errors |

**Fallback is not triggered on:**

- **HTTP 401 / 403** -- authentication or authorization failures. These trigger the fallback immediately rather than after retry backoff, since a credential problem is provider-specific.
- **HTTP 413** -- payload too large. Context compression is attempted first.
- **Context length errors** -- the request needs to be shortened, not re-routed.

### Logging

Each time Hermes activates a fallback, it emits an INFO log entry:

```
INFO Fallback activated: <primary-model> -> <fallback-model> (<fallback-provider>)
```

The status bar in the CLI also shows a brief notification. If all providers in the chain fail, the last error from the final provider is returned to the user.

### Backward Compatibility

The legacy `fallback_model` single-dict format continues to work and is automatically normalized to a one-element chain internally. When both `fallback_providers` (list) and `fallback_model` (dict) are present in `config.yaml`, `fallback_providers` takes precedence.

---

## Auxiliary Task Fallback

Hermes uses separate lightweight models for side tasks. Each task has its own provider resolution chain that acts as a built-in fallback system.

### Tasks with Independent Provider Resolution

| Task | What It Does | Config Key |
|------|-------------|-----------|
| Vision | Image analysis, browser screenshots | `auxiliary.vision` |
| Web Extract | Web page summarization | `auxiliary.web_extract` |
| Compression | Context compression summaries | `auxiliary.compression` or `compression.summary_provider` |
| Session Search | Past session summarization | `auxiliary.session_search` |
| Skills Hub | Skill search and discovery | `auxiliary.skills_hub` |
| MCP | MCP helper operations | `auxiliary.mcp` |
| Memory Flush | Memory consolidation | `auxiliary.flush_memories` |

### Auto-Detection Chain

When a task's provider is set to `"auto"` (the default), Hermes tries providers in order until one works:

**For text tasks (compression, web extract, etc.):**

```text
OpenRouter -> Nous Portal -> Custom endpoint -> Codex OAuth ->
API-key providers (z.ai, Kimi, MiniMax, Hugging Face, Anthropic) -> give up
```

**For vision tasks (v0.8.0):**

```text
OpenRouter -> Nous Portal -> active main provider -> give up
```

If the resolved provider fails at call time, Hermes also has an internal retry: if the provider is not OpenRouter and no explicit `base_url` is set, it tries OpenRouter as a last-resort fallback.

### Configuring Auxiliary Providers

Each task can be configured independently in `config.yaml`:

```yaml
auxiliary:
  vision:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""

  web_extract:
    provider: "auto"
    model: ""

  compression:
    provider: "auto"
    model: ""
```

### Provider Options for Auxiliary Tasks

| Provider | Description | Requirements |
|----------|-------------|-------------|
| `"auto"` | Try providers in order until one works (default) | At least one provider configured |
| `"openrouter"` | Force OpenRouter | `OPENROUTER_API_KEY` |
| `"nous"` | Force Nous Portal | `hermes login` |
| `"codex"` | Force Codex OAuth | `hermes model` -> Codex |
| `"main"` | Use whatever provider the main agent uses | Active main provider configured |
| `"anthropic"` | Force Anthropic native | `ANTHROPIC_API_KEY` or Claude Code credentials |

### Direct Endpoint Override

For any auxiliary task, setting `base_url` bypasses provider resolution entirely:

```yaml
auxiliary:
  vision:
    base_url: "http://localhost:1234/v1"
    api_key: "local-key"
    model: "qwen2.5-vl"
```

---

## Context Compression Fallback

Context compression has a legacy configuration path in addition to the auxiliary system:

```yaml
compression:
  summary_provider: "auto"
  summary_model: "google/gemini-3-flash-preview"
```

This is equivalent to configuring `auxiliary.compression.provider` and `auxiliary.compression.model`. If both are set, the `auxiliary.compression` values take precedence.

If no provider is available for compression, Hermes drops middle conversation turns without generating a summary rather than failing the session.

---

## Delegation Provider Override

Subagents spawned by `delegate_task` do not use the primary fallback model. However, they can be routed to a different provider:model pair for cost optimization:

```yaml
delegation:
  provider: "openrouter"
  model: "google/gemini-3-flash-preview"
```

---

## Cron Job Providers

Cron jobs run with whatever provider is configured at execution time. They do not support a fallback model. To use a different provider for cron jobs, configure `provider` and `model` overrides on the cron job itself:

```python
cronjob(
    action="create",
    schedule="every 2h",
    prompt="Check server status",
    provider="openrouter",
    model="google/gemini-3-flash-preview"
)
```

---

## Summary

| Feature | Fallback Mechanism | Config Location |
|---------|-------------------|----------------|
| Main agent model | `fallback_model` or `fallback_providers` in config.yaml | Top-level |
| Vision | Auto-detection chain + internal OpenRouter retry | `auxiliary.vision` |
| Web extraction | Auto-detection chain + internal OpenRouter retry | `auxiliary.web_extract` |
| Context compression | Auto-detection chain, degrades to no-summary if unavailable | `auxiliary.compression` or `compression.summary_provider` |
| Session search | Auto-detection chain | `auxiliary.session_search` |
| Skills hub | Auto-detection chain | `auxiliary.skills_hub` |
| MCP helpers | Auto-detection chain | `auxiliary.mcp` |
| Memory flush | Auto-detection chain | `auxiliary.flush_memories` |
| Delegation | Provider override only (no automatic fallback) | `delegation.provider` / `delegation.model` |
| Cron jobs | Per-job provider override only (no automatic fallback) | Per-job `provider` / `model` |
