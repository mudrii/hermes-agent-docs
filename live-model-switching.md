# Live Model Switching

Added in v0.8.0 ([PR #5181](https://github.com/NousResearch/hermes-agent/pull/5181), [#5742](https://github.com/NousResearch/hermes-agent/pull/5742)).

The `/model` command switches model and provider mid-session without losing conversation history, tool calls, or context. The switch pipeline is implemented in `hermes_cli/model_switch.py` and shared between the CLI and all gateway platforms.

---

## Basic Usage

```
/model                              — show current model + available providers
/model <name>                       — switch model (session only)
/model <name> --global              — switch and persist to config.yaml
/model <name> --provider <slug>     — switch provider and model
/model --provider <slug>            — switch provider, auto-detect model
```

**Show current model:**

```
/model
```

With no arguments, `/model` shows the current model and provider. On Telegram and Discord it launches an interactive picker (see below). On other platforms it prints a text list of available providers and models.

**Switch model for this session:**

```
/model sonnet
/model gemini-3.1-pro-preview
/model gpt-5.4
```

Aliases like `sonnet`, `opus`, `grok`, `gemini`, `gpt5`, `codex` are resolved dynamically against the live models.dev catalog and map to the latest matching model.

**Switch provider and model:**

```
/model claude-sonnet-4-6 --provider anthropic
/model gemini-3.1-pro-preview --provider gemini
/model gpt-5.4 --provider openrouter
```

**Persist a switch globally:**

```
/model claude-sonnet-4-6 --provider anthropic --global
```

`--global` writes `model.default` and `model.provider` to `~/.hermes/config.yaml` so the switch survives session restarts.

---

## Where It Works

| Platform | Supported | Interactive Picker |
|----------|-----------|-------------------|
| CLI (`hermes chat`) | Yes | No (text list) |
| Telegram | Yes | Yes (paginated inline buttons) |
| Discord | Yes | Yes (Select menus) |
| Slack | Yes | No (text list) |
| WhatsApp | Yes | No (text list) |
| Signal | Yes | No (text list) |
| Matrix | Yes | No (text list) |
| Mattermost | Yes | No (text list) |
| Feishu / Lark | Yes | No (text list) |
| WeCom | Yes | No (text list) |
| API Server | Yes | No |
| Cron jobs | No | — |

---

## Session vs Global Persistence

A `/model` switch **without `--global`** is session-only:

- The switch is stored in memory (`_session_model_overrides`).
- `config.yaml` is **not modified**.
- The next new session (`/new`, bot restart, fresh CLI invocation) uses the model from `config.yaml`.
- The agent receives a note at the start of the next turn so it can update its self-identification.

A `/model` switch **with `--global`** is persistent:

- `config.yaml` is updated with `model.default` and `model.provider`.
- All future sessions (including new CLI invocations and bot restarts) use the new model.

---

## Aggregator-Aware Resolution

The switch pipeline uses the current provider context to minimize unnecessary provider changes:

1. When the requested model is available on OpenRouter or Nous Portal (which shares OpenRouter's curated model catalog), the provider stays the same — only the model name changes.
2. When the model is not available on the current aggregator, Hermes searches authenticated providers in order and falls back to the first one that has the model.
3. When `--provider` is given explicitly, that provider is used directly.

This means `/model gpt-5.4` while on OpenRouter stays on OpenRouter as `openai/gpt-5.4`, but `/model claude-sonnet-4-6 --provider anthropic` switches to the native Anthropic endpoint.

---

## Interactive Pickers

### Telegram

When `/model` is sent with no arguments, Hermes sends an inline keyboard message (edits in-place as you navigate):

1. **Provider selection:** Inline buttons, 2 per row, showing provider name and model count. The current provider is marked with `✓`.
2. **Model selection:** Paginated list, 8 models per page with **◀ Prev** and **Next ▶** navigation. Models are displayed as short names (vendor prefix stripped). A **◀ Back** button returns to provider selection.
3. **Confirmation:** After selecting a model, Hermes sends a confirmation message showing the new model, provider, context window, max output, cost (if available), and capabilities.

If more models exist than fit on the picker pages, a note like `_(42 more available — type /model <name> directly)_` is shown.

### Discord

When `/model` is sent with no arguments, Hermes sends an embedded message with Discord Select menus:

1. **Provider dropdown:** Select from all authenticated providers.
2. **Model dropdown:** Select from that provider's model list.
3. **Confirmation:** Embed updated with the switch result.

---

## Pricing Display

When switching models on OpenRouter or Nous Portal, the confirmation message includes live pricing:

```
Model switched to `anthropic/claude-sonnet-4.6`
Provider: OpenRouter
Context: 1,000,000 tokens
Max output: 64,000 tokens
Cost: in $3.00 · out $15.00/Mtok
Capabilities: tools, vision, reasoning
_(session only — use /model <name> --global to persist)_
```

Pricing is fetched from the provider's `/models` endpoint and formatted as `$/Mtok`. A price of `0` is displayed as `free`. Cache read pricing is shown when available.

---

## Model Aliases

Short aliases are resolved dynamically against the models.dev catalog. The alias maps to a `(vendor, family)` pair and the catalog is queried for the latest matching model.

| Alias | Resolves to (example) |
|-------|-----------------------|
| `sonnet` | `anthropic/claude-sonnet-4.6` (on aggregators) or `claude-sonnet-4-6` (on Anthropic native) |
| `opus` | `anthropic/claude-opus-4.6` |
| `haiku` | `anthropic/claude-haiku-4.5` |
| `gpt5` | `openai/gpt-5.4` |
| `codex` | `openai/gpt-5.3-codex` |
| `gemini` | `google/gemini-3.1-pro-preview` |
| `grok` | `x-ai/grok-4.20-beta` |
| `deepseek` | `deepseek/deepseek-chat` |

---

## Non-Agentic Model Warning

If you switch to a model whose name contains `hermes` (e.g. `NousResearch/Hermes-3-Llama-3.1-70B`), Hermes emits a warning:

```
Nous Research Hermes 3 & 4 models are NOT agentic and are not designed
for use with Hermes Agent. They lack the tool-calling capabilities
required for agent workflows.
```

---

## Implementation Notes

- **Pipeline:** `hermes_cli/model_switch.py` — `switch_model()` handles flag parsing, alias resolution, aggregator detection, credential resolution, model name normalization, and result construction. The CLI and gateway share this function identically.
- **Aggregator check:** `hermes_cli/providers.py` — `is_aggregator()` returns `True` for `openrouter`, `vercel`, `opencode`, `opencode-go`, `kilo`, and `huggingface`. Non-aggregators (including `nous`) accept bare model names rather than `vendor/model` slugs.
- **Model normalization:** `hermes_cli/model_normalize.py` — `normalize_model_for_provider()` applies per-provider formatting rules (e.g. Anthropic bare names vs OpenRouter vendor/name slugs).
- **Metadata:** `agent/models_dev.py` — `get_model_info()` fetches `ModelInfo` (context window, max output, cost, capabilities) from the models.dev registry for the confirmation message.
- **In-place agent update:** When a cached agent exists for the session, `agent.switch_model()` is called to update model, provider, api_key, base_url, and api_mode in-place without creating a new agent instance.
