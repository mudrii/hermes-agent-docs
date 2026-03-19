# Configuration

This document covers the complete `config.yaml` schema, environment variables (grouped by category), the `~/.hermes/` directory structure, and configuration precedence rules.

---

## Directory Structure

All Hermes configuration and state lives in `~/.hermes/` (or the directory set by `HERMES_HOME`):

```text
~/.hermes/
├── config.yaml     # Settings (model, terminal, TTS, compression, etc.)
├── .env            # API keys and secrets (0600 permissions)
├── auth.json       # OAuth provider credentials (Nous Portal, Codex, Copilot)
├── SOUL.md         # Primary agent identity (slot #1 in system prompt)
├── memories/       # Persistent memory (MEMORY.md, USER.md)
├── skills/         # Agent-created skills (managed via skill_manage tool)
├── cron/           # Scheduled jobs
├── sessions/       # Gateway sessions
└── logs/           # Logs (errors.log, gateway.log — secrets auto-redacted)
```

---

## Configuration Precedence

Settings are resolved in this order (highest priority first):

1. **CLI arguments** — e.g., `hermes chat --model anthropic/claude-sonnet-4` (per-invocation override)
2. **`~/.hermes/config.yaml`** — primary config file for all non-secret settings
3. **`~/.hermes/.env`** — required for secrets (API keys, bot tokens, passwords); fallback for other env vars
4. **Built-in defaults** — hardcoded safe defaults

**Rule of thumb:** Secrets (API keys, bot tokens, passwords) go in `.env`. Everything else (model, terminal backend, compression settings, memory limits) goes in `config.yaml`. When both are set, `config.yaml` wins for non-secret settings.

For provider selection specifically:

1. Explicit `--provider` CLI argument
2. `model.provider` in `config.yaml`
3. `HERMES_INFERENCE_PROVIDER` environment variable
4. Auto-detection (`auto`)

---

## Managing Configuration

```bash
hermes config show                      # View current configuration
hermes config edit                      # Open config.yaml in $EDITOR
hermes config set model anthropic/claude-opus-4   # Set a key in config.yaml
hermes config set OPENROUTER_API_KEY sk-or-...    # Set an API key (saves to .env)
hermes config set terminal.backend docker         # Set a nested key
hermes config path                      # Print config.yaml path
hermes config env-path                  # Print .env path
hermes config check                     # Check for missing options
hermes config migrate                   # Interactively add missing options
```

`hermes config set` automatically routes API keys to `.env` and everything else to `config.yaml`.

---

## Complete `config.yaml` Schema

### Model

The model section controls which inference provider and model Hermes uses by default.

```yaml
# String form (simple):
model: "anthropic/claude-opus-4.6"

# Dict form (full control):
model:
  provider: "anthropic"         # Provider ID (auto | openrouter | nous | anthropic | etc.)
  default: "claude-sonnet-4-6"  # Model name
  base_url: ""                  # Optional: custom endpoint base URL
  api_key: ""                   # Optional: API key for this endpoint
  api_mode: ""                  # Optional: chat_completions | codex_responses | anthropic_messages
```

`config.yaml` `model.provider` takes priority over the `HERMES_INFERENCE_PROVIDER` environment variable. This prevents a stale shell export from silently overriding the endpoint a user last selected in `hermes model`.

---

### Agent Behavior

```yaml
agent:
  max_turns: 90             # Max tool-calling iterations per conversation turn. Default: 90.
  reasoning_effort: ""      # "" = medium (default). Options: xhigh, high, medium, low, minimal, none.
  system_prompt: ""         # Custom system prompt (appended to SOUL.md)
  prefill_messages_file: "" # Path to JSON file of ephemeral prefill messages
  verbose: false            # Verbose agent output
```

**Budget pressure:** When the agent approaches the iteration limit, Hermes injects warnings into tool results:

| Threshold | Message |
|-----------|---------|
| 70% (63/90) | `[BUDGET: 63/90. 27 iterations left. Start consolidating.]` |
| 90% (81/90) | `[BUDGET WARNING: 81/90. Only 9 left. Respond NOW.]` |

Warnings are injected as `_budget_warning` fields in tool results to preserve prompt caching.

**Reasoning effort** can be changed at runtime with `/reasoning [level|show|hide]`.

---

### Terminal Backend

```yaml
terminal:
  backend: local            # local | docker | ssh | singularity | modal | daytona
  cwd: "."                  # Working directory ("." = resolved to os.getcwd() at runtime)
  timeout: 60               # Command timeout in seconds. Default: 60.
  lifetime_seconds: 300     # Max lifetime for terminal sessions. Default: 300.

  # Docker-specific
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: false  # SECURITY: off by default. Mount launch cwd into /workspace.
  docker_forward_env: []    # Env var names to explicitly forward into Docker terminal sessions
  docker_volumes: []        # Additional host:container volume mounts (e.g., "/host/path:/container/path")

  # Container resource limits (docker, singularity, modal, daytona)
  container_cpu: 1          # CPU cores. Default: 1.
  container_memory: 5120    # Memory in MB. Default: 5120 (5GB).
  container_disk: 51200     # Disk in MB. Default: 51200 (50GB).
  container_persistent: true  # Persist container filesystem across sessions. Default: true.

  # Persistent shell — keep a long-lived bash process across commands
  persistent_shell: true    # Enabled by default for SSH. Default: true for non-local backends.

  # Singularity
  singularity_image: "docker://python:3.11"

  # Modal
  modal_image: "python:3.11"

  # Daytona
  daytona_image: "nikolaik/python-nodejs:python3.11-nodejs20"
```

**Persistent shell precedence:**

| Level | Variable | Default |
|-------|----------|---------|
| Config | `terminal.persistent_shell` | `true` |
| SSH override | `TERMINAL_SSH_PERSISTENT` | follows config |
| Local override | `TERMINAL_LOCAL_PERSISTENT` | `false` |

Per-backend environment variables take highest precedence.

**Docker volume mounts:** Each entry uses Docker `-v` syntax: `host_path:container_path[:options]`. `:ro` for read-only.

**Docker credential forwarding:** Variables listed in `docker_forward_env` are resolved from the current shell first, then from `~/.hermes/.env` as fallback.

---

### Provider Routing (OpenRouter)

```yaml
provider_routing:
  sort: "price"              # "price" (default), "throughput", or "latency"
  only: []                   # Whitelist of provider names to allow
  ignore: []                 # Blacklist of provider names to skip
  order: []                  # Explicit provider priority order
  require_parameters: false  # Only use providers supporting all request params
  data_collection: null      # "allow" or "deny"
```

Only applies when using OpenRouter. See [providers.md](./providers.md) for details.

---

### Fallback Model

```yaml
fallback_model:
  provider: openrouter                # required
  model: anthropic/claude-sonnet-4    # required
  # base_url: http://localhost:8000/v1  # optional, for custom endpoints
  # api_key_env: MY_CUSTOM_KEY          # optional, env var name for that endpoint's API key
```

Configured exclusively through `config.yaml` — no environment variables. Activates at most once per session. See [providers.md](./providers.md) for full details.

---

### Smart Model Routing

```yaml
smart_model_routing:
  enabled: false            # Default: false
  max_simple_chars: 160     # Max characters for a "simple" turn
  max_simple_words: 28      # Max words for a "simple" turn
  cheap_model:
    provider: openrouter
    model: google/gemini-2.5-flash
    # base_url: http://localhost:8000/v1   # optional
    # api_key_env: MY_CUSTOM_KEY           # optional
```

Routes short, simple turns to a cheap model. Falls back to the primary model if routing fails.

---

### Context Compression

```yaml
compression:
  enabled: true                                     # Toggle compression on/off
  threshold: 0.50                                   # Compress at this % of context limit. Default: 0.50.
  summary_model: "google/gemini-3-flash-preview"    # Model for summarization
  summary_provider: "auto"                          # Provider: "auto", "openrouter", "nous", "codex", "main", etc.
  summary_base_url: null                            # Custom OpenAI-compatible endpoint (overrides provider)
```

All compression settings are in `config.yaml` only — no environment variables.

**Common setups:**

```yaml
# Default (auto-detect, no configuration needed):
compression:
  enabled: true
  threshold: 0.50

# Force a specific provider:
compression:
  summary_provider: nous
  summary_model: gemini-3-flash

# Custom endpoint:
compression:
  summary_model: glm-4.7
  summary_base_url: https://api.z.ai/api/coding/paas/v4
```

**How the three knobs interact:**

| `summary_provider` | `summary_base_url` | Result |
|--------------------|--------------------|--------|
| `auto` (default) | not set | Auto-detect best available provider |
| `nous` / `openrouter` / etc. | not set | Force that provider, use its auth |
| any | set | Use the custom endpoint directly (provider ignored) |

---

### Auxiliary Models

Side-task models for vision, web extraction, compression, session search, skills hub, MCP helpers, and memory flush.

```yaml
auxiliary:
  # Image analysis (vision_analyze tool + browser screenshots)
  vision:
    provider: "auto"           # auto | openrouter | nous | codex | main | anthropic
    model: ""                  # e.g. "openai/gpt-4o", "google/gemini-2.5-flash"
    base_url: ""               # Direct OpenAI-compatible endpoint (overrides provider)
    api_key: ""                # API key for base_url (falls back to OPENAI_API_KEY)

  # Web page summarization + browser page text extraction
  web_extract:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""

  # Dangerous command approval classifier
  approval:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""

  # Additional tasks also support provider/model/base_url/api_key:
  # session_search, skills_hub, mcp, flush_memories
```

When `base_url` is set, it takes precedence over `provider`. Authentication uses the configured `api_key`, falling back to `OPENAI_API_KEY`. `OPENROUTER_API_KEY` is never reused for auxiliary task custom endpoints.

**Environment variable equivalents (legacy):**

```bash
AUXILIARY_VISION_PROVIDER=openrouter
AUXILIARY_VISION_MODEL=openai/gpt-4o
AUXILIARY_VISION_BASE_URL=http://localhost:1234/v1
AUXILIARY_VISION_API_KEY=local-key
AUXILIARY_WEB_EXTRACT_PROVIDER=openrouter
AUXILIARY_WEB_EXTRACT_MODEL=google/gemini-2.5-flash
AUXILIARY_WEB_EXTRACT_BASE_URL=
AUXILIARY_WEB_EXTRACT_API_KEY=
```

`config.yaml` is the preferred method — it supports all options including `base_url` and `api_key`.

---

### Memory

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200     # ~800 tokens
  user_char_limit: 1375       # ~500 tokens
```

---

### Delegation

```yaml
delegation:
  max_iterations: 50           # Max iterations per subagent. Default: 50.
  default_toolsets:             # Toolsets available to subagents
    - terminal
    - file
    - web
  model: ""                    # Subagent model override (empty = inherit parent)
  provider: ""                 # Subagent provider override (empty = inherit parent)
  base_url: ""                 # Direct OpenAI-compatible endpoint for subagents
  api_key: ""                  # API key for base_url (falls back to OPENAI_API_KEY)
```

**Precedence:** `delegation.base_url` → `delegation.provider` → parent provider (inherited). `delegation.model` → parent model (inherited). Setting `model` without `provider` changes only the model name while keeping the parent's credentials.

---

### Display

```yaml
display:
  tool_progress: all          # off | new | all | verbose
  skin: default               # Built-in or custom CLI skin
  theme_mode: auto            # auto | light | dark
  personality: "kawaii"       # Legacy cosmetic field
  compact: false              # Compact output mode (less whitespace)
  resume_display: full        # full (show previous messages on resume) | minimal (one-liner only)
  bell_on_complete: false     # Play terminal bell when agent finishes
  show_reasoning: false       # Show model reasoning/thinking above each response
  streaming: false            # Stream tokens to terminal as they arrive
  background_process_notifications: all  # all | result | error | off (gateway only)
```

**Tool progress levels:**

| Value | Description |
|-------|-------------|
| `off` | Silent — just the final response |
| `new` | Tool indicator only when the tool changes |
| `all` | Every tool call with a short preview (default) |
| `verbose` | Full args, results, and debug logs |

**Theme mode:**

| Value | Description |
|-------|-------------|
| `auto` | Detects terminal background automatically, falls back to dark |
| `light` | Forces light-mode skin colors |
| `dark` | Forces dark-mode skin colors |

---

### TTS (Text-to-Speech)

```yaml
tts:
  provider: "edge"              # "edge" | "elevenlabs" | "openai" | "neutts"
  edge:
    voice: "en-US-AriaNeural"   # 322 voices, 74 languages
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"
    model_id: "eleven_multilingual_v2"
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"              # alloy | echo | fable | onyx | nova | shimmer
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
```

Controls both the `text_to_speech` tool and spoken replies in `/voice tts` mode.

---

### STT (Speech-to-Text)

```yaml
stt:
  provider: "local"            # "local" | "groq" | "openai"
  local:
    model: "base"              # tiny | base | small | medium | large-v3
  openai:
    model: "whisper-1"         # whisper-1 | gpt-4o-mini-transcribe | gpt-4o-transcribe
```

Provider fallback order: `local` → `groq` → `openai`.

- `local` uses `faster-whisper` running locally. Install with `pip install faster-whisper`.
- `groq` reads `GROQ_API_KEY`.
- `openai` reads `VOICE_TOOLS_OPENAI_KEY`.

---

### Voice Mode (CLI)

```yaml
voice:
  record_key: "ctrl+b"         # Push-to-talk key inside the CLI
  max_recording_seconds: 120    # Hard stop for long recordings
  auto_tts: false               # Enable spoken replies automatically when /voice on
  silence_threshold: 200        # RMS threshold for speech detection
  silence_duration: 3.0         # Seconds of silence before auto-stop
```

---

### Streaming

**CLI streaming:**

```yaml
display:
  streaming: true               # Stream tokens to terminal in real-time
  show_reasoning: true          # Also stream reasoning/thinking tokens (optional)
```

**Gateway streaming (Telegram, Discord, Slack):**

```yaml
streaming:
  enabled: true                 # Enable progressive message editing
  edit_interval: 0.3            # Seconds between message edits
  buffer_threshold: 40          # Characters before forcing an edit flush
  cursor: " ▉"                  # Cursor shown during streaming
```

Platforms that don't support message editing (Signal, Email) skip streaming and deliver the final response normally.

---

### Worktree Isolation

```yaml
worktree: true    # Always create a worktree (same as hermes -w). Default: false.
```

When enabled, each CLI session creates a fresh worktree under `.worktrees/` with its own branch. List gitignored files to copy into worktrees via `.worktreeinclude` in the repo root.

---

### Checkpoints

```yaml
checkpoints:
  enabled: false                # Enable automatic checkpoints. Also: hermes --checkpoints.
  max_snapshots: 50             # Max checkpoints to keep per directory.
```

---

### Smart Approvals

```yaml
approval_mode: ask              # ask | smart | off
```

| Mode | Behavior |
|------|----------|
| `ask` (default) | Prompt before executing any flagged command. Interactive dialog in CLI; pending approval queue in messaging. |
| `smart` | Use an auxiliary LLM to assess danger. Low-risk commands auto-approved with session-level persistence. |
| `off` | Skip all approval checks. Equivalent to `HERMES_YOLO_MODE=true`. Use with caution. |

---

### Session Reset (Gateway)

```yaml
session_reset:
  mode: "both"                  # both | daily | idle | none
  at_hour: 4                    # Hour for daily reset (0-23, local time). Default: 4 (4am).
  idle_minutes: 1440            # Minutes of inactivity before reset. Default: 1440 (24h).
```

---

### Group Session Isolation

```yaml
group_sessions_per_user: true   # true = per-user isolation in groups/channels (default)
                                 # false = one shared session per chat
```

Direct messages are unaffected. With `true`, each participant also gets their own session inside threads.

---

### Unauthorized DM Behavior

```yaml
unauthorized_dm_behavior: pair  # pair (default) | ignore

# Per-platform override:
whatsapp:
  unauthorized_dm_behavior: ignore
```

- `pair` — denies access but replies with a one-time pairing code in DMs.
- `ignore` — silently drops unauthorized DMs.

---

### Quick Commands

Define custom slash commands that run shell commands without invoking the LLM. 30-second timeout. Priority over skill commands.

```yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  disk:
    type: exec
    command: df -h /
  gpu:
    type: exec
    command: nvidia-smi --query-gpu=name,utilization.gpu,memory.used,memory.total --format=csv,noheader
```

Available in CLI and all messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant).

---

### Human Delay

Simulate human-like response pacing in messaging platforms.

```yaml
human_delay:
  mode: "off"                   # off | natural | custom
  min_ms: 500                   # Minimum delay in ms (custom mode)
  max_ms: 2000                  # Maximum delay in ms (custom mode)
```

---

### Code Execution

```yaml
code_execution:
  timeout: 300                  # Max execution time in seconds. Default: 300 (5 min).
  max_tool_calls: 50            # Max tool calls within code execution. Default: 50.
```

---

### Browser

```yaml
browser:
  inactivity_timeout: 120       # Seconds before auto-closing idle sessions. Default: 120.
  record_sessions: false        # Auto-record browser sessions as WebM to ~/.hermes/browser_recordings/.
```

---

### Website Blocklist

```yaml
website_blocklist:
  enabled: false                # Default: false
  domains:                      # Domain patterns (supports wildcards)
    - "*.internal.company.com"
    - "admin.example.com"
    - "*.local"
  shared_files:                 # External files with additional rules (one domain per line)
    - "/etc/hermes/blocked-sites.txt"
```

Applies to `web_search`, `web_extract`, `browser_navigate`, and any tool that accesses URLs. Policy is cached for 30 seconds.

---

### Privacy

```yaml
privacy:
  redact_pii: false             # Strip PII from LLM context (gateway only). Default: false.
```

When `true`, the gateway hashes phone numbers, user IDs, and chat IDs before sending them to the LLM. Hashes are deterministic — the same user always maps to the same hash. Applies to WhatsApp, Signal, and Telegram. Discord and Slack are excluded because their mention systems require real IDs.

---

### Clarify

```yaml
clarify:
  timeout: 120                  # Seconds to wait for user clarification response. Default: 120.
```

---

### Context Files (SOUL.md, AGENTS.md)

| File | Purpose | Scope |
|------|---------|-------|
| `SOUL.md` | Primary agent identity — occupies slot #1 in system prompt | `~/.hermes/SOUL.md` or `$HERMES_HOME/SOUL.md` |
| `AGENTS.md` | Project-specific instructions, coding conventions | Working directory / project tree (hierarchical) |
| `.cursorrules` | Cursor IDE rules (also detected) | Working directory |
| `.cursor/rules/*.mdc` | Cursor rule files (also detected) | Working directory |

All loaded context files are capped at 20,000 characters with smart truncation.

---

### Discord-Specific Config Keys

```yaml
discord:
  require_mention: true         # Require @mention before responding in server channels
  free_response_channels: []    # Channel IDs where @mention is not required
  auto_thread: false            # Auto-thread long replies when supported
```

These keys in `config.yaml` map to `DISCORD_REQUIRE_MENTION`, `DISCORD_FREE_RESPONSE_CHANNELS`, and `DISCORD_AUTO_THREAD` environment variables (env vars take precedence).

---

### Platform-Specific `unauthorized_dm_behavior`

Platform sections can override the global `unauthorized_dm_behavior`:

```yaml
unauthorized_dm_behavior: pair  # global default

telegram:
  unauthorized_dm_behavior: pair

whatsapp:
  unauthorized_dm_behavior: ignore

signal:
  unauthorized_dm_behavior: ignore
```

---

## MCP Section

```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
  my-server:
    command: "python"
    args: ["-m", "my_mcp_server"]
    env:
      MY_SERVER_VAR: "value"
```

Use `/reload-mcp` inside a session to reload MCP servers without restarting. See MCP config reference for full schema including `tools.include`, `tools.exclude`, `tools.resources`, `tools.prompts`, and `enabled` options.

---

## Honcho Section

```yaml
honcho:
  # Configured via: hermes honcho setup
  # See: hermes honcho status
```

Honcho cross-session memory is managed interactively via `hermes honcho setup`. The `HONCHO_API_KEY` environment variable enables it automatically.

---

## All Environment Variables

All variables go in `~/.hermes/.env` or the shell. Use `hermes config set VAR value` to save to the right file.

### LLM Providers

| Variable | Description |
|----------|-------------|
| `OPENROUTER_API_KEY` | OpenRouter API key (recommended for flexibility) |
| `OPENROUTER_BASE_URL` | Override the OpenRouter-compatible base URL |
| `AI_GATEWAY_API_KEY` | Vercel AI Gateway API key |
| `AI_GATEWAY_BASE_URL` | Override AI Gateway base URL. Default: `https://ai-gateway.vercel.sh/v1`. |
| `OPENAI_API_KEY` | API key for custom OpenAI-compatible endpoints |
| `OPENAI_BASE_URL` | Base URL for custom endpoint (vLLM, SGLang, etc.) |
| `COPILOT_GITHUB_TOKEN` | GitHub token for Copilot API (first priority; `gho_*`, `github_pat_*`, or `ghu_*`; classic PATs `ghp_*` are not supported) |
| `GH_TOKEN` | GitHub token (second priority for Copilot; also used by `gh` CLI) |
| `GITHUB_TOKEN` | GitHub token (third priority for Copilot) |
| `HERMES_COPILOT_ACP_COMMAND` | Override Copilot ACP CLI binary path. Default: `copilot`. |
| `COPILOT_CLI_PATH` | Alias for `HERMES_COPILOT_ACP_COMMAND` |
| `HERMES_COPILOT_ACP_ARGS` | Override Copilot ACP arguments. Default: `--acp --stdio`. |
| `COPILOT_ACP_BASE_URL` | Override Copilot ACP base URL |
| `GLM_API_KEY` | Z.AI / ZhipuAI GLM API key |
| `ZAI_API_KEY` | Alias for `GLM_API_KEY` |
| `Z_AI_API_KEY` | Alias for `GLM_API_KEY` |
| `GLM_BASE_URL` | Override Z.AI base URL. Default: `https://api.z.ai/api/paas/v4`. |
| `KIMI_API_KEY` | Kimi / Moonshot AI API key |
| `KIMI_BASE_URL` | Override Kimi base URL. Default: `https://api.moonshot.ai/v1`. |
| `MINIMAX_API_KEY` | MiniMax API key — global endpoint |
| `MINIMAX_BASE_URL` | Override MiniMax base URL. Default: `https://api.minimax.io/v1`. |
| `MINIMAX_CN_API_KEY` | MiniMax API key — China endpoint |
| `MINIMAX_CN_BASE_URL` | Override MiniMax China base URL. Default: `https://api.minimaxi.com/v1`. |
| `KILOCODE_API_KEY` | Kilo Code API key |
| `KILOCODE_BASE_URL` | Override Kilo Code base URL. Default: `https://api.kilo.ai/api/gateway`. |
| `ANTHROPIC_API_KEY` | Anthropic Console API key |
| `ANTHROPIC_TOKEN` | Manual or legacy Anthropic OAuth/setup-token override |
| `DASHSCOPE_API_KEY` | Alibaba Cloud DashScope API key (Qwen models) |
| `DASHSCOPE_BASE_URL` | Custom DashScope base URL |
| `DEEPSEEK_API_KEY` | DeepSeek API key |
| `DEEPSEEK_BASE_URL` | Custom DeepSeek API base URL |
| `OPENCODE_ZEN_API_KEY` | OpenCode Zen API key (pay-as-you-go) |
| `OPENCODE_ZEN_BASE_URL` | Override OpenCode Zen base URL |
| `OPENCODE_GO_API_KEY` | OpenCode Go API key ($10/month) |
| `OPENCODE_GO_BASE_URL` | Override OpenCode Go base URL |
| `CLAUDE_CODE_OAUTH_TOKEN` | Explicit Claude Code token override |
| `HERMES_MODEL` | Preferred model name. Checked before `LLM_MODEL`. |
| `LLM_MODEL` | Default model name. Fallback when not set in `config.yaml`. |
| `VOICE_TOOLS_OPENAI_KEY` | Preferred OpenAI key for speech-to-text and text-to-speech |
| `HERMES_LOCAL_STT_COMMAND` | Optional local STT command template. Supports `{input_path}`, `{output_dir}`, `{language}`, `{model}`. |
| `HERMES_LOCAL_STT_LANGUAGE` | Default language for local STT. Default: `en`. |
| `HERMES_HOME` | Override Hermes config directory. Default: `~/.hermes`. |

### Provider Auth (OAuth)

| Variable | Description |
|----------|-------------|
| `HERMES_INFERENCE_PROVIDER` | Override provider selection: `auto`, `openrouter`, `nous`, `openai-codex`, `copilot`, `anthropic`, `zai`, `kimi-coding`, `minimax`, `minimax-cn`, `kilocode`, `alibaba`. Default: `auto`. |
| `HERMES_PORTAL_BASE_URL` | Override Nous Portal URL (for development/testing) |
| `NOUS_INFERENCE_BASE_URL` | Override Nous inference API URL |
| `HERMES_NOUS_MIN_KEY_TTL_SECONDS` | Min agent key TTL before re-mint. Default: `1800` (30 min). |
| `HERMES_NOUS_TIMEOUT_SECONDS` | HTTP timeout for Nous credential/token flows |
| `HERMES_DUMP_REQUESTS` | Dump API request payloads to log files. Values: `true`/`false`. |
| `HERMES_PREFILL_MESSAGES_FILE` | Path to a JSON file of ephemeral prefill messages |
| `HERMES_TIMEZONE` | IANA timezone override. Example: `America/New_York`. |

### Tool APIs

| Variable | Description |
|----------|-------------|
| `PARALLEL_API_KEY` | AI-native web search (parallel.ai) |
| `FIRECRAWL_API_KEY` | Web scraping (firecrawl.dev) |
| `FIRECRAWL_API_URL` | Custom Firecrawl API endpoint for self-hosted instances |
| `TAVILY_API_KEY` | Tavily API key for AI-native web search, extract, and crawl |
| `BROWSERBASE_API_KEY` | Browser automation (browserbase.com) |
| `BROWSERBASE_PROJECT_ID` | Browserbase project ID |
| `BROWSER_USE_API_KEY` | Browser Use cloud API key (browser-use.com) |
| `BROWSER_CDP_URL` | Chrome DevTools Protocol URL for local browser (set via `/browser connect`, e.g. `ws://localhost:9222`) |
| `BROWSER_INACTIVITY_TIMEOUT` | Browser session inactivity timeout in seconds |
| `FAL_KEY` | Image generation (fal.ai) |
| `GROQ_API_KEY` | Groq Whisper STT API key |
| `ELEVENLABS_API_KEY` | ElevenLabs premium TTS voices |
| `STT_GROQ_MODEL` | Override the Groq STT model. Default: `whisper-large-v3-turbo`. |
| `GROQ_BASE_URL` | Override the Groq OpenAI-compatible STT endpoint |
| `STT_OPENAI_MODEL` | Override the OpenAI STT model. Default: `whisper-1`. |
| `STT_OPENAI_BASE_URL` | Override the OpenAI-compatible STT endpoint |
| `GITHUB_TOKEN` | GitHub token for Skills Hub (higher API rate limits, skill publish) |
| `HONCHO_API_KEY` | Cross-session user modeling (honcho.dev) |
| `TINKER_API_KEY` | RL training (tinker-console.thinkingmachines.ai) |
| `WANDB_API_KEY` | RL training metrics (wandb.ai) |
| `DAYTONA_API_KEY` | Daytona cloud sandboxes (daytona.io) |

### Terminal Backend

| Variable | Description |
|----------|-------------|
| `TERMINAL_ENV` | Backend: `local`, `docker`, `ssh`, `singularity`, `modal`, `daytona` |
| `TERMINAL_DOCKER_IMAGE` | Docker image. Default: `python:3.11`. |
| `TERMINAL_DOCKER_FORWARD_ENV` | JSON array of env var names to forward into Docker terminal sessions |
| `TERMINAL_DOCKER_VOLUMES` | Additional Docker volume mounts (comma-separated `host:container` pairs) |
| `TERMINAL_DOCKER_MOUNT_CWD_TO_WORKSPACE` | Mount the launch cwd into Docker `/workspace`. Values: `true`/`false`. Default: `false`. |
| `TERMINAL_SINGULARITY_IMAGE` | Singularity image or `.sif` path |
| `TERMINAL_MODAL_IMAGE` | Modal container image |
| `TERMINAL_DAYTONA_IMAGE` | Daytona sandbox image |
| `TERMINAL_TIMEOUT` | Command timeout in seconds |
| `TERMINAL_LIFETIME_SECONDS` | Max lifetime for terminal sessions in seconds |
| `TERMINAL_CWD` | Working directory for all terminal sessions |
| `SUDO_PASSWORD` | Enable sudo without interactive prompt |

### SSH Backend

| Variable | Description |
|----------|-------------|
| `TERMINAL_SSH_HOST` | Remote server hostname |
| `TERMINAL_SSH_USER` | SSH username |
| `TERMINAL_SSH_PORT` | SSH port. Default: `22`. |
| `TERMINAL_SSH_KEY` | Path to private key |
| `TERMINAL_SSH_PERSISTENT` | Override persistent shell for SSH. Default: follows `TERMINAL_PERSISTENT_SHELL`. |

### Container Resources (Docker, Singularity, Modal, Daytona)

| Variable | Description |
|----------|-------------|
| `TERMINAL_CONTAINER_CPU` | CPU cores. Default: `1`. |
| `TERMINAL_CONTAINER_MEMORY` | Memory in MB. Default: `5120`. |
| `TERMINAL_CONTAINER_DISK` | Disk in MB. Default: `51200`. |
| `TERMINAL_CONTAINER_PERSISTENT` | Persist container filesystem across sessions. Default: `true`. |
| `TERMINAL_SANDBOX_DIR` | Host directory for workspaces and overlays. Default: `~/.hermes/sandboxes/`. |

### Persistent Shell

| Variable | Description |
|----------|-------------|
| `TERMINAL_PERSISTENT_SHELL` | Enable persistent shell for non-local backends. Default: `true`. Also settable via `terminal.persistent_shell` in `config.yaml`. |
| `TERMINAL_LOCAL_PERSISTENT` | Enable persistent shell for local backend. Default: `false`. |
| `TERMINAL_SSH_PERSISTENT` | Override persistent shell for SSH backend. Default: follows `TERMINAL_PERSISTENT_SHELL`. |

### Messaging

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | Telegram bot token (from @BotFather) |
| `TELEGRAM_ALLOWED_USERS` | Comma-separated user IDs |
| `TELEGRAM_HOME_CHANNEL` | Default Telegram chat/channel for cron delivery |
| `TELEGRAM_HOME_CHANNEL_NAME` | Display name for the Telegram home channel |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `DISCORD_ALLOWED_USERS` | Comma-separated Discord user IDs |
| `DISCORD_HOME_CHANNEL` | Default Discord channel for cron delivery |
| `DISCORD_HOME_CHANNEL_NAME` | Display name for the Discord home channel |
| `DISCORD_REQUIRE_MENTION` | Require @mention before responding in server channels |
| `DISCORD_FREE_RESPONSE_CHANNELS` | Comma-separated channel IDs where mention is not required |
| `DISCORD_AUTO_THREAD` | Auto-thread long replies when supported |
| `SLACK_BOT_TOKEN` | Slack bot token (`xoxb-...`) |
| `SLACK_APP_TOKEN` | Slack app-level token (`xapp-...`, required for Socket Mode) |
| `SLACK_ALLOWED_USERS` | Comma-separated Slack user IDs |
| `SLACK_HOME_CHANNEL` | Default Slack channel for cron delivery |
| `SLACK_HOME_CHANNEL_NAME` | Display name for the Slack home channel |
| `WHATSAPP_ENABLED` | Enable the WhatsApp bridge. Values: `true`/`false`. |
| `WHATSAPP_MODE` | `bot` (separate number) or `self-chat` (message yourself) |
| `WHATSAPP_ALLOWED_USERS` | Comma-separated phone numbers (with country code, no `+`) |
| `SIGNAL_HTTP_URL` | signal-cli daemon HTTP endpoint (e.g. `http://127.0.0.1:8080`) |
| `SIGNAL_ACCOUNT` | Bot phone number in E.164 format |
| `SIGNAL_ALLOWED_USERS` | Comma-separated E.164 phone numbers or UUIDs |
| `SIGNAL_GROUP_ALLOWED_USERS` | Comma-separated group IDs, or `*` for all groups |
| `SIGNAL_HOME_CHANNEL_NAME` | Display name for the Signal home channel |
| `SIGNAL_IGNORE_STORIES` | Ignore Signal stories/status updates |
| `SIGNAL_ALLOW_ALL_USERS` | Allow all Signal users without an allowlist |
| `TWILIO_ACCOUNT_SID` | Twilio Account SID |
| `TWILIO_AUTH_TOKEN` | Twilio Auth Token |
| `TWILIO_PHONE_NUMBER` | Twilio phone number in E.164 format |
| `SMS_WEBHOOK_PORT` | Webhook listener port for inbound SMS. Default: `8080`. |
| `SMS_ALLOWED_USERS` | Comma-separated E.164 phone numbers allowed to chat |
| `SMS_ALLOW_ALL_USERS` | Allow all SMS senders without an allowlist |
| `SMS_HOME_CHANNEL` | Phone number for cron job/notification delivery |
| `SMS_HOME_CHANNEL_NAME` | Display name for the SMS home channel |
| `EMAIL_ADDRESS` | Email address for the Email gateway adapter |
| `EMAIL_PASSWORD` | Password or app password for the email account |
| `EMAIL_IMAP_HOST` | IMAP hostname |
| `EMAIL_IMAP_PORT` | IMAP port |
| `EMAIL_SMTP_HOST` | SMTP hostname |
| `EMAIL_SMTP_PORT` | SMTP port |
| `EMAIL_ALLOWED_USERS` | Comma-separated email addresses allowed to message the bot |
| `EMAIL_HOME_ADDRESS` | Default recipient for proactive email delivery |
| `EMAIL_HOME_ADDRESS_NAME` | Display name for the email home target |
| `EMAIL_POLL_INTERVAL` | Email polling interval in seconds |
| `EMAIL_ALLOW_ALL_USERS` | Allow all inbound email senders |
| `DINGTALK_CLIENT_ID` | DingTalk bot AppKey |
| `DINGTALK_CLIENT_SECRET` | DingTalk bot AppSecret |
| `DINGTALK_ALLOWED_USERS` | Comma-separated DingTalk user IDs |
| `MATTERMOST_URL` | Mattermost server URL (e.g. `https://mm.example.com`) |
| `MATTERMOST_TOKEN` | Bot token or personal access token |
| `MATTERMOST_ALLOWED_USERS` | Comma-separated Mattermost user IDs |
| `MATTERMOST_HOME_CHANNEL` | Channel ID for proactive delivery |
| `MATTERMOST_REPLY_MODE` | `thread` (threaded) or `off` (flat messages, default) |
| `MATRIX_HOMESERVER` | Matrix homeserver URL (e.g. `https://matrix.org`) |
| `MATRIX_ACCESS_TOKEN` | Matrix access token for bot authentication |
| `MATRIX_USER_ID` | Matrix user ID (e.g. `@hermes:matrix.org`) |
| `MATRIX_PASSWORD` | Matrix password (alternative to access token) |
| `MATRIX_ALLOWED_USERS` | Comma-separated Matrix user IDs |
| `MATRIX_HOME_ROOM` | Room ID for proactive delivery |
| `MATRIX_ENCRYPTION` | Enable end-to-end encryption. Values: `true`/`false`. Default: `false`. |
| `HASS_TOKEN` | Home Assistant Long-Lived Access Token |
| `HASS_URL` | Home Assistant URL. Default: `http://homeassistant.local:8123`. |
| `API_SERVER_ENABLED` | Enable the OpenAI-compatible API server. Values: `true`/`false`. |
| `API_SERVER_KEY` | Bearer token for API server authentication |
| `API_SERVER_PORT` | Port for the API server. Default: `8642`. |
| `API_SERVER_HOST` | Host/bind address. Default: `127.0.0.1`. |
| `MESSAGING_CWD` | Working directory for terminal commands in messaging mode. Default: `~`. |
| `GATEWAY_ALLOWED_USERS` | Comma-separated user IDs allowed across all platforms |
| `GATEWAY_ALLOW_ALL_USERS` | Allow all users without allowlists. Values: `true`/`false`. Default: `false`. |

### Agent Behavior

| Variable | Description |
|----------|-------------|
| `HERMES_MAX_ITERATIONS` | Max tool-calling iterations per conversation. Default: `90`. |
| `HERMES_TOOL_PROGRESS` | Deprecated: use `display.tool_progress` in `config.yaml`. |
| `HERMES_TOOL_PROGRESS_MODE` | Deprecated: use `display.tool_progress` in `config.yaml`. |
| `HERMES_HUMAN_DELAY_MODE` | Response pacing: `off`/`natural`/`custom`. |
| `HERMES_HUMAN_DELAY_MIN_MS` | Custom delay range minimum (ms). |
| `HERMES_HUMAN_DELAY_MAX_MS` | Custom delay range maximum (ms). |
| `HERMES_QUIET` | Suppress non-essential output. Values: `true`/`false`. |
| `HERMES_API_TIMEOUT` | LLM API call timeout in seconds. Default: `900`. |
| `HERMES_EXEC_ASK` | Enable execution approval prompts in gateway mode. Values: `true`/`false`. |
| `HERMES_BACKGROUND_NOTIFICATIONS` | Background process notification mode in gateway: `all` (default), `result`, `error`, `off`. |
| `HERMES_EPHEMERAL_SYSTEM_PROMPT` | Ephemeral system prompt injected at API-call time (never persisted to sessions). |

### Auxiliary Task Overrides

| Variable | Description |
|----------|-------------|
| `AUXILIARY_VISION_PROVIDER` | Override provider for vision tasks |
| `AUXILIARY_VISION_MODEL` | Override model for vision tasks |
| `AUXILIARY_VISION_BASE_URL` | Direct OpenAI-compatible endpoint for vision tasks |
| `AUXILIARY_VISION_API_KEY` | API key paired with `AUXILIARY_VISION_BASE_URL` |
| `AUXILIARY_WEB_EXTRACT_PROVIDER` | Override provider for web extraction/summarization |
| `AUXILIARY_WEB_EXTRACT_MODEL` | Override model for web extraction/summarization |
| `AUXILIARY_WEB_EXTRACT_BASE_URL` | Direct OpenAI-compatible endpoint for web extraction |
| `AUXILIARY_WEB_EXTRACT_API_KEY` | API key paired with `AUXILIARY_WEB_EXTRACT_BASE_URL` |

For direct endpoint overrides, Hermes uses the task's configured API key or `OPENAI_API_KEY`. It does not reuse `OPENROUTER_API_KEY` for those custom endpoints.

### Session Settings

| Variable | Description |
|----------|-------------|
| `SESSION_IDLE_MINUTES` | Reset sessions after N minutes of inactivity. Default: `1440`. |
| `SESSION_RESET_HOUR` | Daily reset hour in 24h format. Default: `4` (4am). |

---

## Working Directory Defaults

| Context | Default |
|---------|---------|
| CLI (`hermes`) | Current directory where you run the command |
| Messaging gateway | Home directory `~` (override with `MESSAGING_CWD`) |
| Docker / Singularity / Modal / SSH | User's home directory inside the container or remote machine (override with `TERMINAL_CWD`) |

---

## Gateway Configuration File (Legacy)

The gateway can also be configured via `~/.hermes/gateway.json`. This is the legacy path. `config.yaml` keys always win when both files specify the same setting. The `gateway.json` format:

```json
{
  "platforms": {
    "telegram": {
      "enabled": true,
      "token": "...",
      "home_channel": {"platform": "telegram", "chat_id": "123456789", "name": "Home"}
    },
    "discord": {
      "enabled": true,
      "token": "..."
    }
  },
  "default_reset_policy": {
    "mode": "both",
    "at_hour": 4,
    "idle_minutes": 1440
  },
  "reset_triggers": ["/new", "/reset"],
  "quick_commands": {},
  "stt_enabled": true,
  "group_sessions_per_user": true,
  "always_log_local": true,
  "unauthorized_dm_behavior": "pair",
  "streaming": {
    "enabled": false,
    "edit_interval": 0.3,
    "buffer_threshold": 40,
    "cursor": " ▉"
  }
}
```

Prefer `config.yaml` for all new configuration. The gateway reads `config.yaml` keys (`session_reset`, `quick_commands`, `stt`, `group_sessions_per_user`, `streaming`, `reset_triggers`, `always_log_local`, `unauthorized_dm_behavior`, and per-platform sections) and bridges them into the gateway config on startup.
