# Installation

This guide covers every installation method for Hermes Agent, from the one-line quick installer to fully manual source builds. Covers v0.2.0 (v2026.3.12) and v0.3.0 (v2026.3.17).

---

## Prerequisites

### Required

- **Git** — the only prerequisite you must install yourself. The installer handles everything else.

  Verify: `git --version`

### Automatically handled by the installer

The one-line and `setup-hermes.sh` installers detect what is missing and install it without requiring `sudo`:

| Dependency | Version | Purpose |
|------------|---------|---------|
| **uv** | latest | Fast Python package manager |
| **Python** | 3.11 | Runtime (installed via uv, no sudo needed) |
| **Node.js** | v22 | Browser automation and WhatsApp bridge |
| **ripgrep** | latest | Fast file search (falls back to grep if missing) |
| **ffmpeg** | latest | Audio format conversion for TTS |

### Supported platforms

| Platform | Support |
|----------|---------|
| Linux (any distro) | Fully supported |
| macOS | Fully supported |
| WSL2 | Fully supported |
| Windows (native) | **Not supported** — use WSL2 |

---

## Method 1: One-Line Installer (Recommended)

The fastest way to get started on Linux, macOS, or WSL2:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

The installer:

1. Installs `uv` if not present
2. Installs Python 3.11 via uv (no sudo required)
3. Clones the repository with submodules
4. Creates a Python virtual environment
5. Installs all dependencies including submodule packages
6. Creates the `~/.hermes/` directory structure
7. Copies the example config to `~/.hermes/config.yaml`
8. Symlinks the `hermes` command into `~/.local/bin`
9. Adds `~/.local/bin` to your shell's PATH
10. Installs bundled skills to `~/.hermes/skills/`
11. Prompts you to run the setup wizard

After installation, reload your shell and start chatting:

```bash
source ~/.bashrc   # or: source ~/.zshrc
hermes
```

---

## Method 2: setup-hermes.sh Script

For developers who clone the repository manually, `setup-hermes.sh` in the repo root performs the same steps as the one-line installer without fetching from the internet:

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh
```

The script performs these steps in order:

1. Checks for `uv` in `PATH`, `~/.local/bin/uv`, and `~/.cargo/bin/uv` — installs via `curl -LsSf https://astral.sh/uv/install.sh | sh` if missing
2. Checks for Python 3.11 via `uv python find 3.11` — installs via `uv python install 3.11` if missing
3. Removes any existing `venv` directory and creates a fresh one with `uv venv venv --python 3.11`
4. Sets `VIRTUAL_ENV` so uv installs into the correct venv
5. Installs all extras with `uv pip install -e ".[all]"`, falling back to `uv pip install -e "."` on failure
6. Installs submodule packages:
   - `uv pip install -e "./mini-swe-agent"` (terminal tool backend)
   - `uv pip install -e "./tinker-atropos"` (RL training backend)
7. Optionally installs `ripgrep` via apt, dnf, brew, or cargo
8. Creates `.env` from `.env.example` if `.env` does not already exist
9. Symlinks `venv/bin/hermes` into `~/.local/bin/hermes`
10. Detects your shell (zsh, bash) and adds `export PATH="$HOME/.local/bin:$PATH"` to the appropriate config file if `~/.local/bin` is not already on PATH
11. Syncs bundled skills to `~/.hermes/skills/` via `tools/skills_sync.py`
12. Prompts to run the setup wizard: `hermes setup`

After the script completes, reload your shell:

```bash
source ~/.zshrc   # or ~/.bashrc
```

---

## Method 3: Manual Installation (Full Control)

### Step 1: Clone the Repository

Always clone with `--recurse-submodules` to pull the required submodules:

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
```

If you already cloned without the flag:

```bash
git submodule update --init --recursive
```

### Step 2: Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc  # or source ~/.zshrc
```

### Step 3: Create a Virtual Environment

```bash
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"
```

You do not need to activate the venv. The `hermes` entry point has a shebang pointing to the venv Python and works globally once symlinked.

### Step 4: Install Python Dependencies

Install with all optional extras:

```bash
uv pip install -e ".[all]"
```

Install only the core agent (no messaging, no cron, no browser tools):

```bash
uv pip install -e "."
```

Available extras you can combine with `[extra1,extra2]`:

| Extra | What it adds | Install |
|-------|-------------|---------|
| `all` | All extras combined | `uv pip install -e ".[all]"` |
| `messaging` | Telegram, Discord, Slack gateway | `uv pip install -e ".[messaging]"` |
| `cron` | Cron expression parsing | `uv pip install -e ".[cron]"` |
| `cli` | Terminal menu UI for setup wizard | `uv pip install -e ".[cli]"` |
| `modal` | Modal cloud execution backend | `uv pip install -e ".[modal]"` |
| `daytona` | Daytona cloud sandbox backend | `uv pip install -e ".[daytona]"` |
| `tts-premium` | ElevenLabs premium TTS voices | `uv pip install -e ".[tts-premium]"` |
| `voice` | CLI microphone input and audio playback | `uv pip install -e ".[voice]"` |
| `pty` | PTY terminal support | `uv pip install -e ".[pty]"` |
| `honcho` | Honcho AI-native memory | `uv pip install -e ".[honcho]"` |
| `mcp` | Model Context Protocol support | `uv pip install -e ".[mcp]"` |
| `homeassistant` | Home Assistant integration | `uv pip install -e ".[homeassistant]"` |
| `acp` | ACP editor integration | `uv pip install -e ".[acp]"` |
| `slack` | Slack messaging | `uv pip install -e ".[slack]"` |
| `dev` | pytest and test utilities | `uv pip install -e ".[dev]"` |

Combine extras: `uv pip install -e ".[messaging,cron,honcho]"`

### Step 5: Install Submodule Packages

```bash
# Terminal tool backend (required for terminal/command-execution tools)
uv pip install -e "./mini-swe-agent"

# RL training backend (optional)
uv pip install -e "./tinker-atropos"
```

Without `mini-swe-agent`, the terminal toolset is unavailable. Without `tinker-atropos`, RL training tools are unavailable.

### Step 6: Install Node.js Dependencies (Optional)

Required only for browser automation (Browserbase) and WhatsApp bridge:

```bash
npm install
```

### Step 7: Create the Configuration Directory

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills,pairing,hooks,image_cache,audio_cache,whatsapp/session}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env
```

### Step 8: Add API Keys

Open `~/.hermes/.env` and add at least one LLM provider key:

```bash
# Required — at least one LLM provider:
OPENROUTER_API_KEY=sk-or-v1-your-key-here

# Optional — enable additional tools:
FIRECRAWL_API_KEY=fc-your-key          # Web search and scraping
FAL_KEY=your-fal-key                   # Image generation (FLUX)
```

Or set keys via the CLI:

```bash
hermes config set OPENROUTER_API_KEY sk-or-v1-your-key-here
```

### Step 9: Add `hermes` to PATH

```bash
mkdir -p ~/.local/bin
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
```

Add `~/.local/bin` to PATH if it is not already there:

```bash
# Bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# Zsh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

# Fish
fish_add_path $HOME/.local/bin
```

### Step 10: Configure Your Provider

```bash
hermes model    # Select your LLM provider and model
```

---

## Setup Wizard

After any installation method, run the interactive setup wizard to configure your provider, terminal backend, messaging platforms, and tools:

```bash
hermes setup
```

The wizard is modular — you can run individual sections at any time:

```bash
hermes model          # Choose LLM provider and model
hermes tools          # Configure which tools are enabled
hermes gateway setup  # Set up messaging platforms
hermes config set     # Set individual config values
```

Wizard sections:

1. **Model and Provider** — choose your AI provider and model
2. **Terminal Backend** — where the agent runs commands (`local`, `docker`, `ssh`, `modal`, `daytona`, `singularity`)
3. **Messaging Platforms** — connect Telegram, Discord, Slack, WhatsApp, Signal, Email, or Home Assistant
4. **Tools** — configure TTS, web search, image generation, and other tool-specific settings
5. **Agent Settings** — max iterations, context compression, session reset behavior

---

## Provider Credential Configuration

### Supported Providers and Their Credentials

| Provider | Auth Method | Credential |
|----------|-------------|-----------|
| **Nous Portal** | OAuth device code via `hermes model` | Stored in `~/.hermes/auth.json` |
| **OpenAI Codex** | OAuth device code via `hermes model` | Stored in `~/.hermes/auth.json` |
| **Anthropic** | Claude Code auto-discovery or API key | `ANTHROPIC_API_KEY` or `ANTHROPIC_TOKEN` |
| **GitHub Copilot** | GitHub token | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, or `GITHUB_TOKEN` |
| **OpenRouter** | API key | `OPENROUTER_API_KEY` |
| **Z.AI / GLM** | API key | `GLM_API_KEY` or `ZAI_API_KEY` |
| **Kimi / Moonshot** | API key | `KIMI_API_KEY` |
| **MiniMax** | API key | `MINIMAX_API_KEY` |
| **MiniMax China** | API key | `MINIMAX_CN_API_KEY` |
| **Alibaba Cloud** | API key | `DASHSCOPE_API_KEY` |
| **Kilo Code** | API key | `KILOCODE_API_KEY` |
| **Custom Endpoint** | Base URL plus API key | `OPENAI_BASE_URL` and `OPENAI_API_KEY` |

For local models (Ollama, vLLM, llama.cpp, SGLang):

```bash
hermes config set OPENAI_BASE_URL http://localhost:11434/v1
hermes config set OPENAI_API_KEY ollama
hermes config set HERMES_MODEL llama3.1
```

### Credential Priority Order

The runtime provider resolver (`hermes_cli/runtime_provider.py`) resolves credentials in this order:

1. Explicit CLI argument (`--provider`, `--api-key`)
2. `model.provider` in `~/.hermes/config.yaml`
3. `HERMES_INFERENCE_PROVIDER` environment variable
4. Auto-detection from available env vars and the auth store

For OpenRouter, when the resolved base URL contains `openrouter.ai`, `OPENROUTER_API_KEY` is preferred over `OPENAI_API_KEY` to prevent key leakage to unrelated providers. For custom endpoints, `OPENAI_API_KEY` takes precedence.

---

## Verifying the Installation

```bash
hermes version    # Print the installed version
hermes doctor     # Run diagnostics — reports what is working or missing
hermes status     # Show current configuration summary
hermes chat -q "Hello! What tools do you have available?"
```

`hermes doctor` checks for missing config keys, unavailable tools, provider connectivity, and other common issues. It also validates Honcho configuration when enabled.

---

## Condensed Manual Install Reference

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and enter repo
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# Create venv with Python 3.11
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"

# Install everything
uv pip install -e ".[all]"
uv pip install -e "./mini-swe-agent"
uv pip install -e "./tinker-atropos"
npm install   # optional: browser tools and WhatsApp

# Configure
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills,pairing,hooks,image_cache,audio_cache,whatsapp/session}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env
echo 'OPENROUTER_API_KEY=sk-or-v1-your-key' >> ~/.hermes/.env

# Make hermes available globally
mkdir -p ~/.local/bin
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

# Verify
hermes doctor
hermes
```

---

## Updating to a New Version

```bash
hermes update
```

This pulls the latest code, updates dependencies, and prompts you to configure any new options added since your last update. If you skipped the post-update prompt:

```bash
hermes config check    # Show missing config options
hermes config migrate  # Interactively add missing options
```

For manual (non-installer) installations:

```bash
cd /path/to/hermes-agent
export VIRTUAL_ENV="$(pwd)/venv"

git pull origin main
git submodule update --init --recursive

uv pip install -e ".[all]"
uv pip install -e "./mini-swe-agent"
uv pip install -e "./tinker-atropos"

hermes config check
hermes config migrate
```

You can also update from a connected messaging platform by sending `/update` to the bot.

---

## Troubleshooting Installation Issues

| Problem | Solution |
|---------|----------|
| `hermes: command not found` | Reload shell: `source ~/.bashrc` or `source ~/.zshrc`. Verify `~/.local/bin` is on PATH with `echo $PATH`. |
| Python version too old | Run `uv python install 3.11`. Hermes requires Python 3.11+. |
| `uv: command not found` | Run `curl -LsSf https://astral.sh/uv/install.sh \| sh && source ~/.bashrc` |
| Permission denied during install | Do not use `sudo`. The installer writes to `~/.local/bin` and requires no root access. |
| `API key not set` | Run `hermes model` to configure your provider, or `hermes config set OPENROUTER_API_KEY your_key` |
| Missing config after update | Run `hermes config check` then `hermes config migrate` |
| Submodule directory is empty | Run `git submodule update --init --recursive` inside the repo root |
| Node.js not found | Install Node.js v22 from nodejs.org, or re-run the installer which installs it automatically |
| `mini-swe-agent` install fails | Run `git submodule update --init --recursive`, then retry `uv pip install -e "./mini-swe-agent"` |

For detailed diagnostics, run `hermes doctor` — it reports exactly what is missing and how to fix it.

---

## Uninstalling

```bash
hermes uninstall
```

The uninstaller prompts whether to keep `~/.hermes/` (your config, memories, and sessions) for a future reinstall.

Manual uninstall:

```bash
rm -f ~/.local/bin/hermes
rm -rf /path/to/hermes-agent
rm -rf ~/.hermes    # Optional — keep if you plan to reinstall
```

If the gateway is installed as a system service, stop it first:

```bash
hermes gateway stop
# Linux:
systemctl --user disable hermes-gateway
# macOS:
launchctl remove ai.hermes.gateway
```
