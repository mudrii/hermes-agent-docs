# Installation

---

## Quick Install (Recommended)

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.zshrc    # or ~/.bashrc
hermes             # start immediately
```

The install script handles Python, Node.js, all dependencies, and the `hermes` command. No prerequisites except git.

**Supported platforms:** Linux, macOS, WSL2
**Not supported:** Windows native (WSL2 required)

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | ≥ 3.11 | Required |
| Node.js | ≥ 18 | Optional — needed for WhatsApp (Baileys) |
| An LLM API key | any | Nous Portal works free to start |

---

## Manual Install (Developer)

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
git submodule update --init mini-swe-agent

# Install uv (fast Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create virtual environment
uv venv .venv --python 3.11
source .venv/bin/activate

# Install with all dependencies
uv pip install -e ".[all,dev]"
uv pip install -e "./mini-swe-agent"

# Verify
python -m pytest tests/ -q
hermes --version
```

---

## Optional Dependency Groups

Install only what you need:

```bash
# Messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal)
uv pip install -e ".[messaging]"

# Slack only (adds slack-bolt + slack-sdk)
uv pip install -e ".[slack]"

# Text-to-speech premium (ElevenLabs)
uv pip install -e ".[tts-premium]"

# Voice input (microphone recording)
uv pip install -e ".[voice]"

# Modal serverless backend
uv pip install -e ".[modal]"

# Daytona dev environment backend
uv pip install -e ".[daytona]"

# Honcho user memory
uv pip install -e ".[honcho]"

# MCP support
uv pip install -e ".[mcp]"

# Home Assistant integration
uv pip install -e ".[homeassistant]"

# ACP editor protocol
uv pip install -e ".[acp]"

# RL training tools
uv pip install -e ".[rl]"

# Cron scheduler
uv pip install -e ".[cron]"

# Everything
uv pip install -e ".[all]"

# Development tools (pytest, pytest-asyncio)
uv pip install -e ".[dev]"
```

---

## First Run

### 1. Interactive Setup Wizard

```bash
hermes setup
```

The setup wizard walks you through:
- Choosing your LLM provider
- Entering API keys (stored securely in `~/.hermes/.env` with 0600 permissions)
- Configuring the terminal backend
- Setting up messaging platforms (optional)

### 2. Quick Model Selection

```bash
hermes model     # Interactive model selector
```

Select from the built-in model catalog or enter a custom model string.

### 3. Start Chatting

```bash
hermes           # Start interactive CLI
```

---

## Configuration Paths

All configuration is stored in `~/.hermes/`:

| Path | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | Main configuration file |
| `~/.hermes/.env` | API keys (0600 permissions) |
| `~/.hermes/auth.json` | OAuth tokens (v1 format) |
| `~/.hermes/state.db` | SQLite session database |
| `~/.hermes/SOUL.md` | AI identity/persona prompt |
| `~/.hermes/skills/` | Installed skills |
| `~/.hermes/cron/` | Scheduled tasks |
| `~/.hermes/sessions/` | Session files |
| `~/.hermes/memories/` | Memory files |
| `~/.hermes/logs/` | Log files |

---

## Update

```bash
hermes update
```

Or for manual installs:

```bash
cd /path/to/hermes-agent
git pull origin main
uv pip install -e ".[all]"
```

---

## Uninstall

```bash
hermes uninstall
```

This removes the `hermes` command and optionally the `~/.hermes/` data directory.

---

## Diagnostics

```bash
hermes doctor
```

Runs a system health check: verifies Python version, dependencies, API key validity, terminal backend availability, and messaging platform connectivity.

---

## Environment Variables

Key environment variables (can also be set in `~/.hermes/.env`):

```bash
# LLM Providers
OPENROUTER_API_KEY=sk-or-...
NOUS_PORTAL_ACCESS_TOKEN=...
ANTHROPIC_API_KEY=sk-ant-...
GLM_API_KEY=...
KIMI_API_KEY=...
MINIMAX_API_KEY=...

# Tools
FIRECRAWL_API_KEY=...     # Web crawling (web_extract tool)
FAL_KEY=...               # Image generation (image_generate tool)
HONCHO_API_KEY=...        # User memory persistence
BROWSERBASE_API_KEY=...   # Browser automation

# Messaging Platforms
TELEGRAM_BOT_TOKEN=...
DISCORD_BOT_TOKEN=...
SLACK_BOT_TOKEN=...
SLACK_APP_TOKEN=...

# Terminal Backend
TERMINAL_ENV=local        # local | docker | ssh | modal | daytona | singularity

# Hermes Home
HERMES_HOME=~/.hermes     # Override config directory
HERMES_TIMEZONE=UTC       # Force timezone
```

---

## Migrate from OpenClaw

```bash
hermes claw migrate
```

Runs the official OpenClaw migration skill, which imports agents, memory, channels, and sessions from your existing OpenClaw installation.
