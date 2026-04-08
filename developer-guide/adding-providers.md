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

The important abstraction is `api_mode`:

- Most providers use `chat_completions`
- Codex uses `codex_responses`
- Anthropic uses `anthropic_messages`
- A new non-OpenAI protocol usually means adding a new adapter and a new `api_mode` branch

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

### Step 2: Add auth metadata

Add a `ProviderConfig` entry to `PROVIDER_REGISTRY` in `hermes_cli/auth.py` with id, name, auth_type, inference_base_url, api_key_env_vars. Also add aliases.

### Step 3: Add model catalog and aliases

Update `_PROVIDER_MODELS`, `_PROVIDER_LABELS`, `_PROVIDER_ALIASES`, provider display order in `list_available_providers()`, and `provider_model_ids()` if the provider supports a live `/models` fetch.

### Step 4: Resolve runtime data

Add a branch in `resolve_runtime_provider()` that returns provider, api_mode, base_url, api_key, and source. Be careful with API-key precedence.

### Step 5: Wire the CLI

Update `hermes_cli/main.py`: provider_labels dict, providers list, provider dispatch, `--provider` argument choices, login/logout choices.

### Step 6: Keep auxiliary calls working

Add a cheap/fast default aux model to `_API_KEY_PROVIDER_AUX_MODELS` in `agent/auxiliary_client.py`. Add context lengths in `agent/model_metadata.py`.

### Step 7: Native provider adapter (if needed)

Isolate provider-specific logic in `agent/<provider>_adapter.py`. In `run_agent.py`, search for `api_mode` and audit every switch point: `__init__`, client construction, `_build_api_kwargs()`, `_api_call_with_interrupt()`, response validation, finish-reason extraction, token-usage extraction, fallback-model activation.

### Step 8: Tests

Common test files to update:
- `tests/test_runtime_provider_resolution.py`
- `tests/test_cli_provider_resolution.py`
- `tests/test_cli_model_command.py`
- `tests/test_setup_model_selection.py`
- `tests/test_provider_parity.py`
- `tests/test_run_agent.py`

### Step 9: Live verification

```bash
source venv/bin/activate
python -m hermes_cli.main chat -q "Say hello" --provider your-provider --model your-model
```

### Step 10: Update user-facing docs

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
