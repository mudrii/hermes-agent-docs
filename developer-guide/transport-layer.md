---
sidebar_position: 6
title: "Transport Layer"
description: "ProviderTransport ABC contract, the four concrete transports, NormalizedResponse, and how api_mode resolves to a transport instance"
---

# Transport Layer

The transport layer is the v0.11.0 abstraction that owns provider-specific format conversion and response normalization. Before v0.11.0, `run_agent.py` carried inline `if api_mode == ...` branches for every protocol; v0.11.0 extracted those branches into a small ABC under `agent/transports/` so each protocol lives in its own file and `AIAgent` only sees a normalized shape.

Source path: `agent/transports/`

```
agent/transports/
├── base.py             # ProviderTransport ABC
├── types.py            # NormalizedResponse, ToolCall, Usage, build_tool_call
├── anthropic.py        # AnthropicTransport — anthropic_messages
├── chat_completions.py # ChatCompletionsTransport — chat_completions (OpenAI-compatible)
├── codex.py            # ResponsesApiTransport — codex_responses
├── bedrock.py          # BedrockTransport — bedrock_converse
└── __init__.py         # _REGISTRY + register_transport / get_transport
```

## What the Transport Layer Owns

A transport owns the **data path** for one `api_mode`:

```
convert_messages → convert_tools → build_kwargs → normalize_response
```

In other words, it knows how to translate the agent's internal OpenAI-flavored message and tool format into the provider's native shape, build the kwargs dict that the provider SDK expects, and turn the raw response back into the canonical `NormalizedResponse` that the rest of the agent consumes.

## What the Transport Layer Does NOT Own

These responsibilities stay on `AIAgent` (`run_agent.py`) and are explicitly **not** part of the transport contract:

- **Streaming.** Stream readers, delta callbacks, and the interrupt event live on `AIAgent`.
- **Retry logic.** The retry loop, backoff, and `agent.api_max_retries` are owned by `AIAgent`.
- **Prompt caching.** Cache markers and cache-aware prompt assembly live in `agent/prompt_caching.py`.
- **Credential refresh.** OAuth refresh, Nous Portal token rotation, and Bedrock IAM live in `hermes_cli/runtime_provider.py` and `hermes_cli/auth.py`.
- **Client construction.** The provider SDK client (`openai.OpenAI`, `anthropic.Anthropic`, `boto3` Bedrock) is built by `AIAgent`; the transport receives a raw response, never a client.

This separation is what makes the transport ABC small enough to be useful: a new transport implements four (sometimes seven) methods and inherits everything else.

## The ABC Contract

```python
# agent/transports/base.py
class ProviderTransport(ABC):

    @property
    @abstractmethod
    def api_mode(self) -> str:
        """The api_mode string this transport handles."""

    @abstractmethod
    def convert_messages(self, messages: List[Dict[str, Any]], **kwargs) -> Any: ...

    @abstractmethod
    def convert_tools(self, tools: List[Dict[str, Any]]) -> Any: ...

    @abstractmethod
    def build_kwargs(
        self,
        model: str,
        messages: List[Dict[str, Any]],
        tools: Optional[List[Dict[str, Any]]] = None,
        **params,
    ) -> Dict[str, Any]: ...

    @abstractmethod
    def normalize_response(self, response: Any, **kwargs) -> NormalizedResponse: ...

    # Optional hooks with default implementations
    def validate_response(self, response: Any) -> bool: ...
    def extract_cache_stats(self, response: Any) -> Optional[Dict[str, int]]: ...
    def map_finish_reason(self, raw_reason: str) -> str: ...
```

### Required methods (4)

| Method | Purpose |
|--------|---------|
| `convert_messages` | Translate the agent's OpenAI-format conversation history into the provider's native message structure. For Anthropic this returns `(system, messages)`; for chat_completions it sanitizes Codex leaks and otherwise passes through; for Codex it returns Responses API input items; for Bedrock it returns Converse content blocks. |
| `convert_tools` | Translate OpenAI-format tool schemas into the provider's tool definition shape (e.g. Anthropic `input_schema`, Bedrock `toolConfig`, Responses API `function` items). |
| `build_kwargs` | The primary entry point. Calls `convert_messages` and `convert_tools` internally and returns the complete kwargs dict ready to pass to the provider SDK call. Provider-specific knobs — reasoning effort, temperature handling, max tokens defaults, `extra_body` assembly — all live here. |
| `normalize_response` | Convert the raw provider response into a `NormalizedResponse`. This is the only method that returns a transport-layer type. |

### Optional hooks (3)

| Method | Default | When to override |
|--------|---------|------------------|
| `validate_response` | returns `True` | Override when the provider can return structurally degraded responses (empty content, missing tool-call IDs) that the agent should treat as a retryable failure rather than a real reply. |
| `extract_cache_stats` | returns `None` | Override for providers that report cache hits/creation tokens differently from the standard OpenAI `usage.prompt_tokens_details.cached_tokens`. |
| `map_finish_reason` | returns the raw reason unchanged | Override when the provider uses a different stop-reason vocabulary (e.g. `end_turn` → `stop`). |

## The Four Concrete Transports

Each transport is one file under `agent/transports/`. Format conversion logic still lives in adapter modules (`agent/anthropic_adapter.py`, `agent/codex_responses_adapter.py`, `agent/bedrock_adapter.py`); the transport classes are thin façades over those adapters.

| Class | File | `api_mode` | Backing adapter |
|-------|------|-----------|-----------------|
| `AnthropicTransport` | `anthropic.py` | `anthropic_messages` | `agent/anthropic_adapter.py` |
| `ChatCompletionsTransport` | `chat_completions.py` | `chat_completions` | (inline; provider is OpenAI-compatible) |
| `ResponsesApiTransport` | `codex.py` | `codex_responses` | `agent/codex_responses_adapter.py` |
| `BedrockTransport` | `bedrock.py` | `bedrock_converse` | `agent/bedrock_adapter.py` |

### `AnthropicTransport` — `anthropic_messages`

Used by the native Anthropic provider and any base URL that resolves to Anthropic Messages. `convert_messages` returns the `(system, messages)` tuple; `build_kwargs` honors prompt caching markers from `agent/prompt_caching.py`; `normalize_response` produces a `NormalizedResponse` with `provider_data["reasoning_details"]` populated when extended thinking is on.

### `ChatCompletionsTransport` — `chat_completions`

The default path used by ~16 OpenAI-compatible providers (OpenRouter, Nous Portal, NVIDIA NIM, Step Plan, Vercel ai-gateway, Qwen, Ollama, DeepSeek, xAI, Kimi, MiniMax, etc.). Messages and tools are already in OpenAI format, so `convert_messages` only sanitizes leaked Codex fields (`codex_reasoning_items`, `codex_message_items`, `call_id`, `response_item_id`) that strict providers reject with 400/422. The complexity lives in `build_kwargs`, which carries provider-specific conditionals for max-tokens defaults, reasoning configuration, temperature handling, Moonshot tool sanitization, and `extra_body` assembly.

### `ResponsesApiTransport` — `codex_responses`

OpenAI Responses API path used by Codex (GPT-5 and GPT-5.5 over Codex OAuth) and by xAI Grok in Responses mode. `convert_messages` translates chat messages to Responses API input items via `_chat_messages_to_responses_input`; `build_kwargs` accepts `instructions`, `reasoning_config`, `session_id` (used for `prompt_cache_key` and the xAI conversation header), and `request_overrides`; `normalize_response` extracts function-call output items and preserves `codex_reasoning_items` / `codex_message_items` in `provider_data` so subsequent turns can replay them.

### `BedrockTransport` — `bedrock_converse`

AWS Bedrock Converse API. Bedrock uses its own boto3 client (not the OpenAI SDK), so the transport owns format conversion and normalization while the boto3 client and `converse()` call stay on `AIAgent`. `build_kwargs` honors AWS region, inference profiles (`us.` / `global.` prefixes), and Bedrock Guardrails configuration.

## How `api_mode` Maps to a Transport

Provider runtime resolution (see [Provider Runtime Resolution](./provider-runtime.md)) returns an `api_mode` string in the `(provider, api_mode, base_url, api_key, source)` tuple. `AIAgent` looks up the transport by that string:

```python
from agent.transports import get_transport

transport = get_transport(api_mode)   # e.g. "anthropic_messages"
if transport is None:
    # Legacy code path — no transport registered for this api_mode.
    # Used during gradual migration; new api_modes always have a transport.
    ...
else:
    kwargs = transport.build_kwargs(model, messages, tools, **params)
    raw = client.create(**kwargs)               # streaming or non-streaming
    nr = transport.normalize_response(raw)
```

The mapping that ships in v0.11.0:

| `api_mode` | Transport |
|------------|-----------|
| `anthropic_messages` | `AnthropicTransport` |
| `chat_completions` | `ChatCompletionsTransport` |
| `codex_responses` | `ResponsesApiTransport` |
| `bedrock_converse` | `BedrockTransport` |

`get_transport()` returns `None` when no transport is registered, which lets call sites fall back to the legacy in-line code path during gradual migration. In v0.11.0 every shipped `api_mode` has a registered transport.

### Registration

Each transport module registers itself at import time via the small registry in `agent/transports/__init__.py`:

```python
# agent/transports/anthropic.py — bottom of file
from agent.transports import register_transport
register_transport("anthropic_messages", AnthropicTransport)
```

`get_transport()` triggers `_discover_transports()` on a registry miss, which imports every transport module so test/order-dependent imports do not make valid `api_mode`s unavailable.

## `NormalizedResponse`

Every transport's `normalize_response()` returns the same shape:

```python
# agent/transports/types.py
@dataclass
class NormalizedResponse:
    content: Optional[str]
    tool_calls: Optional[List[ToolCall]]
    finish_reason: str  # "stop" | "tool_calls" | "length" | "content_filter"
    reasoning: Optional[str] = None
    usage: Optional[Usage] = None
    provider_data: Optional[Dict[str, Any]] = None
```

The shared fields are truly cross-provider — every caller in `run_agent.py` reads them without branching on `api_mode`. Protocol-specific state (Anthropic `reasoning_details`, Codex `codex_reasoning_items` / `codex_message_items`, Gemini `extra_content` thought signatures) lives in the optional `provider_data` dict so only protocol-aware code paths read it.

`ToolCall` mirrors the same pattern:

```python
@dataclass
class ToolCall:
    id: Optional[str]
    name: str
    arguments: str          # JSON string
    provider_data: Optional[Dict[str, Any]] = None
```

`ToolCall` exposes backward-compatibility properties (`type`, `function`, `call_id`, `response_item_id`, `extra_content`) so the 45+ sites in `run_agent.py` that read `tc.function.name` and `tc.function.arguments` keep working without a shim.

`Usage` is the canonical token-count dataclass: `prompt_tokens`, `completion_tokens`, `total_tokens`, `cached_tokens`.

The `build_tool_call(id, name, arguments, **provider_fields)` helper auto-serializes dict arguments to JSON and packs any extra keyword arguments into `provider_data`.

## Adding a New Transport

Adding a non-OpenAI protocol is a four-step process:

1. **Subclass `ProviderTransport`.** Implement the four required methods. Override the optional hooks only when the provider's behavior differs from defaults.
2. **Pick an `api_mode` string.** Convention is `<protocol>_<flavor>` — e.g. `anthropic_messages`, `bedrock_converse`. The string is also what runtime provider resolution returns for this provider.
3. **Register at module import.** Add `register_transport("<api_mode>", MyTransport)` at the bottom of the new module, plus a guarded import inside `_discover_transports()` in `agent/transports/__init__.py`.
4. **Wire into the provider catalog.** Update `hermes_cli/runtime_provider.py` so the new provider's resolution branch returns the matching `api_mode`. See [Adding Providers](./adding-providers.md) for the full provider-side checklist (auth, model catalog, CLI wiring, auxiliary models, context lengths).

If the provider is OpenAI-compatible (chat completions shape), you do **not** need a new transport — just add a runtime-resolution branch that returns `api_mode = "chat_completions"`.

### Testing checklist

- Unit-test `convert_messages` with: system + user + assistant + tool messages, parallel tool calls, mixed content (text + image inputs).
- Unit-test `convert_tools` with at least one schema that exercises nested object types.
- Unit-test `build_kwargs` for the model-specific knobs your provider uses (reasoning effort, temperature handling, `max_tokens` defaults).
- Unit-test `normalize_response` for: text-only response, tool-call response, length-cutoff, content-filter, and (if applicable) reasoning content.
- Add an integration test that round-trips one full turn through `AIAgent` against a recorded provider response.

## Related Docs

- [Provider Runtime Resolution](./provider-runtime.md) — how a `(provider, model)` tuple maps to `(api_mode, base_url, api_key)`.
- [Agent Loop Internals](./agent-loop.md) — where the transport is invoked inside the turn lifecycle.
- [Adding Providers](./adding-providers.md) — full checklist for shipping a new built-in provider, including transport selection.
- [Architecture Overview](./architecture.md) — the top-level map; the transport layer sits between provider runtime resolution and the agent loop.
