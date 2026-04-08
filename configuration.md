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
└── logs/           # Logs (errors.log, gateway.log -- secrets auto-redacted)
```

---

## Configuration Precedence

Settings are resolved in this order (highest priority first):

1. **CLI arguments** -- e.g., `hermes chat --model anthropic/claude-sonnet-4` (per-invocation override)
2. **`~/.hermes/config.yaml`** -- primary config file for all non-secret settings
3. **`~/.hermes/.env`** -- required for secrets (API keys, bot tokens, passwords); fallback for other env vars
4. **Built-in defaults** -- hardcoded safe defaults

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
hermes config set model anthropic/claude-opus-4.6   # Set a key in config.yaml
hermes config set OPENROUTER_API_KEY sk-or-...      # Set an API key (saves to .env)
hermes config set terminal.backend docker           # Set a nested key
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

**v0.5.0 (PR #3112):** Root-level `provider` and `base_url` keys are now also read from `config.yaml` as a shorthand when you don't want to nest them under `model:`:

```yaml
# Root-level shorthand (v0.5.0):
provider: "openrouter"
base_url: "https://openrouter.ai/api/v1"
model: "anthropic/claude-opus-4.6"
```

When both root-level and `model.provider` / `model.base_url` are set, the `model:` block takes precedence.

`config.yaml` `model.provider` takes priority over the `HERMES_INFERENCE_PROVIDER` environment variable. This prevents a stale shell export from silently overriding the endpoint a user last selected in `hermes model`.

---

### Agent Behavior

```yaml
agent:
  max_turns: 90             # Max tool-calling iterations per conversation turn. Default: 90.
  reasoning_effort: ""      # "" = medium (default). Options: xhigh, high, medium, low, minimal, none.
  tool_use_enforcement: "auto"  # v0.5.0 (PR #3551). See below.
```

**`reasoning_effort`** (v0.8.0): Configurable through `config.yaml` only. The `HERMES_REASONING_EFFORT` environment variable was removed in v0.8.0 — set this in `config.yaml` or change it at runtime with `/reasoning [level]`.

**`tool_use_enforcement`** (v0.5.0 — PR #3551): Injects system-prompt guidance that tells the model to actually call tools instead of describing intended actions. Fixes GPT model reliability.

| Value | Behavior |
|-------|----------|
| `"auto"` (default) | Apply guidance only to `gpt` and `codex` model names |
| `true` | Force on for all models |
| `false` | Disable for all models |
| `["gpt", "codex", "gemini"]` | Apply to any model whose name contains one of the listed substrings |

**Budget pressure:** When the agent approaches the iteration limit, Hermes injects warnings into tool results:

| Threshold | Message |
|-----------|---------|
| 70% (63/90) | `[BUDGET: 63/90. 27 iterations left. Start consolidating.]` |
| 90% (81/90) | `[BUDGET WARNING: 81/90. Only 9 left. Respond NOW.]` |

**Reasoning effort** can be changed at runtime with `/reasoning [level|show|hide]`.

---

### Toolsets

```yaml
toolsets:
  - hermes-cli              # Default toolset. Others: terminal, web, file, skills, etc.
```

---

### Terminal Backend

```yaml
terminal:
  backend: local            # local | docker | ssh | singularity | modal | daytona
  cwd: "."                  # Working directory ("." = resolved to os.getcwd() at runtime)
  timeout: 180              # Command timeout in seconds. Default: 180.

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

  # Persistent shell -- keep a long-lived bash process across commands
  persistent_shell: true    # Enabled by default for SSH. Default: true for non-local backends.

  # Singularity
  singularity_image: "docker://nikolaik/python-nodejs:python3.11-nodejs20"

  # Modal
  modal_image: "nikolaik/python-nodejs:python3.11-nodejs20"

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

Configured exclusively through `config.yaml` -- no environment variables. Activates at most once per session. See [providers.md](./providers.md) for full details.

---

### Fallback Provider Chain (v0.6.0)

Added in v0.6.0 (PR [#3813](https://github.com/NousResearch/hermes-agent/pull/3813), closes [#1734](https://github.com/NousResearch/hermes-agent/issues/1734)).

Configure an ordered list of backup providers that Hermes tries automatically when the primary provider fails. Unlike the legacy single-entry `fallback_model`, `fallback_providers` accepts any number of entries and advances through them in order.

```yaml
fallback_providers:
  - provider: openrouter
    model: anthropic/claude-sonnet-4.6
  - provider: anthropic
    model: claude-sonnet-4-6
  - provider: deepseek
    model: deepseek-chat
```

Each entry requires `provider` and `model`. Optional fields `base_url` and `api_key_env` are supported for custom endpoints (same as `fallback_model`).

**Trigger conditions** — fallback is attempted on:

- HTTP 429 (rate limit / quota exhausted)
- HTTP 5xx server errors (500, 503, 529, etc.)
- Network connection failures and timeouts
- Empty or malformed responses (a common rate-limit symptom)

**Fallback is not triggered on:**

- HTTP 401 / 403 (authentication or authorization failures — these indicate a credential problem that a different provider would not fix differently)
- HTTP 413 (payload too large — context compression is attempted instead)
- Context length errors

Each failed attempt is logged at INFO level:

```
INFO Fallback activated: claude-opus-4-6 → anthropic/claude-sonnet-4.6 (openrouter)
```

If all providers in the chain are exhausted, the last error is returned to the user.

**Backward compatibility:** The legacy `fallback_model` single-dict format continues to work and is automatically normalized to a one-element chain. `fallback_providers` takes precedence when both are present.

See [providers.md](./providers.md#fallback-provider-chain-v060) for full details.

---

### Skills

```yaml
skills:
  external_dirs:
    - "~/.agents/shared-skills"
    - "/mnt/team-skills"
```

Configure additional read-only skill directories scanned alongside `~/.hermes/skills/`. Added in v0.6.0 (PR [#3678](https://github.com/NousResearch/hermes-agent/pull/3678)). See [skills.md](./skills.md#external-skill-directories-v060) for full details.

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
```

Routes short, simple turns to a cheap model. Falls back to the primary model if routing fails.

---

### Context Compression

```yaml
compression:
  enabled: true                                     # Toggle compression on/off. Default: true.
  threshold: 0.50                                   # Compress at this % of context limit. Default: 0.50.
  target_ratio: 0.20                                # v0.5.0: fraction of threshold to keep as recent tail. Default: 0.20.
  protect_last_n: 20                                # v0.5.0: min recent messages to preserve. Default: 20.
  summary_model: "google/gemini-3-flash-preview"    # Model for summarization
  summary_provider: "auto"                          # Provider: "auto", "openrouter", "nous", "codex", "main", etc.
  summary_base_url: null                            # Custom OpenAI-compatible endpoint (overrides provider)
```

**v0.4.0 (PR #2323):** Context compression was overhauled to use structured summaries with iterative updates, a configurable `summary_base_url`, and token-budget tail protection.

**v0.5.0 (PR #2554):** The deprecated `summary_target_tokens` key was replaced by ratio-based scaling. `target_ratio` and `protect_last_n` are now the canonical controls and are exposed in `DEFAULT_CONFIG`. Summary output is capped at 12K tokens internally.

**Post-compression tail budget formula:**
```
tail_budget = target_ratio × threshold × context_length
# Example: 1M context, threshold=0.50, target_ratio=0.20 → 100K tokens of recent tail
```

All compression settings are in `config.yaml` only -- no environment variables.

---

### Auxiliary Models

Side-task models for vision, web extraction, compression, session search, skills hub, approval, MCP helpers, and memory flush.

```yaml
auxiliary:
  vision:
    provider: "auto"           # auto | openrouter | nous | codex | main | anthropic | custom
    model: ""                  # e.g. "openai/gpt-4o", "google/gemini-2.5-flash"
    base_url: ""               # Direct OpenAI-compatible endpoint (overrides provider)
    api_key: ""                # API key for base_url (falls back to OPENAI_API_KEY)
  web_extract:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
  compression:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
  session_search:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
  skills_hub:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
  approval:
    provider: "auto"
    model: ""                  # fast/cheap model recommended (e.g. gemini-flash, haiku)
    base_url: ""
    api_key: ""
  mcp:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
  flush_memories:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
```

When `base_url` is set, it takes precedence over `provider`. Authentication uses the configured `api_key`, falling back to `OPENAI_API_KEY`. `OPENROUTER_API_KEY` is never reused for auxiliary task custom endpoints.

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
  model: ""                    # Subagent model override (empty = inherit parent)
  provider: ""                 # Subagent provider override (empty = inherit parent)
  base_url: ""                 # Direct OpenAI-compatible endpoint for subagents
  api_key: ""                  # API key for base_url (falls back to OPENAI_API_KEY)
```

**Precedence:** `delegation.base_url` -> `delegation.provider` -> parent provider (inherited). `delegation.model` -> parent model (inherited). Setting `model` without `provider` changes only the model name while keeping the parent's credentials.

---

### Display

```yaml
display:
  compact: false              # Compact output mode (less whitespace)
  personality: "kawaii"        # Legacy cosmetic field
  resume_display: full        # full (show previous messages on resume) | minimal (one-liner only)
  bell_on_complete: false     # Play terminal bell when agent finishes
  show_reasoning: false       # Show model reasoning/thinking above each response
  streaming: false            # Stream tokens to terminal as they arrive
  show_cost: false            # Show $ cost in the status bar (off by default)
  skin: default               # Built-in or custom CLI skin
  tool_progress: "new"        # Tool progress display level (see table below)
  background_process_notifications: "all"  # Background task notification level (see table below)
  tool_preview_length: 120    # Max characters shown in tool argument previews. Default: 120.
```

**`display.tool_progress`** controls how much detail is shown for tool calls. Also toggled at runtime via the `/verbose` slash command.

| Value | Behavior |
|-------|----------|
| `"off"` | No tool progress output |
| `"new"` (default) | Show tool name when a new tool call starts |
| `"all"` | Show tool name and arguments for every call |
| `"verbose"` | Show full tool call details including results |

**`display.background_process_notifications`** controls notifications from `/background` tasks in the gateway.

| Value | Behavior |
|-------|----------|
| `"all"` (default) | Notify on task start, result, and error |
| `"result"` | Notify only on successful completion |
| `"error"` | Notify only on errors |
| `"off"` | Suppress all background task notifications |

**`display.resume_display`** controls what is shown when resuming a session.

| Value | Behavior |
|-------|----------|
| `"full"` (default) | Replay previous messages on resume |
| `"minimal"` | Show a one-liner summary only |

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
    ref_audio: ''               # Path to reference voice audio (empty = bundled default)
    ref_text: ''                # Path to reference voice transcript (empty = bundled default)
    model: neuphonic/neutts-air-q4-gguf  # HuggingFace model repo
    device: cpu                 # cpu, cuda, or mps
```

Controls both the `text_to_speech` tool and spoken replies in `/voice tts` mode.

---

### STT (Speech-to-Text)

```yaml
stt:
  enabled: true               # Master toggle. Default: true.
  provider: "local"            # "local" | "groq" | "openai"
  local:
    model: "base"              # tiny | base | small | medium | large-v3
  openai:
    model: "whisper-1"         # whisper-1 | gpt-4o-mini-transcribe | gpt-4o-transcribe
```

Provider fallback order: `local` -> `groq` -> `openai`.

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
  silence_threshold: 200        # RMS threshold for speech detection (0-32767)
  silence_duration: 3.0         # Seconds of silence before auto-stop
```

---

### Streaming

**CLI streaming** (`display.streaming`):

```yaml
display:
  streaming: false              # Stream tokens to terminal in real-time. Default: false.
  show_reasoning: false         # Also display reasoning/thinking tokens when streaming. Default: false.
```

**v0.4.0 (PR #2340):** CLI streaming mode was introduced with full spinner and tool-progress display during streaming turns (PR #2161). Reasoning/thinking blocks display when `show_reasoning` is `true` (PR #2118). Context pressure warnings appear in both CLI and gateway (PR #2159).

Enable CLI streaming:
```yaml
display:
  streaming: true
```

**Gateway streaming (Telegram, Discord, Slack):**

```yaml
streaming:
  enabled: false                # Enable progressive message editing. Default: false.
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
  enabled: true               # Enable automatic checkpoints. Default: true.
  max_snapshots: 50            # Max checkpoints to keep per directory. Default: 50.
```

Automatic snapshots before destructive file ops. The agent takes a snapshot of the working directory once per conversation turn (on first `write_file`/`patch` call). Use `/rollback` to restore.

---

### Approvals

```yaml
approvals:
  mode: "manual"              # manual | smart | off
  timeout: 60                 # seconds to wait for approval before auto-denying (v0.6.0)
```

| Mode | Behavior |
|------|----------|
| `manual` (default) | Prompt before executing any flagged command. Interactive dialog in CLI; pending approval queue in messaging. |
| `smart` | Use an auxiliary LLM to assess danger. Low-risk commands auto-approved with session-level persistence. |
| `off` | Skip all approval checks. Equivalent to `--yolo`. Use with caution. |

**`approvals.timeout`** (v0.6.0 — PR [#3886](https://github.com/NousResearch/hermes-agent/pull/3886), closes [#3765](https://github.com/NousResearch/hermes-agent/issues/3765)): How long (in seconds) Hermes waits for the user to respond to a dangerous command approval prompt before automatically denying it. Default is `60` seconds.

Set to `0` to require explicit approval with no timeout (the prompt waits indefinitely):

```yaml
approvals:
  mode: "manual"
  timeout: 0
```

This setting applies in CLI mode. In gateway mode (messaging platforms), approvals use the platform's message queue and the timeout is not enforced the same way.

### Command Allowlist

```yaml
command_allowlist: []         # Permanently allowed dangerous command patterns (added via "always" approval)
```

Patterns are glob-style strings matched against the full command text. When a user selects "always allow" at an approval prompt, the matching pattern is appended to this list automatically. You can also add patterns manually:

```yaml
command_allowlist:
  - "git push *"
  - "docker build *"
  - "npm publish"
```

---

### Logging

Centralized logging was introduced in v0.8.0 (PR [#5430](https://github.com/NousResearch/hermes-agent/pull/5430)). Log files are written to `~/.hermes/logs/` and rotate automatically. Optional config overrides:

```yaml
logging:
  level: "INFO"           # Minimum level for agent.log. Default: INFO.
  max_size_mb: 5          # Max size per log file before rotation (MB). Default: 5.
  backup_count: 3         # Rotated backup files to keep. Default: 3.
```

These settings are optional. Without them, the defaults above apply. See [logging.md](./logging.md) for full details on log files, the `hermes logs` command, and quiet/verbose modes.

---

### Privacy

```yaml
privacy:
  redact_pii: false             # Strip PII from LLM context (gateway only). Default: false.
```

When `true`, the gateway hashes phone numbers, user IDs, and chat IDs before sending them to the LLM. Applies to WhatsApp, Signal, and Telegram. Discord and Slack are excluded because their mention systems require real IDs.

---

### Security

```yaml
security:
  redact_secrets: true          # Redact API keys/tokens from tool output. Default: true.
  tirith_enabled: true          # Pre-exec security scanning via tirith. Default: true.
  tirith_path: "tirith"         # Path to tirith binary.
  tirith_timeout: 5             # Tirith scan timeout in seconds. Default: 5.
  tirith_fail_open: true        # Allow command if tirith scan fails. Default: true.
  website_blocklist:
    enabled: false              # Default: false
    domains: []                 # Domain patterns (supports wildcards)
    shared_files: []            # External files with additional rules (one domain per line)
```

---

### Human Delay

Simulate human-like response pacing in messaging platforms.

```yaml
human_delay:
  mode: "off"                   # off | natural | custom
  min_ms: 800                   # Minimum delay in ms (custom mode). Default: 800.
  max_ms: 2500                  # Maximum delay in ms (custom mode). Default: 2500.
```

---

### Browser

```yaml
browser:
  inactivity_timeout: 120       # Seconds before auto-closing idle sessions. Default: 120.
  record_sessions: false        # Auto-record browser sessions as WebM to ~/.hermes/browser_recordings/.
```

---

### Timezone

```yaml
timezone: ""                    # IANA timezone (e.g. "Asia/Kolkata", "America/New_York"). Empty = server-local.
```

---

### Custom Personalities

```yaml
personalities:
  pirate:
    description: "Talk like a pirate"
    system_prompt: "You speak like a pirate captain."
    tone: "playful"
    style: "pirate-speak"
  # Or simple string form:
  concise: "Be extremely concise. One sentence answers only."
```

Activated via `/personality <name>`.

---

### Discord-Specific Config Keys

```yaml
discord:
  require_mention: true         # Require @mention before responding in server channels. Default: true.
  free_response_channels: ""    # Comma-separated channel IDs where @mention is not required
  auto_thread: true             # Auto-create threads on @mention in channels. Default: true.
  reactions: true               # Show processing reactions in Discord. Default: true.
```

---

### WhatsApp-Specific Config Keys

```yaml
whatsapp:
  require_mention: false        # Require a mention in WhatsApp group chats before responding. Default: false.
  # Reply prefix prepended to every outgoing WhatsApp message.
  # Default uses the built-in header. Set to "" to disable.
  # Supports \n for newlines.
```

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
# Root-level shorthand:
group_sessions_per_user: true   # true = per-user isolation in groups/channels (default)
                                 # false = one shared session per chat

# Or nested under session_reset:
session_reset:
  group_sessions_per_user: true
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

- `pair` -- denies access but replies with a one-time pairing code in DMs.
- `ignore` -- silently drops unauthorized DMs.

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

Available in CLI and all messaging platforms.

---

### Prefill Messages

```yaml
prefill_messages_file: ""       # Path to JSON file of ephemeral prefill messages for few-shot priming
```

The JSON file contains an array of `{"role": "...", "content": "..."}` message objects. These are injected at the start of every API call as few-shot examples or behavioral priming. They are never saved to sessions, logs, or trajectories -- purely ephemeral.

```json
[
  {"role": "user", "content": "What is 2+2?"},
  {"role": "assistant", "content": "4"}
]
```

Also settable via `HERMES_PREFILL_MESSAGES_FILE` environment variable.

---

### Memory Nudge and Flush

```yaml
nudge_interval: 15              # Turns between memory-save nudges. Default: 15.
flush_min_turns: 5              # Minimum turns before automatic memory flush on compression. Default: 5.
```

**`nudge_interval`** controls how often the agent is nudged (via a system-level hint) to save important information to persistent memory. Every N turns, a reminder is injected encouraging the agent to call `memory_manage` if it has learned something worth persisting.

**`flush_min_turns`** sets the minimum number of conversation turns that must have elapsed before an automatic memory flush is triggered during context compression. This prevents flushing on very short conversations where there is little worth persisting.

---

### Cron Wrap Response

```yaml
cron:
  wrap_response: true             # Add header/footer wrapping to cron job output. Default: true.
```

When `true`, cron job responses delivered to messaging platforms are wrapped with a header (showing the job name and schedule) and a footer (showing execution time and next run). Set to `false` for clean, unwrapped output.

---

### Auxiliary Task Timeouts

Individual timeout overrides per auxiliary task, in seconds. These are set inside each auxiliary task block:

```yaml
auxiliary:
  vision:
    timeout: 60                   # Timeout for vision tasks (default: 60)
  web_extract:
    timeout: 90                   # Timeout for web extraction (default: 90)
  compression:
    timeout: 120                  # Timeout for context compression (default: 120)
  session_search:
    timeout: 30                   # Timeout for session search (default: 30)
  mcp:
    timeout: 30                   # Timeout for MCP helper calls (default: 30)
  flush_memories:
    timeout: 60                   # Timeout for memory flush (default: 60)
```

Each timeout controls how long Hermes waits for the auxiliary model to respond for that specific task before giving up. These are independent of `HERMES_API_TIMEOUT`, which controls the primary LLM timeout.

---

### Code Execution

```yaml
execute_code:
  timeout: 30                     # Max seconds for code execution tool. Default: 30.
  max_tool_calls: 5               # Max sequential code executions per turn. Default: 5.
```

Controls the `execute_code` tool used for running inline code snippets. `timeout` caps individual execution time. `max_tool_calls` limits how many code blocks the agent can execute in a single conversation turn to prevent runaway loops.

---

## MCP Servers

```yaml
mcp_servers:
  # Stdio transport — launch a subprocess
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
    env:                            # Extra env vars passed to the subprocess
      MY_SERVER_VAR: "value"
    timeout: 30                     # Per-request timeout in seconds (default: 30)

  # HTTP/SSE transport — connect to a remote server
  remote-server:
    url: "https://mcp.example.com/sse"
    headers:                        # Auth headers sent with every request
      Authorization: "Bearer sk-..."
      X-Custom-Header: "value"
    timeout: 60

  # Sampling config — let the MCP server call back to the LLM
  sampler:
    command: "python"
    args: ["-m", "my_mcp_server"]
    sampling:
      enabled: true                 # Allow server-initiated LLM sampling (default: false)
      max_tokens: 4096              # Cap tokens per sampling request
      model: ""                     # Override model for sampling (empty = inherit main model)
```

Each server entry supports:

| Key | Transport | Description |
|-----|-----------|-------------|
| `command` + `args` | stdio | Binary and arguments to spawn |
| `url` | HTTP/SSE | Remote MCP endpoint URL |
| `env` | stdio | Extra environment variables for the subprocess |
| `headers` | HTTP/SSE | HTTP headers (typically auth tokens) |
| `timeout` | both | Per-request timeout in seconds (default: 30) |
| `sampling.enabled` | both | Allow the MCP server to request LLM completions (default: false) |
| `sampling.max_tokens` | both | Cap on tokens per sampling request |
| `sampling.model` | both | Model override for sampling calls |

Use `/reload-mcp` inside a session to reload MCP servers without restarting.

---

## Honcho Section

```yaml
honcho:
  # Configured via: hermes honcho setup
  # See: hermes honcho status
```

Honcho cross-session memory is managed interactively via `hermes honcho setup`. The `HONCHO_API_KEY` environment variable enables it automatically.

---

## Context Files (SOUL.md, AGENTS.md)

| File | Purpose | Scope |
|------|---------|-------|
| `SOUL.md` | Primary agent identity -- occupies slot #1 in system prompt | `~/.hermes/SOUL.md` or `$HERMES_HOME/SOUL.md` |
| `AGENTS.md` | Project-specific instructions, coding conventions | Working directory / project tree (hierarchical) |
| `.cursorrules` | Cursor IDE rules (also detected) | Working directory |
| `.cursor/rules/*.mdc` | Cursor rule files (also detected) | Working directory |

All loaded context files are capped at 20,000 characters with smart truncation.

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
| `MINIMAX_API_KEY` | MiniMax API key -- global endpoint |
| `MINIMAX_BASE_URL` | Override MiniMax base URL. Default: `https://api.minimax.io/v1`. |
| `MINIMAX_CN_API_KEY` | MiniMax API key -- China endpoint |
| `MINIMAX_CN_BASE_URL` | Override MiniMax China base URL. Default: `https://api.minimaxi.com/v1`. |
| `KILOCODE_API_KEY` | Kilo Code API key |
| `KILOCODE_BASE_URL` | Override Kilo Code base URL. Default: `https://api.kilo.ai/api/gateway`. |
| `ANTHROPIC_API_KEY` | Anthropic Console API key |
| `ANTHROPIC_TOKEN` | Manual or legacy Anthropic OAuth/setup-token override |
| `CLAUDE_CODE_OAUTH_TOKEN` | Explicit Claude Code token override |
| `DASHSCOPE_API_KEY` | Alibaba Cloud DashScope API key (Qwen models) |
| `DASHSCOPE_BASE_URL` | Custom DashScope base URL |
| `DEEPSEEK_API_KEY` | DeepSeek API key |
| `DEEPSEEK_BASE_URL` | Custom DeepSeek API base URL |
| `OPENCODE_ZEN_API_KEY` | OpenCode Zen API key (pay-as-you-go) |
| `OPENCODE_ZEN_BASE_URL` | Override OpenCode Zen base URL |
| `OPENCODE_GO_API_KEY` | OpenCode Go API key ($10/month) |
| `OPENCODE_GO_BASE_URL` | Override OpenCode Go base URL |
| `HERMES_MODEL` | Preferred model name. Checked before `LLM_MODEL`. |
| `LLM_MODEL` | Default model name. Fallback when not set in `config.yaml`. |
| `HERMES_HOME` | Override Hermes config directory. Default: `~/.hermes`. |

### Provider Auth (OAuth)

| Variable | Description |
|----------|-------------|
| `HERMES_INFERENCE_PROVIDER` | Override provider selection: `auto`, `openrouter`, `nous`, `openai-codex`, `copilot`, `anthropic`, `zai`, `kimi-coding`, `minimax`, `minimax-cn`, `kilocode`, `alibaba`, `deepseek`, `opencode-zen`, `opencode-go`, `ai-gateway`. Default: `auto`. |
| `HERMES_PORTAL_BASE_URL` | Override Nous Portal URL (for development/testing) |
| `NOUS_INFERENCE_BASE_URL` | Override Nous inference API URL |
| `HERMES_NOUS_MIN_KEY_TTL_SECONDS` | Min agent key TTL before re-mint. Default: `1800` (30 min). |
| `HERMES_NOUS_TIMEOUT_SECONDS` | HTTP timeout for Nous credential/token flows. Default: `15`. |
| `HERMES_DUMP_REQUESTS` | Dump API request payloads to log files. Values: `true`/`false`. |
| `HERMES_PREFILL_MESSAGES_FILE` | Path to a JSON file of ephemeral prefill messages |
| `HERMES_TIMEZONE` | IANA timezone override. Example: `America/New_York`. |
| `HERMES_OAUTH_TRACE` | Enable verbose OAuth tracing. Values: `1`/`true`/`yes`/`on`. |

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
| `VOICE_TOOLS_OPENAI_KEY` | OpenAI key for speech-to-text (Whisper) and OpenAI TTS |
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
| `TERMINAL_DOCKER_IMAGE` | Docker image. Default: `nikolaik/python-nodejs:python3.11-nodejs20`. |
| `TERMINAL_DOCKER_FORWARD_ENV` | JSON array of env var names to forward into Docker terminal sessions |
| `TERMINAL_DOCKER_VOLUMES` | Additional Docker volume mounts (comma-separated `host:container` pairs) |
| `TERMINAL_DOCKER_MOUNT_CWD_TO_WORKSPACE` | Mount the launch cwd into Docker `/workspace`. Values: `true`/`false`. Default: `false`. |
| `TERMINAL_SINGULARITY_IMAGE` | Singularity image or `.sif` path |
| `TERMINAL_MODAL_IMAGE` | Modal container image |
| `TERMINAL_DAYTONA_IMAGE` | Daytona sandbox image |
| `TERMINAL_TIMEOUT` | Command timeout in seconds. Default: `180`. |
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
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `DISCORD_ALLOWED_USERS` | Comma-separated Discord user IDs |
| `DISCORD_HOME_CHANNEL` | Default Discord channel for cron delivery |
| `SLACK_BOT_TOKEN` | Slack bot token (`xoxb-...`) |
| `SLACK_APP_TOKEN` | Slack app-level token (`xapp-...`, required for Socket Mode) |
| `SLACK_ALLOWED_USERS` | Comma-separated Slack user IDs |
| `WHATSAPP_ENABLED` | Enable the WhatsApp bridge. Values: `true`/`false`. |
| `WHATSAPP_MODE` | `bot` (separate number) or `self-chat` (message yourself) |
| `WHATSAPP_ALLOWED_USERS` | Comma-separated phone numbers (with country code, no `+`) |
| `SIGNAL_HTTP_URL` | signal-cli daemon HTTP endpoint (e.g. `http://127.0.0.1:8080`) |
| `SIGNAL_ACCOUNT` | Bot phone number in E.164 format |
| `SIGNAL_ALLOWED_USERS` | Comma-separated E.164 phone numbers or UUIDs |
| `SIGNAL_GROUP_ALLOWED_USERS` | Comma-separated group IDs, or `*` for all groups |
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
| `DINGTALK_CLIENT_ID` | DingTalk bot AppKey |
| `DINGTALK_CLIENT_SECRET` | DingTalk bot AppSecret |
| `TWILIO_ACCOUNT_SID` | Twilio Account SID |
| `TWILIO_AUTH_TOKEN` | Twilio Auth Token |
| `TWILIO_PHONE_NUMBER` | Twilio phone number in E.164 format |
| `SMS_ALLOWED_USERS` | Comma-separated E.164 phone numbers allowed to chat |
| `SMS_HOME_CHANNEL` | Phone number for cron job/notification delivery |
| `EMAIL_ADDRESS` | Email address for the Email gateway adapter |
| `EMAIL_PASSWORD` | Password or app password for the email account |
| `EMAIL_IMAP_HOST` | IMAP hostname |
| `EMAIL_SMTP_HOST` | SMTP hostname |
| `EMAIL_ALLOWED_USERS` | Comma-separated email addresses allowed to message the bot |
| `API_SERVER_ENABLED` | Enable the OpenAI-compatible API server. Values: `true`/`false`. |
| `API_SERVER_KEY` | Bearer token for API server authentication |
| `API_SERVER_PORT` | Port for the API server. Default: `8642`. |
| `API_SERVER_HOST` | Host/bind address. Default: `127.0.0.1`. |
| `MESSAGING_CWD` | Working directory for terminal commands in messaging mode. Default: `~`. |
| `GATEWAY_ALLOW_ALL_USERS` | Allow all users without allowlists. Values: `true`/`false`. Default: `false`. |

### Agent Behavior

| Variable | Description |
|----------|-------------|
| `HERMES_MAX_ITERATIONS` | Max tool-calling iterations per conversation. Default: `90`. |
| `HERMES_TOOL_PROGRESS` | Deprecated: use `display.tool_progress` in `config.yaml`. |
| `HERMES_TOOL_PROGRESS_MODE` | Deprecated: use `display.tool_progress` in `config.yaml`. |
| `HERMES_HUMAN_DELAY_MODE` | Response pacing: `off`/`natural`/`custom`. |
| `HERMES_HUMAN_DELAY_MIN_MS` | Custom delay range minimum (ms). Default: `800`. |
| `HERMES_HUMAN_DELAY_MAX_MS` | Custom delay range maximum (ms). Default: `2500`. |
| `HERMES_QUIET` | Suppress non-essential output. Values: `true`/`false`. |
| ~~`HERMES_REASONING_EFFORT`~~ | **Removed in v0.8.0.** Set `agent.reasoning_effort` in `config.yaml` instead. |
| `HERMES_API_TIMEOUT` | LLM API call timeout in seconds. Default: `900`. |
| `HERMES_EXEC_ASK` | Enable execution approval prompts in gateway mode. Values: `true`/`false`. |
| `HERMES_BACKGROUND_NOTIFICATIONS` | Background process notification mode in gateway: `all` (default), `result`, `error`, `off`. |
| `HERMES_EPHEMERAL_SYSTEM_PROMPT` | Ephemeral system prompt injected at API-call time (never persisted to sessions). |

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
