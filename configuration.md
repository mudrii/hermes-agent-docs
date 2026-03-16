# Configuration

Hermes Agent is configured via `~/.hermes/config.yaml`. Keys use snake_case throughout.

Config is read at startup. Force a reload: `hermes config reload` or restart.

---

## Minimal Configuration

```yaml
# ~/.hermes/config.yaml

model: "anthropic/claude-opus-4.6"    # OpenRouter format: provider/model
```

API keys go in `~/.hermes/.env` (0600 permissions):

```bash
OPENROUTER_API_KEY=sk-or-...
```

---

## Full Configuration Reference

### Model

```yaml
model: "anthropic/claude-opus-4.6"    # Default LLM model (OpenRouter format)
```

### Agent Behavior

```yaml
agent:
  max_turns: 90                        # Maximum tool-calling iterations
  reasoning_effort: "xhigh"           # xhigh | high | medium | low | minimal | none
```

### Terminal Backend

```yaml
terminal:
  backend: "local"                     # local | docker | ssh | modal | daytona | singularity
  cwd: "."                             # Working directory
  timeout: 180                         # Command timeout in seconds
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  container_cpu: 1                     # CPU cores for Docker container
  container_memory: 5120              # Memory in MB
  container_disk: 51200               # Disk in MB
  persistent_shell: true              # Reuse shell across commands
  docker_volumes: []                   # Additional volume mounts
```

### Browser

```yaml
browser:
  inactivity_timeout: 120             # Seconds before closing idle browser
  record_sessions: false              # Save video recordings
```

### Context Compression

```yaml
compression:
  enabled: true
  threshold: 0.50                     # Trigger compression at 50% token budget
  summary_model: "google/gemini-3-flash-preview"  # Model for summarization
```

### Auxiliary LLM (Side-Task Models)

```yaml
auxiliary:
  vision:
    provider: "auto"                   # auto | openai | anthropic | custom
    model: ""                          # Empty = auto-select
    base_url: ""
    api_key: ""
  web_extract:
    provider: "auto"
    model: ""
  compression:
    provider: "auto"
    model: ""
  session_search:
    provider: "auto"
    model: ""
  skills_hub:
    provider: "auto"
    model: ""
  mcp:
    provider: "auto"
    model: ""
```

### Display

```yaml
display:
  compact: false                       # Compact mode (less verbose)
  personality: "kawaii"               # Display personality style
  resume_display: "full"              # full | brief | none
  show_reasoning: false               # Show extended reasoning steps
  skin: "default"                     # CLI skin/theme
```

### Text-to-Speech (TTS)

```yaml
tts:
  provider: "edge"                     # edge (free) | elevenlabs | openai

  edge:
    voice: "en-US-AriaNeural"         # Any Edge TTS voice name

  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"
    model_id: "eleven_multilingual_v2"

  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"                     # alloy | echo | fable | onyx | nova | shimmer
```

### Speech-to-Text (STT)

```yaml
stt:
  enabled: true
  provider: "local"                    # local | groq | openai

  local:
    model: "base"                      # tiny | base | small | medium | large-v3
```

### Voice Input

```yaml
voice:
  record_key: "ctrl+b"               # Keybinding to start recording
  max_recording_seconds: 120
  auto_tts: false                    # Auto-speak responses
```

### Filesystem Checkpoints

```yaml
checkpoints:
  enabled: false
  max_snapshots: 50
```

### Session Reset (Gateway)

```yaml
session_reset:
  mode: "both"                       # both | daily | idle | none
  at_hour: 0                         # Hour for daily reset (0-23)
  idle_minutes: 60                   # Inactivity timeout for idle reset
```

### Discord-Specific

```yaml
discord:
  require_mention: true              # Require @bot mention in servers
  free_response_channels: []         # Channel IDs where mention not required
  auto_thread: false                 # Auto-create threads for responses
```

### Quick Commands

Define custom shortcut commands:

```yaml
quick_commands:
  /standup: "Prepare my daily standup summary from recent work"
  /review: "Do a code review of recent changes"
  /brief: "Give me a morning briefing: weather, calendar, emails"
```

### Timezone

```yaml
timezone: "America/New_York"        # IANA timezone name
```

---

## Environment Variables

All sensitive values go in `~/.hermes/.env`:

### LLM Providers

```bash
OPENROUTER_API_KEY=sk-or-...
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GLM_API_KEY=...
KIMI_API_KEY=...
MINIMAX_API_KEY=...
NOUS_PORTAL_ACCESS_TOKEN=...
```

### Tool API Keys

```bash
FIRECRAWL_API_KEY=...        # web_extract tool
FAL_KEY=...                   # image_generate tool
HONCHO_API_KEY=...            # User memory persistence
BROWSERBASE_API_KEY=...       # Browser automation (optional)
VOICE_TOOLS_OPENAI_KEY=...    # Whisper/TTS (optional)
```

### Messaging Platforms

```bash
TELEGRAM_BOT_TOKEN=...
TELEGRAM_HOME_CHANNEL=123456789

DISCORD_BOT_TOKEN=...
DISCORD_HOME_CHANNEL=987654321

SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
SLACK_HOME_CHANNEL=C0123456789

WHATSAPP_ENABLED=true
WHATSAPP_ALLOWED_USERS=+1234567890,+0987654321

SIGNAL_HTTP_URL=http://localhost:8080
SIGNAL_ACCOUNT=+1234567890
SIGNAL_HOME_CHANNEL=+0987654321

EMAIL_ADDRESS=bot@example.com
EMAIL_PASSWORD=...
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com

HASS_TOKEN=...
HASS_URL=http://homeassistant.local:8123
```

### Terminal Backends

```bash
TERMINAL_ENV=local             # local | docker | singularity | modal | ssh | daytona
TERMINAL_DOCKER_IMAGE=...
TERMINAL_MODAL_IMAGE=...
TERMINAL_SSH_HOST=...
TERMINAL_SSH_USER=...
TERMINAL_SSH_KEY=~/.ssh/id_rsa
```

### System

```bash
HERMES_HOME=~/.hermes          # Override config directory
HERMES_TIMEZONE=UTC            # Force timezone (overrides config.yaml)
```

---

## Gateway Configuration

The gateway uses `~/.hermes/gateway.json` (or environment variables):

```json
{
  "platforms": {
    "telegram": {
      "enabled": true,
      "token": "${TELEGRAM_BOT_TOKEN}",
      "home_channel": {"platform": "telegram", "chat_id": "123456789"}
    },
    "discord": {
      "enabled": true,
      "token": "${DISCORD_BOT_TOKEN}",
      "require_mention": true,
      "free_response_channels": [],
      "auto_thread": false
    }
  },
  "default_reset_policy": {
    "mode": "both",
    "at_hour": 0,
    "idle_minutes": 60
  },
  "reset_triggers": ["/new", "/reset"],
  "quick_commands": {},
  "stt_enabled": true,
  "group_sessions_per_user": true,
  "always_log_local": true
}
```

---

## Honcho Configuration

```yaml
# In ~/.hermes/config.yaml

honcho:
  enabled: true                        # Auto-enabled if API key present
  api_key: "${HONCHO_API_KEY}"
  environment: "production"
  workspace_id: "hermes"

  peer_name: "my-laptop"              # Display name for this host
  ai_peer: "hermes"                   # Honcho AI peer name

  save_messages: true                  # Persist messages to Honcho
  memory_mode: "hybrid"               # hybrid | honcho | context
  recall_mode: "hybrid"               # hybrid | context | tools

  write_frequency: "async"            # async | turn | session | N (int)
  context_tokens: 800                 # Token budget for context prefetch

  dialectic_reasoning_level: "medium" # minimal | low | medium | high | max
  dialectic_max_chars: 600

  session_strategy: "per-session"     # per-session | per-repo | per-directory | global
```

---

## Configuration via CLI

```bash
# Show current config
hermes config show

# Set a value
hermes config set terminal.backend docker
hermes config set agent.max_turns 50
hermes config set display.compact true

# Get a value
hermes config get terminal.backend

# Open in $EDITOR
hermes config edit
```
