# Provider Routing

When using [OpenRouter](https://openrouter.ai) as your LLM provider, Hermes Agent supports **provider routing** -- fine-grained control over which underlying AI providers handle your requests and how they're prioritized.

OpenRouter routes requests to many providers (e.g., Anthropic, Google, AWS Bedrock, Together AI). Provider routing lets you optimize for cost, speed, quality, or enforce specific provider requirements.

Provider routing via `provider_preferences` has been available since v0.2.0.

## Smart Model Routing

Smart model routing is a separate feature that sends short, simple turns to a cheap model while routing complex turns to your primary model. This reduces costs on quick interactions without sacrificing quality for complex work.

Implemented in `agent/smart_model_routing.py`.

### Configuration

```yaml
smart_model_routing:
  enabled: true
  max_simple_chars: 160
  max_simple_words: 28
  cheap_model:
    provider: openrouter
    model: google/gemini-2.5-flash
```

### Complexity Detection

A message is classified as "complex" (routed to the primary model) when any of these conditions is true:

- Exceeds `max_simple_chars` (default: 160 characters)
- Exceeds `max_simple_words` (default: 28 words)
- Contains a URL
- Contains any complexity keyword: `debug`, `implement`, `refactor`, `patch`, `traceback`, `exception`, `error`, `analyze`, `investigate`, `architecture`, `design`, `compare`, `benchmark`, `optimize`, `review`, `terminal`, `shell`, `tool`, `pytest`, `test`, `plan`, `delegate`, `subagent`, `cron`, `docker`, `kubernetes`, and others

Otherwise the message is classified as "simple" and sent to the cheap model.

---

## OpenRouter Provider Routing

### Configuration

Add a `provider_routing` section to `~/.hermes/config.yaml`:

```yaml
provider_routing:
  sort: "price"           # How to rank providers
  only: []                # Whitelist: only use these providers
  ignore: []              # Blacklist: never use these providers
  order: []               # Explicit provider priority order
  require_parameters: false  # Only use providers that support all parameters
  data_collection: null   # Control data collection ("allow" or "deny")
```

Provider routing only applies when using OpenRouter. It has no effect with direct provider connections (e.g., connecting directly to the Anthropic API).

### Options

#### `sort`

Controls how OpenRouter ranks available providers for your request.

| Value | Description |
|-------|-------------|
| `"price"` | Cheapest provider first |
| `"throughput"` | Fastest tokens-per-second first |
| `"latency"` | Lowest time-to-first-token first |

#### `only`

Whitelist of provider names. When set, **only** these providers will be used. All others are excluded.

```yaml
provider_routing:
  only:
    - "Anthropic"
    - "Google"
```

#### `ignore`

Blacklist of provider names. These providers will **never** be used, even if they offer the cheapest or fastest option.

```yaml
provider_routing:
  ignore:
    - "Together"
    - "DeepInfra"
```

#### `order`

Explicit priority order. Providers listed first are preferred. Unlisted providers are used as fallbacks.

```yaml
provider_routing:
  order:
    - "Anthropic"
    - "Google"
    - "AWS Bedrock"
```

#### `require_parameters`

When `true`, OpenRouter will only route to providers that support **all** parameters in your request (like `temperature`, `top_p`, `tools`, etc.). This avoids silent parameter drops.

#### `data_collection`

Controls whether providers can use your prompts for training. Options are `"allow"` or `"deny"`.

### Practical Examples

**Optimize for cost:**

```yaml
provider_routing:
  sort: "price"
```

**Optimize for speed:**

```yaml
provider_routing:
  sort: "latency"
```

**Lock to specific providers:**

```yaml
provider_routing:
  only:
    - "Anthropic"
```

**Avoid specific providers for data privacy:**

```yaml
provider_routing:
  ignore:
    - "Together"
    - "Lepton"
  data_collection: "deny"
```

**Preferred order with fallbacks:**

```yaml
provider_routing:
  order:
    - "Anthropic"
    - "Google"
  require_parameters: true
```

### How It Works

Provider routing preferences are passed to the OpenRouter API via the `extra_body.provider` field on every API call. This applies to both CLI mode and gateway mode.

The routing config is read from `config.yaml` and passed as parameters when creating the `AIAgent`:

```
providers_allowed  <- from provider_routing.only
providers_ignored  <- from provider_routing.ignore
providers_order    <- from provider_routing.order
provider_sort      <- from provider_routing.sort
provider_require_parameters <- from provider_routing.require_parameters
provider_data_collection    <- from provider_routing.data_collection
```

You can combine multiple options. For example, sort by price but exclude certain providers and require parameter support:

```yaml
provider_routing:
  sort: "price"
  ignore: ["Together"]
  require_parameters: true
  data_collection: "deny"
```

### Default Behavior

When no `provider_routing` section is configured, OpenRouter uses its own default routing logic, which generally balances cost and availability automatically.

### Provider Routing vs Fallback Models

Provider routing controls which **sub-providers within OpenRouter** handle your requests. For automatic failover to an entirely different provider when your primary model fails, see [Fallback Providers](./fallback-providers.md).
