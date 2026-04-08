# FAQ & Troubleshooting

Quick answers and fixes for the most common questions and issues.

---

## Frequently Asked Questions

### What LLM providers work with Hermes?

Hermes Agent works with any OpenAI-compatible API. Supported providers include:

- **OpenRouter** -- access hundreds of models through one API key (recommended for flexibility)
- **Nous Portal** -- Nous Research's own inference endpoint
- **OpenAI** -- GPT-4o, o1, o3, etc.
- **Anthropic** -- Claude models (via OpenRouter or compatible proxy)
- **Google** -- Gemini models (via OpenRouter or compatible proxy)
- **z.ai / ZhipuAI** -- GLM models
- **Kimi / Moonshot AI** -- Kimi models
- **MiniMax** -- global and China endpoints
- **Hugging Face** -- open models via unified OpenAI-compatible endpoint
- **DeepSeek** -- direct DeepSeek access
- **Alibaba Cloud DashScope** -- Qwen models
- **OpenCode Zen / Go** -- curated and open model access
- **GitHub Copilot** -- via Copilot ACP
- **Kilo Code** -- Kilo Code API
- **Local models** -- via Ollama, vLLM, llama.cpp, SGLang, or any OpenAI-compatible server

Set your provider with `hermes model` or by editing `~/.hermes/.env`. See the [Environment Variables](./environment-variables.md) reference for all provider keys.

### Does it work on Windows?

Not natively. Hermes Agent requires a Unix-like environment. On Windows, install WSL2 and run Hermes from inside it. The standard install command works in WSL2:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

### Is my data sent anywhere?

API calls go only to the LLM provider you configure (e.g., OpenRouter, your local Ollama instance). Hermes Agent does not collect telemetry, usage data, or analytics. Your conversations, memory, and skills are stored locally in `~/.hermes/`.

### Can I use it offline / with local models?

Yes. Run `hermes model`, select Custom endpoint, and enter your server's URL:

```bash
hermes model
# Select: Custom endpoint (enter URL manually)
# API base URL: http://localhost:11434/v1
# API key: ollama
# Model name: qwen3.5:27b
# Context length: 32768
```

Or configure directly in `config.yaml`:

```yaml
model:
  default: qwen3.5:27b
  provider: custom
  base_url: http://localhost:11434/v1
```

This works with Ollama, vLLM, llama.cpp server, SGLang, LocalAI, and others. If your local server has exactly one model loaded, `/model custom` auto-detects it.

For Ollama users: if you set a custom `num_ctx` (e.g., `ollama run --num_ctx 16384`), make sure to set the matching context length in Hermes -- Ollama's `/api/show` reports the model's maximum context, not the effective `num_ctx` you configured.

### How much does it cost?

Hermes Agent itself is free and open-source (MIT license). You pay only for the LLM API usage from your chosen provider. Local models are completely free to run.

### Can multiple people use one instance?

Yes. The messaging gateway lets multiple users interact with the same Hermes Agent instance via Telegram, Discord, Slack, WhatsApp, or Home Assistant. Access is controlled through allowlists (specific user IDs) and DM pairing (first user to message claims access).

### What's the difference between memory and skills?

- **Memory** stores facts -- things the agent knows about you, your projects, and preferences. Memories are retrieved automatically based on relevance.
- **Skills** store procedures -- step-by-step instructions for how to do things. Skills are recalled when the agent encounters a similar task.

Both persist across sessions.

### Can I use it in my own Python project?

Yes. Import the `AIAgent` class and use Hermes programmatically:

```python
from hermes.agent import AIAgent

agent = AIAgent(model="openrouter/nous/hermes-3-llama-3.1-70b")
response = agent.chat("Explain quantum computing briefly")
```

---

## Troubleshooting

### Installation Issues

#### `hermes: command not found` after installation

The shell hasn't reloaded the updated PATH.

```bash
source ~/.bashrc    # bash
source ~/.zshrc     # zsh
```

If it still doesn't work, verify: `which hermes` or `ls ~/.local/bin/hermes`. The installer adds `~/.local/bin` to your PATH. For non-standard shell configs, add `export PATH="$HOME/.local/bin:$PATH"` manually.

#### Python version too old

Hermes requires Python 3.11 or newer (`requires-python = ">=3.11"` in `pyproject.toml`).

```bash
python3 --version
sudo apt install python3.12   # Ubuntu/Debian
brew install python@3.12      # macOS
```

#### `uv: command not found`

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
```

#### Permission denied errors during install

Don't use sudo with the installer -- it installs to `~/.local/bin`. If you previously installed with sudo, clean up with `sudo rm /usr/local/bin/hermes` and re-run the standard installer.

---

### Provider & Model Issues

#### API key not working

```bash
hermes config show       # Check configuration
hermes model             # Re-configure provider
hermes config set OPENROUTER_API_KEY sk-or-v1-xxxxxxxxxxxx
```

Make sure the key matches the provider. Check `~/.hermes/.env` for conflicting entries.

#### Model not available / model not found

```bash
hermes model                                               # List available models
hermes config set HERMES_MODEL openrouter/nous/hermes-3-llama-3.1-70b
hermes chat --model openrouter/meta-llama/llama-3.1-70b-instruct
```

#### Rate limiting (429 errors)

Wait a moment and retry. For sustained usage: upgrade your provider plan, switch to a different model or provider, or use `hermes chat --provider <alternative>`.

#### Context length exceeded

```bash
/compress                   # Compress the current session
hermes chat                 # Start fresh
```

If this happens on the first long conversation, Hermes may have the wrong context length for your model. Set it explicitly:

```yaml
model:
  default: your-model-name
  context_length: 131072
```

---

### Terminal Issues

#### Command blocked as dangerous

Hermes detected a potentially destructive command. Review the command and type `y` to approve it. Ask the agent to use a safer alternative if preferred.

#### `sudo` not working via messaging gateway

The messaging gateway runs without an interactive terminal, so `sudo` cannot prompt for a password. Avoid `sudo` in messaging, configure passwordless sudo for specific commands, or switch to the terminal interface.

#### Docker backend not connecting

```bash
docker info                              # Check Docker is running
sudo usermod -aG docker $USER           # Add user to docker group
newgrp docker
```

---

### Messaging Issues

#### Bot not responding to messages

```bash
hermes gateway status          # Check if running
hermes gateway start           # Start the gateway
cat ~/.hermes/logs/gateway.log | tail -50
```

#### Allowlist confusion -- who can talk to the bot?

| Mode | How it works |
|------|-------------|
| Allowlist | Only user IDs listed in config can interact |
| DM pairing | First user to message in DM claims exclusive access |
| Open | Anyone can interact (not recommended for production) |

Configure in `~/.hermes/config.yaml` under your gateway's settings.

#### macOS: Node.js / ffmpeg / other tools not found by gateway

launchd services inherit a minimal PATH. Re-run `hermes gateway install` to capture your current PATH, then `hermes gateway start`.

---

### Performance Issues

#### Slow responses

- Try a faster/smaller model
- Reduce active toolsets: `hermes chat -t "terminal"`
- Check network latency to the provider
- For local models, ensure sufficient GPU VRAM

#### High token usage

```bash
/compress          # Compress the conversation
/usage             # Check session token usage
```

Use `/compress` regularly during long sessions.

---

### MCP Issues

#### MCP server not connecting

```bash
node --version     # Ensure Node.js is available
npx --version
```

Verify `~/.hermes/config.yaml` MCP configuration. Test the server manually.

#### Tools not showing up from MCP server

- Check logs for MCP connection errors
- Ensure the server responds to `tools/list`
- Review `tools.include`, `tools.exclude`, `tools.resources`, `tools.prompts`, and `enabled` settings
- Use `/reload-mcp` after changing config

---

## Profiles

### How do profiles differ from just setting HERMES_HOME?

Profiles are a managed layer on top of `HERMES_HOME`. They handle directory structure creation, shell alias generation, active profile tracking, skill syncing, and tab completion integration.

### Can two profiles share the same bot token?

No. Each messaging platform requires exclusive access to a bot token. Create a separate bot per profile.

### Do profiles share memory or sessions?

No. Each profile has its own memory store, session database, and skills directory. Use `hermes profile create newname --clone-all` to copy everything from an existing profile.

### What happens when I run `hermes update`?

It pulls latest code and reinstalls dependencies once (not per-profile), then syncs updated skills to all profiles automatically.

### Can I move a profile to a different machine?

Yes. Export with `hermes profile export work ./work-backup.tar.gz`, copy, then import with `hermes profile import ./work-backup.tar.gz work`.

### How many profiles can I run?

No hard limit. Each profile is a directory under `~/.hermes/profiles/`. Idle profiles use no resources.

---

## Still Stuck?

1. Search existing issues: [GitHub Issues](https://github.com/NousResearch/hermes-agent/issues)
2. Ask the community: [Nous Research Discord](https://discord.gg/nousresearch)
3. File a bug report: include your OS, Python version, Hermes version, and the full error message
