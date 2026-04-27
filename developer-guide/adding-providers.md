# Adding Providers

Hermes can already talk to any OpenAI-compatible endpoint through the custom provider path. Do not add a built-in provider unless you want first-class UX for that service: provider-specific auth or token refresh, a curated model catalog, setup / `hermes model` menu entries, provider aliases for `provider:model` syntax, or a non-OpenAI API shape that needs an adapter.

If the provider is just "another OpenAI-compatible base URL and API key", a named custom provider may be enough.

## The Mental Model

A built-in provider has to line up across a few layers:

1. `hermes_cli/auth.py` decides how credentials are found
2. `hermes_cli/runtime_provider.py` turns that into runtime data (provider, api_mode, base_url, api_key, source)
3. `run_agent.py` uses `api_mode` to decide how requests are built and sent
4. `hermes_cli/models.py` and `hermes_cli/main.py` make the provider show up in the CLI
5. `agent/auxiliary_client.py` and `agent/model_metadata.py` keep side tasks and token budgeting working

The important abstraction is `api_mode`. In v0.11.0 each `api_mode` is owned by a `ProviderTransport` subclass under `agent/transports/` (see [Transport Layer](./transport-layer.md)):

| `api_mode` | Transport class | Used for |
|------------|-----------------|----------|
| `chat_completions` | `ChatCompletionsTransport` | OpenAI-compatible providers (most providers) |
| `codex_responses` | `ResponsesApiTransport` | Codex / Responses API (incl. xAI Grok Responses) |
| `anthropic_messages` | `AnthropicTransport` | Native Anthropic |
| `bedrock_converse` | `BedrockTransport` | AWS Bedrock |

A new non-OpenAI protocol means picking or implementing a `ProviderTransport`, registering it with an `api_mode` string, then wiring that string into runtime resolution.

## Choose the Implementation Path

### Path A -- OpenAI-compatible provider

Use this when the provider accepts standard chat-completions style requests. Typical work: add auth metadata, add model catalog/aliases, add runtime resolution, add CLI menu wiring, add aux-model defaults, add tests and user docs. You usually do not need a new adapter or a new `api_mode`.

### Path B -- Native provider

Use this when the provider does not behave like OpenAI chat completions. Examples in-tree today: `codex_responses`, `anthropic_messages`. This path includes everything from Path A plus a provider adapter in `agent/`, `run_agent.py` branches, and adapter tests.

## File Checklist

### Required for every built-in provider

1. `hermes_cli/auth.py` -- add `ProviderConfig` to `PROVIDER_REGISTRY` and aliases to `_PROVIDER_ALIASES`
2. `hermes_cli/models.py` -- add model catalog, labels, aliases
3. `hermes_cli/runtime_provider.py` -- add resolution branch returning provider, api_mode, base_url, api_key, source
4. `hermes_cli/main.py` -- add to provider labels, providers list, dispatch, `--provider` choices
5. `agent/auxiliary_client.py` -- add default aux model to `_API_KEY_PROVIDER_AUX_MODELS`
6. `agent/model_metadata.py` -- add context lengths for the provider's models
7. Tests
8. User-facing docs

Note: `hermes_cli/setup.py` does not need changes -- the setup wizard delegates provider/model selection to `select_provider_and_model()` in `main.py`, so any provider added there is automatically available in `hermes setup`.

### Additional for native / non-OpenAI providers

- `agent/<provider>_adapter.py` -- build client, resolve tokens, convert messages, normalize responses
- `run_agent.py` -- audit every `api_mode` switch point
- `pyproject.toml` if a provider SDK is required

## Step-by-Step Guide

### Step 1: Pick one canonical provider id

Choose a single provider id and use it everywhere (e.g. `openai-codex`, `kimi-coding`, `minimax-cn`). The same id must appear in `PROVIDER_REGISTRY`, `_PROVIDER_LABELS`, `_PROVIDER_ALIASES`, CLI `--provider` choices, auxiliary-model defaults, and tests.

### Step 2: Pick or implement a `ProviderTransport`

Decide which transport handles your provider's wire protocol **before** wiring runtime resolution. Most providers reuse one of the four shipped transports — only invent a new one if the protocol is genuinely new.

| Wire protocol | `api_mode` | Transport (in `agent/transports/`) |
|---------------|-----------|-----------------------------------|
| OpenAI Chat Completions | `chat_completions` | `ChatCompletionsTransport` (`chat_completions.py`) |
| OpenAI Responses API | `codex_responses` | `ResponsesApiTransport` (`codex.py`) |
| Anthropic Messages | `anthropic_messages` | `AnthropicTransport` (`anthropic.py`) |
| AWS Bedrock Converse | `bedrock_converse` | `BedrockTransport` (`bedrock.py`) |

If your provider speaks one of these protocols, just record the matching `api_mode` string — you do not need a new transport. If your provider needs a brand-new protocol, subclass `ProviderTransport`, implement the four required methods (`convert_messages`, `convert_tools`, `build_kwargs`, `normalize_response`), override the optional hooks (`validate_response`, `extract_cache_stats`, `map_finish_reason`) when behavior differs, and register at module import:

```python
# agent/transports/myproto.py
from agent.transports import register_transport
from agent.transports.base import ProviderTransport

class MyProtoTransport(ProviderTransport):
    @property
    def api_mode(self) -> str:
        return "myproto"
    # ... implement the four required methods ...

register_transport("myproto", MyProtoTransport)
```

Add a guarded import for the new module to `_discover_transports()` in `agent/transports/__init__.py` so the registry can lazy-import it.

`BedrockTransport` is a worked example of a non-OpenAI transport: it uses its own boto3 client (not the OpenAI SDK), owns format conversion via `agent/bedrock_adapter.py`, and leaves client construction + `converse()` invocation on `AIAgent`. The transport class itself is a thin façade over the adapter — about 150 lines.

See [Transport Layer](./transport-layer.md) for the full ABC contract and testing checklist.

### Step 3: Add auth metadata

Add a `ProviderConfig` entry to `PROVIDER_REGISTRY` in `hermes_cli/auth.py` with id, name, auth_type, inference_base_url, api_key_env_vars. Also add aliases.

### Step 4: Add model catalog and aliases

Update `_PROVIDER_MODELS`, `_PROVIDER_LABELS`, `_PROVIDER_ALIASES`, provider display order in `list_available_providers()`, and `provider_model_ids()` if the provider supports a live `/models` fetch.

### Step 5: Resolve runtime data

Add a branch in `resolve_runtime_provider()` that returns provider, **the `api_mode` you picked in Step 2**, base_url, api_key, and source. Be careful with API-key precedence. The agent loop uses `api_mode` to look up the transport via `agent.transports.get_transport(api_mode)`, so the string must match a registered transport.

### Step 6: Wire the CLI

Update `hermes_cli/main.py`: provider_labels dict, providers list, provider dispatch, `--provider` argument choices, login/logout choices.

### Step 7: Keep auxiliary calls working

Add a cheap/fast default aux model to `_API_KEY_PROVIDER_AUX_MODELS` in `agent/auxiliary_client.py`. Add context lengths in `agent/model_metadata.py`.

### Step 8: Native provider adapter (if needed)

If you implemented a new transport in Step 2, the provider-specific format conversion logic typically lives in a sibling adapter module (`agent/<provider>_adapter.py`) that the transport delegates to — `agent/anthropic_adapter.py`, `agent/codex_responses_adapter.py`, and `agent/bedrock_adapter.py` are the in-tree examples. Keep `run_agent.py` free of provider branches; the transport is the only place that should know about the protocol shape.

### Step 9: Tests

Common test files to update:
- `tests/test_runtime_provider_resolution.py`
- `tests/test_cli_provider_resolution.py`
- `tests/test_cli_model_command.py`
- `tests/test_setup_model_selection.py`
- `tests/test_provider_parity.py`
- `tests/test_run_agent.py`

### Step 10: Live verification

```bash
source venv/bin/activate
python -m hermes_cli.main chat -q "Say hello" --provider your-provider --model your-model
```

### Step 11: Update user-facing docs

Update quickstart, configuration, and environment variables reference docs.

## Google AI Studio as a Reference Implementation

Google AI Studio (`gemini`) is a good reference for Path A (OpenAI-compatible provider):

- Uses Google's OpenAI-compatible endpoint (`/v1beta/openai`) — no adapter needed
- Two env var aliases for the API key: `GOOGLE_API_KEY` (primary) and `GEMINI_API_KEY`
- Context lengths resolved via models.dev by provider ID `"gemini"` — the `generativelanguage.googleapis.com` base URL is mapped to `"gemini"` in `_URL_TO_PROVIDER` in `agent/model_metadata.py`, enabling automatic context length detection for any Gemini model
- Provider aliases cover three common names: `google`, `google-gemini`, `google-ai-studio`
- Gemma open models (e.g. `gemma-4-31b-it`) appear in the same catalog since they are served through AI Studio

The key insight: by adding a URL-to-provider mapping in `model_metadata.py`, any provider using a recognizable base URL automatically benefits from models.dev context length lookups, even for custom endpoints pointing at that provider.

## Common Pitfalls

1. **Adding auth but not model parsing** -- credentials resolve but `/model` and `provider:model` inputs fail
2. **Forgetting config model can be string or dict** -- provider-selection code must normalize both forms
3. **Assuming a built-in provider is required** -- custom provider may already solve the problem with less maintenance
4. **Forgetting auxiliary paths** -- main chat works but summarization, memory flushes, or vision helpers fail
5. **Native-provider branches hiding in run_agent.py** -- search for `api_mode` and `self.client.`
6. **Sending OpenRouter-only knobs to other providers** -- provider routing fields belong only on providers that support them

## OpenAI-compatible Provider Checklist

- `ProviderConfig` added in `hermes_cli/auth.py`
- Aliases added in `hermes_cli/auth.py` and `hermes_cli/models.py`
- Model catalog added in `hermes_cli/models.py`
- Runtime branch added in `hermes_cli/runtime_provider.py`
- CLI wiring added in `hermes_cli/main.py`
- Aux model added in `agent/auxiliary_client.py`
- Context lengths added in `agent/model_metadata.py`
- Runtime / CLI tests updated
- User docs updated

## Native Provider Checklist

Everything in the OpenAI-compatible checklist, plus:

- Adapter added in `agent/<provider>_adapter.py`
- New `api_mode` supported in `run_agent.py`
- Interrupt / rebuild path works
- Usage and finish-reason extraction works
- Fallback path works
- Adapter tests added
- Live smoke test passes
