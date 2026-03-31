# Installation

This guide covers every installation method for Hermes Agent, from the one-line quick installer to fully manual source builds.

---

## Prerequisites

### Required

- **Git** -- the only prerequisite you must install yourself. The installer handles everything else.

  Verify: `git --version`

### Automatically handled by the installer

The one-line and `setup-hermes.sh` installers detect what is missing and install it without requiring `sudo`:

| Dependency | Version | Purpose |
|------------|---------|---------|
| **uv** | latest | Fast Python package manager |
| **Python** | 3.11 | Runtime (installed via uv, no sudo needed) |
| **Node.js** | v22 LTS | Browser automation and WhatsApp bridge |
| **ripgrep** | latest | Fast file search (falls back to grep if missing) |
| **ffmpeg** | latest | Audio format conversion for TTS voice messages |

### Supported platforms

| Platform | Support |
|----------|---------|
| Linux (any distro) | Fully supported (apt, dnf, pacman auto-detected) |
| macOS (Intel and Apple Silicon) | Fully supported |
| WSL2 | Fully supported |
| Windows (native PowerShell) | Supported via `install.ps1` (installs to `%LOCALAPPDATA%\hermes`) |

---

## Method 1: One-Line Installer (Recommended)

### Linux / macOS / WSL2

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

### Windows (PowerShell)

```powershell
irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1 | iex
```

Or download and run with options:

```powershell
.\install.ps1 -NoVenv -SkipSetup
```

The Windows installer installs to `%LOCALAPPDATA%\hermes\hermes-agent` by default and sets the `HERMES_HOME` user environment variable. It uses winget, chocolatey, or scoop to install optional dependencies (ripgrep, ffmpeg).

### What the installer does

1. Installs `uv` if not present (via `https://astral.sh/uv/install.sh`)
2. Installs Python 3.11 via uv (no sudo required)
3. Checks for Git (required)
4. Installs Node.js v22 LTS if not present (binary download to `~/.hermes/node/`)
5. Installs ripgrep and ffmpeg via package manager (optional, asks for sudo if needed)
6. Clones the repository (tries SSH first, falls back to HTTPS)
7. Creates a Python virtual environment with `uv venv venv --python 3.11`
8. Installs all dependencies with `uv pip install -e ".[all]"`, falling back to base install (no `mini-swe-agent` submodule required as of v0.5.0 -- the Docker and Modal backends are now inlined)
9. Installs Node.js dependencies and Playwright Chromium for browser tools
12. Symlinks the `hermes` command into `~/.local/bin`
13. Adds `~/.local/bin` to your shell's PATH if not already there
14. Creates the `~/.hermes/` directory structure with config templates
15. Copies `.env.example` to `~/.hermes/.env` and `cli-config.yaml.example` to `~/.hermes/config.yaml`
16. Creates `~/.hermes/SOUL.md` persona template
17. Syncs bundled skills to `~/.hermes/skills/`
18. Prompts you to run the setup wizard
19. Optionally installs the gateway as a background service

### Installer options (Linux/macOS)

```
--no-venv      Don't create virtual environment
--skip-setup   Skip interactive setup wizard
--branch NAME  Git branch to install (default: main)
--dir PATH     Installation directory (default: ~/.hermes/hermes-agent)
```

### Installer options (Windows PowerShell)

```
-NoVenv        Don't create virtual environment
-SkipSetup     Skip interactive setup wizard
-Branch NAME   Git branch to install (default: main)
-HermesHome    Override home directory (default: %LOCALAPPDATA%\hermes)
-InstallDir    Override install directory (default: %LOCALAPPDATA%\hermes\hermes-agent)
```

After installation, reload your shell and start chatting:

```bash
source ~/.bashrc   # or: source ~/.zshrc
hermes
```

---

## Method 2: setup-hermes.sh Script

For developers who clone the repository manually, `setup-hermes.sh` in the repo root performs the same dependency setup without fetching the repo from the internet:

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh
```

The script performs these steps in order:

1. Checks for `uv` in `PATH`, `~/.local/bin/uv`, and `~/.cargo/bin/uv` -- installs via `curl -LsSf https://astral.sh/uv/install.sh | sh` if missing
2. Checks for Python 3.11 via `uv python find 3.11` -- installs via `uv python install 3.11` if missing
3. Removes any existing `venv` directory and creates a fresh one with `uv venv venv --python 3.11`
4. Sets `VIRTUAL_ENV` so uv installs into the correct venv
5. Installs all extras with `uv pip install -e ".[all]"`, falling back to `uv pip install -e "."` on failure
6. Optionally installs the `tinker-atropos` submodule package for RL training (the `mini-swe-agent` submodule was removed in v0.5.0; Docker and Modal backends are now inlined)
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

Clone the repository. The `mini-swe-agent` submodule is no longer required as of v0.5.0 (Docker and Modal backends are now inlined). Only `tinker-atropos` remains as an optional submodule for RL training.

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
```

To install the optional RL training backend later:

```bash
git submodule update --init tinker-atropos
uv pip install -e ./tinker-atropos
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
| `all` | All extras combined (excludes `rl` and `yc-bench`) | `uv pip install -e ".[all]"` |
| `messaging` | Telegram, Discord, Slack gateway | `uv pip install -e ".[messaging]"` |
| `cron` | Cron expression parsing | `uv pip install -e ".[cron]"` |
| `cli` | Terminal menu UI for setup wizard | `uv pip install -e ".[cli]"` |
| `modal` | Modal cloud execution backend | `uv pip install -e ".[modal]"` |
| `daytona` | Daytona cloud sandbox backend | `uv pip install -e ".[daytona]"` |
| `tts-premium` | ElevenLabs premium TTS voices | `uv pip install -e ".[tts-premium]"` |
| `voice` | CLI microphone input and audio playback | `uv pip install -e ".[voice]"` |
| `pty` | PTY terminal support | `uv pip install -e ".[pty]"` |
| `honcho` | Honcho AI-native memory (`honcho-ai>=2.0.1`) | `uv pip install -e ".[honcho]"` |
| `mcp` | Model Context Protocol support (`mcp>=1.2.0`) | `uv pip install -e ".[mcp]"` |
| `homeassistant` | Home Assistant integration | `uv pip install -e ".[homeassistant]"` |
| `sms` | SMS messaging | `uv pip install -e ".[sms]"` |
| `acp` | ACP editor integration (`agent-client-protocol>=0.8.1,<1.0`) | `uv pip install -e ".[acp]"` |
| `slack` | Slack messaging | `uv pip install -e ".[slack]"` |
| `matrix` | Matrix messaging (`matrix-nio[e2e]>=0.24.0`) | `uv pip install -e ".[matrix]"` |
| `dev` | pytest, pytest-asyncio, pytest-xdist, mcp | `uv pip install -e ".[dev]"` |
| `rl` | RL training (Atropos, Tinker, FastAPI, WandB) | `uv pip install -e ".[rl]"` |

Combine extras: `uv pip install -e ".[messaging,cron,honcho]"`

### Step 5: Install Submodule Packages (Optional)

The `mini-swe-agent` submodule was removed in v0.5.0; the Docker and Modal terminal backends are now inlined in the main package. Only the RL training backend remains as an optional submodule:

```bash
# RL training backend (optional, heavy)
git submodule update --init tinker-atropos
uv pip install -e "./tinker-atropos"
```

Without `tinker-atropos`, RL training tools are unavailable. All other terminal toolsets are available without submodules.

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
# Required -- at least one LLM provider:
OPENROUTER_API_KEY=sk-or-v1-your-key-here

# Optional -- enable additional tools:
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

## Method 4: Nix / NixOS (v0.5.0)

A full Nix flake is available as of v0.5.0, contributed by @alt-glitch. It provides a uv2nix build, a NixOS module with two service modes, auto-generated config keys from Python source, and suffix PATHs so agent-installed tools (apt, pip, npm) take priority over Nix store paths.

Supported systems: `x86_64-linux`, `aarch64-linux`, `aarch64-darwin`.

### Run Without Installing

```bash
nix run github:NousResearch/hermes-agent
```

### NixOS Module

Add to your NixOS configuration:

```nix
{
  inputs.hermes-agent.url = "github:NousResearch/hermes-agent";

  outputs = { nixpkgs, hermes-agent, ... }: {
    nixosConfigurations.mymachine = nixpkgs.lib.nixosSystem {
      modules = [
        hermes-agent.nixosModules.default
        {
          services.hermes-agent = {
            enable = true;
            settings.model = "anthropic/claude-sonnet-4";
            environmentFiles = [ config.sops.secrets."hermes/env".path ];
          };
        }
      ];
    };
  };
}
```

Key NixOS module options:

| Option | Default | Description |
|--------|---------|-------------|
| `settings` | `{}` | Declarative config.yaml (attrset, deep-merged) |
| `environmentFiles` | `[]` | Paths to secret env files (API keys, tokens) |
| `environment` | `{}` | Non-secret environment variables |
| `stateDir` | `/var/lib/hermes` | State directory (`HERMES_HOME` lives here) |
| `workingDirectory` | `stateDir/workspace` | Agent working directory (`MESSAGING_CWD`) |
| `mcpServers` | `{}` | Declarative MCP server configurations |
| `addToSystemPackages` | `false` | Add `hermes` CLI to system PATH |
| `container.enable` | `false` | Enable OCI container mode (see below) |

### Persistent Container Mode

When `container.enable = true`, the NixOS module runs Hermes inside an OCI container (Ubuntu 24.04 by default, Docker or Podman backend). The Nix store is bind-mounted read-only; a persistent writable layer survives restarts and agent updates. On first boot, the container provisions nodejs/npm via apt, uv via curl, and a Python 3.11 venv so the agent can install packages at runtime (`npm i -g`, `pip install`, `uv tool install`). Environment variables are written to `$HERMES_HOME/.env` and read at startup -- no container recreation is needed for env changes.

```nix
services.hermes-agent = {
  enable = true;
  container = {
    enable = true;
    backend = "docker";  # or "podman"
    image = "ubuntu:24.04";
  };
};
```

### Suffix PATHs for Agent-Friendliness

The Nix wrapper uses `--suffix PATH` (not `--prefix`) so that apt/uv/npm-installed tools take priority over Nix store paths. This allows the agent to install and immediately use tools at runtime without rebuilding the derivation.

---

## Method 5: Docker Container (v0.6.0)

Introduced in **v0.6.0 (v2026.3.30)**. ([#3668](https://github.com/NousResearch/hermes-agent/pull/3668))

The official Dockerfile builds a self-contained image based on `debian:13.4`. It installs all Python and Node.js dependencies, Playwright Chromium, and the WhatsApp bridge. The entrypoint is `docker/entrypoint.sh`, which bootstraps the config directory and then passes all arguments to `hermes`.

Data is stored inside the container at `/opt/data` (`HERMES_HOME=/opt/data`). Mount a local directory to this path to persist config, memory, sessions, and skills across container restarts.

The Dockerfile does not declare a fixed `EXPOSE` port — the gateway port is runtime-configurable via `API_SERVER_PORT` (default: `8642`).

### Build the image

From the repository root:

```bash
docker build -t hermes-agent .
```

### CLI mode

Run a one-off chat session with your local `~/.hermes` config mounted:

```bash
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  hermes-agent hermes chat
```

### Gateway mode

Run the gateway as a background service:

```bash
docker run -d --name hermes-gateway \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  hermes-agent hermes gateway start
```

This mounts your existing `~/.hermes` directory so the container reads your `config.yaml`, `.env` (API keys and bot tokens), and skills. All data written by the gateway (sessions, logs, cron) persists in `~/.hermes` on the host.

### Notes

- **Data persistence** — all state is stored in the volume-mounted directory. Removing the container does not affect your config or history.
- **Podman compatible** — substitute `podman` for `docker` in every command above. No other changes are required.
- **First run** — if `~/.hermes` does not yet exist, the entrypoint creates the directory structure and copies config templates automatically.

---

## Supply Chain Note (v0.5.0)

v0.5.0 includes a comprehensive supply chain audit:

- **`litellm`, `typer`, and `platformdirs` removed** from dependencies due to supply chain concerns
- **All dependency version ranges pinned** (e.g. `openai>=2.21.0,<3`)
- **`uv.lock` regenerated with hashes** and used by the installer for reproducible installs
- **CI workflow added** to scan incoming PRs for supply chain attack patterns

These changes mean the dependency set is smaller, faster to install, and more auditable. If you maintained a local patch or override for `litellm`, remove it -- Hermes no longer uses it.

---

## Setup Wizard

After any installation method, run the interactive setup wizard to configure your provider, terminal backend, messaging platforms, and tools:

```bash
hermes setup
```

The wizard is modular -- you can run individual sections at any time:

```bash
hermes model          # Choose LLM provider and model
hermes tools          # Configure which tools are enabled
hermes gateway setup  # Set up messaging platforms
hermes config set     # Set individual config values
```

Wizard sections (defined in `hermes_cli/setup.py`):

1. **Model and Provider** -- choose your AI provider and model
2. **Terminal Backend** -- where the agent runs commands (`local`, `docker`, `ssh`, `modal`, `daytona`, `singularity`)
3. **Messaging Platforms** -- connect Telegram, Discord, Slack, WhatsApp, Signal, Email, or Home Assistant
4. **Tools** -- configure TTS, web search, image generation, and other tool-specific settings
5. **Agent Settings** -- max iterations, context compression, session reset behavior

---

## Provider Credential Configuration

### Supported Providers and Their Credentials

| Provider | Auth Method | Credential |
|----------|-------------|-----------|
| **Nous Portal** | OAuth device code via `hermes model` | Stored in `~/.hermes/auth.json` |
| **OpenAI Codex** | OAuth external via `hermes model` | Stored in `~/.hermes/auth.json` |
| **Anthropic** | Claude Code auto-discovery or API key | `ANTHROPIC_API_KEY`, `ANTHROPIC_TOKEN`, or `CLAUDE_CODE_OAUTH_TOKEN` |
| **GitHub Copilot** | GitHub token | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, or `GITHUB_TOKEN` |
| **GitHub Copilot ACP** | External process (editor-managed) | `COPILOT_ACP_BASE_URL` |
| **OpenRouter** | API key | `OPENROUTER_API_KEY` |
| **Z.AI / GLM** | API key | `GLM_API_KEY`, `ZAI_API_KEY`, or `Z_AI_API_KEY` |
| **Kimi / Moonshot** | API key | `KIMI_API_KEY` |
| **MiniMax** | API key | `MINIMAX_API_KEY` |
| **MiniMax China** | API key | `MINIMAX_CN_API_KEY` |
| **DeepSeek** | API key | `DEEPSEEK_API_KEY` |
| **Alibaba Cloud (DashScope)** | API key | `DASHSCOPE_API_KEY` |
| **AI Gateway** | API key | `AI_GATEWAY_API_KEY` |
| **Kilo Code** | API key | `KILOCODE_API_KEY` |
| **OpenCode Zen** | API key | `OPENCODE_ZEN_API_KEY` |
| **OpenCode Go** | API key | `OPENCODE_GO_API_KEY` |
| **Custom Endpoint** | Base URL plus API key | `OPENAI_BASE_URL` and `OPENAI_API_KEY` |

For local models (Ollama, vLLM, llama.cpp, SGLang):

```bash
hermes config set OPENAI_BASE_URL http://localhost:11434/v1
hermes config set OPENAI_API_KEY ollama
hermes config set HERMES_MODEL llama3.1
```

### Credential Priority Order

The runtime provider resolver resolves credentials in this order:

1. Explicit CLI argument (`--provider`, `--api-key`)
2. `model.provider` in `~/.hermes/config.yaml`
3. `HERMES_INFERENCE_PROVIDER` environment variable
4. Auto-detection from available env vars and the auth store

For OpenRouter, when the resolved base URL contains `openrouter.ai`, `OPENROUTER_API_KEY` is preferred over `OPENAI_API_KEY` to prevent key leakage to unrelated providers. For custom endpoints, `OPENAI_API_KEY` takes precedence.

---

## Entry Points

The `pyproject.toml` defines three entry points:

| Command | Entry point | Purpose |
|---------|-------------|---------|
| `hermes` | `hermes_cli.main:main` | Main CLI (chat, setup, config, gateway, etc.) |
| `hermes-agent` | `run_agent:main` | Direct agent invocation |
| `hermes-acp` | `acp_adapter.entry:main` | ACP editor server |

---

## Verifying the Installation

```bash
hermes version    # Print the installed version
hermes doctor     # Run diagnostics -- reports what is working or missing
hermes status     # Show current configuration summary
hermes chat -q "Hello! What tools do you have available?"
```

`hermes doctor` checks for missing config keys, unavailable tools, provider connectivity, submodule status, Skills Hub state, and Honcho configuration. Use `hermes doctor --fix` to auto-fix what is possible (creates missing directories, config files, etc.).

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

# Install everything (mini-swe-agent submodule no longer required as of v0.5.0)
uv pip install -e ".[all]"
uv pip install -e "./tinker-atropos"  # optional: RL training only
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

This pulls the latest code, updates dependencies, and prompts you to configure any new options added since your last update. The installer script handles existing installations gracefully -- it detects the `.git` directory, stashes local changes, pulls updates, and offers to restore the stash.

If you skipped the post-update prompt:

```bash
hermes config check    # Show missing config options
hermes config migrate  # Interactively add missing options
```

For manual (non-installer) installations:

```bash
cd /path/to/hermes-agent
export VIRTUAL_ENV="$(pwd)/venv"

git pull origin main

uv pip install -e ".[all]"

hermes config check
hermes config migrate
```

You can also update from a connected messaging platform by sending `/update` to the bot.

---

## Troubleshooting Installation Issues

| Problem | Solution |
|---------|----------|
| `hermes: command not found` | Reload shell: `source ~/.bashrc` or `source ~/.zshrc`. Verify `~/.local/bin` is on PATH with `echo $PATH`. |
| Python version too old | Run `uv python install 3.11`. Hermes requires Python 3.11+ (`requires-python = ">=3.11"` in pyproject.toml). |
| `uv: command not found` | Run `curl -LsSf https://astral.sh/uv/install.sh \| sh && source ~/.bashrc` |
| Permission denied during install | Do not use `sudo`. The installer writes to `~/.local/bin` and requires no root access. |
| `API key not set` | Run `hermes model` to configure your provider, or `hermes config set OPENROUTER_API_KEY your_key` |
| Missing config after update | Run `hermes config check` then `hermes config migrate` |
| Node.js not found | Install Node.js v22 from nodejs.org, or re-run the installer which installs it automatically to `~/.hermes/node/` |
| Windows git `copy-fd` error | The PowerShell installer sets `windows.appendAtomically=false` automatically. If cloning manually, run `git config --global windows.appendAtomically false` |
| Windows: git clone fails entirely | The PowerShell installer has a ZIP fallback. Download the repo ZIP from GitHub and extract it manually. |

For detailed diagnostics, run `hermes doctor` -- it reports exactly what is missing and how to fix it.

---

## Directory Structure After Installation

```
~/.hermes/                        # HERMES_HOME
  .env                            # API keys and secrets
  config.yaml                     # Agent configuration
  auth.json                       # OAuth credentials (Nous Portal, Codex)
  SOUL.md                         # Agent persona customization
  state.db                        # SQLite session database
  cron/                           # Scheduled job data
  sessions/                       # JSON session logs
  logs/                           # Gateway and service logs
  memories/                       # MEMORY.md, USER.md
  skills/                         # Active skills (bundled + hub-installed)
  pairing/                        # Gateway DM pairing data
  hooks/                          # Lifecycle hooks
  image_cache/                    # Cached generated images
  audio_cache/                    # Cached TTS audio
  whatsapp/session/               # WhatsApp bridge credentials
  skins/                          # Custom UI themes (YAML)
  hermes-agent/                   # Source code (installer installs here)
```

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
rm -rf ~/.hermes    # Optional -- keep if you plan to reinstall
```

If the gateway is installed as a system service, stop it first:

```bash
hermes gateway stop
# Linux:
systemctl --user disable hermes-gateway
# macOS:
launchctl remove ai.hermes.gateway
```
