# Hermes Agent

**The self-improving AI agent built by [Nous Research](https://nousresearch.com).**

Hermes Agent is an autonomous AI agent with a built-in learning loop. It creates skills from experience, improves them during use, searches its own past conversations for context, and builds a deepening model of who you are across sessions. It runs on a $5 VPS, a GPU cluster, or serverless infrastructure — not tied to your laptop.

Current version: **v0.3.0 (v2026.3.17)** — released March 17, 2026.

---

## What Hermes Agent Is

Hermes Agent is not a coding copilot or a chatbot wrapper. It is a multi-platform, multi-provider autonomous agent that:

- Executes tool calls sequentially or concurrently via `ThreadPoolExecutor`
- Manages long-running conversations with context compression and session lineage
- Persists memory, skills, and session history in SQLite across restarts
- Streams responses token-by-token from the model to the user interface (v0.3.0)
- Runs as a CLI, a messaging gateway (Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant), or an IDE integration (VS Code, Zed, JetBrains via ACP)
- Delegates work to isolated subagents and spawns scheduled cron jobs
- Exports training trajectories for SFT data generation and RL fine-tuning

The agent core is `AIAgent` in `run_agent.py`. All other subsystems — gateway, CLI, ACP server, cron scheduler — use this single agent core, so behavior is consistent across all platforms.

---

## Key Features

### Unified Streaming (v0.3.0)

Real-time token-by-token delivery in the CLI and all gateway platforms. Responses stream as they are generated instead of arriving as a single block. Supported on Telegram, Discord, Slack, WhatsApp, Signal, and the interactive CLI.

### Closed Learning Loop

- **Persistent memory** — agent-curated facts written to `MEMORY.md` with periodic nudges to persist durable knowledge
- **Autonomous skill creation** — after complex tasks (5+ tool calls), the agent creates reusable skill documents
- **Skill self-improvement** — skills are patched during use when they are outdated, incomplete, or wrong
- **FTS5 session search** — full-text search across all past sessions with LLM summarization for cross-session recall
- **Honcho integration** — dialectic user modeling that builds a persistent model of who you are across sessions

### Multi-Platform Messaging Gateway

Telegram, Discord, Slack, WhatsApp, Signal, Email (IMAP/SMTP), and Home Assistant — all from a single gateway process. Unified session management, media attachments, voice transcription, and per-platform tool configuration.

### Plugin Architecture (v0.3.0)

Drop Python files into `~/.hermes/plugins/` to extend Hermes with custom tools, commands, and hooks. No forking required.

### Provider Flexibility

Use any model without code changes. Supported providers:

- **Nous Portal** — first-class provider; Claude, GPT-5, Gemini via Nous inference
- **OpenRouter** — 200+ models
- **Anthropic** (native, v0.3.0) — direct API with Claude Code credential auto-discovery, OAuth PKCE flows, and native prompt caching
- **OpenAI** — GPT-5 variants via chat completions or Codex Responses API
- **Vercel AI Gateway** (v0.3.0) — access Vercel's model catalog and infrastructure
- **z.ai/GLM**, **Kimi/Moonshot**, **MiniMax** — direct API-key providers
- **Custom endpoints** — any OpenAI-compatible API

Switch with `hermes model` — no code changes, no lock-in.

### Concurrent Tool Execution (v0.3.0)

Multiple independent tool calls run in parallel via `ThreadPoolExecutor`, significantly reducing latency for multi-tool turns. Message and result ordering is preserved when reinserting tool responses into conversation history.

### Context Compression

`ContextCompressor` monitors token usage and compresses context when approaching the model's context limit (default threshold: 50% of the context window). The algorithm protects the first `protect_first_n` turns (default: 3) and the last `protect_last_n` turns (default: 4), summarizes the middle section via an auxiliary model call, and sanitizes orphaned tool-call/result pairs. Session lineage is preserved via `parent_session_id` chains in the SQLite state store.

### Six Terminal Backends

Local, Docker, SSH, Daytona, Singularity, and Modal. Daytona and Modal offer serverless persistence — the environment hibernates when idle and wakes on demand, costing nearly nothing between sessions.

### Persistent Shell Mode (v0.3.0)

Local and SSH terminal backends maintain shell state across tool calls — `cd`, environment variables, and aliases persist within a session.

### Smart Approvals (v0.3.0)

Codex-inspired approval system that learns which commands are safe and remembers your preferences. `/stop` kills the current agent run immediately.

### Voice Mode (v0.3.0)

Push-to-talk in the CLI, voice notes in Telegram and Discord, Discord voice channel support, and local Whisper transcription via faster-whisper. Configurable STT backends with a `stt.enabled` config flag.

### MCP Integration

Native MCP client with stdio and HTTP transports, selective tool loading with utility policies, sampling (server-initiated LLM requests), and auto-reload when `mcp_servers` config changes. Tools are prefixed as `mcp_<server>_<tool_name>`.

### ACP Integration (v0.3.0)

VS Code, Zed, and JetBrains connect to Hermes as an agent backend over stdio/JSON-RPC. Full slash command support. The `hermes-acp` entry point runs the ACP server.

### PII Redaction (v0.3.0)

When `privacy.redact_pii` is enabled, personally identifiable information is automatically scrubbed before sending context to LLM providers.

### Research-Ready

Batch trajectory generation, trajectory compression for training datasets (`trajectory_compressor.py`), and Atropos RL environments including WebResearchEnv, YC-Bench, and Agentic On-Policy Distillation (OPD, v0.3.0). The pipeline supports SFT data generation and GRPO/PPO RL fine-tuning.

### Skills Ecosystem

70+ bundled and optional skills across 15+ categories. Compatible with the [agentskills.io](https://agentskills.io) open standard. Skills support per-platform enable/disable, conditional activation based on tool availability, and prerequisite validation. The Skills Hub integrates with both ClawHub and skills.sh (v0.3.0).

---

## Requirements

- **Python**: 3.11 or newer (`requires-python = ">=3.11"` in `pyproject.toml`)
- **Operating Systems**: Linux, macOS, WSL2. Windows native is not supported; use WSL2.
- **Git**: Required by the installer

Key Python dependencies (from `pyproject.toml`):

| Package | Purpose |
|---------|---------|
| `openai` | OpenAI-compatible API client |
| `anthropic>=0.39.0` | Native Anthropic API client |
| `prompt_toolkit` | Interactive CLI TUI |
| `rich` | Terminal output formatting |
| `pyyaml` | Configuration file parsing |
| `pydantic>=2.0` | Data validation |
| `faster-whisper>=1.0.0` | Local voice transcription |
| `firecrawl-py` | Web content extraction |
| `litellm>=1.75.5` | Terminal backend (mini-swe-agent) |
| `edge-tts` | Free text-to-speech (no API key needed) |

Optional extras (install with `pip install "hermes-agent[extra]"`):

| Extra | Contents |
|-------|---------|
| `messaging` | Telegram, Discord, Slack, WhatsApp gateway |
| `voice` | Push-to-talk CLI and voice note transcription |
| `mcp` | Model Context Protocol client |
| `honcho` | Honcho AI user modeling |
| `modal` / `daytona` | Serverless sandbox backends |
| `rl` | Atropos RL training integration |
| `acp` | ACP IDE integration server |
| `cron` | Cron job scheduler |
| `tts-premium` | ElevenLabs premium TTS |
| `all` | All of the above |

---

## Quick Install

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Works on Linux, macOS, and WSL2. The installer handles Python, Node.js, dependencies, and the `hermes` command. No prerequisites except Git.

After installation:

```bash
source ~/.bashrc    # reload shell (or: source ~/.zshrc)
hermes              # start chatting
```

---

## Getting Started

```bash
hermes              # Interactive CLI — start a conversation
hermes model        # Choose your LLM provider and model
hermes tools        # Configure which tools are enabled (curses UI)
hermes setup        # Run the full setup wizard (configures everything at once)
hermes gateway      # Start the messaging gateway (Telegram, Discord, etc.)
hermes claw migrate # Migrate settings and memories from OpenClaw
hermes doctor       # Diagnose configuration issues across all providers
hermes update       # Update to the latest version (with auto-restart for gateway)
```

---

## Documentation in This Directory

### Core

| Document | Contents |
|----------|----------|
| [README.md](README.md) | This file — overview, features, requirements, install |
| [architecture.md](architecture.md) | Component diagram, agent loop, subsystems, internal design |
| [changelog.md](changelog.md) | v0.3.0 and v0.2.0 release notes with full feature lists |
| [installation.md](installation.md) | Detailed installation and upgrade instructions |
| [configuration.md](configuration.md) | Full config.yaml reference, env vars, precedence rules |
| [cli-reference.md](cli-reference.md) | All CLI commands, slash commands, and flags |
| [providers.md](providers.md) | All 17 providers, credential resolution, routing, custom providers |
| [security.md](security.md) | Approval system, PII redaction, OAuth, secrets, container isolation |
| [contributing.md](contributing.md) | Development setup, test suite, PR process, code style |

### Features

| Document | Contents |
|----------|----------|
| [streaming.md](streaming.md) | Unified streaming infrastructure, token delivery, platform support (v0.3.0) |
| [plugins.md](plugins.md) | Plugin architecture, custom tools/commands/hooks, distribution (v0.3.0) |
| [hooks.md](hooks.md) | Hook types, gateway hooks, plugin hooks, lifecycle |
| [skills.md](skills.md) | Full skills catalog, SKILL.md format, Skills Hub, creating skills |
| [tools.md](tools.md) | All built-in tools, parameters, security notes, custom tools |
| [toolsets.md](toolsets.md) | Toolset definitions, distributions, per-platform config |
| [mcp.md](mcp.md) | MCP client, transports, server config, sampling, reconnection |
| [memory.md](memory.md) | Built-in memory, Honcho integration, recall modes, multi-user isolation |
| [acp.md](acp.md) | ACP IDE integration, VS Code/Zed/JetBrains setup, Copilot auth (v0.3.0) |
| [voice-mode.md](voice-mode.md) | Voice mode, TTS/STT providers, push-to-talk, Discord VC (v0.3.0) |
| [browser.md](browser.md) | Browser tool, CDP connect, web automation, Browserbase (v0.3.0) |
| [checkpoints.md](checkpoints.md) | Filesystem checkpoints, rollback, git worktree isolation |
| [cron.md](cron.md) | Cron job scheduling, schedule formats, delivery, monitoring |
| [batch-processing.md](batch-processing.md) | Batch runner, JSONL datasets, datagen config, trajectory output |
| [delegation.md](delegation.md) | Task delegation, subagent hierarchy, budget sharing |
| [api-server.md](api-server.md) | API server endpoints, streaming, auth, compatible frontends |
| [skins.md](skins.md) | CLI themes, built-in skins, custom skin creation |

### Gateway & Messaging

| Document | Contents |
|----------|----------|
| [gateway.md](gateway.md) | Gateway architecture, session management, approval system |
| [messaging/README.md](messaging/README.md) | Platform overview and comparison table |
| [messaging/telegram.md](messaging/telegram.md) | Telegram bot setup and configuration |
| [messaging/discord.md](messaging/discord.md) | Discord bot setup and configuration |
| [messaging/slack.md](messaging/slack.md) | Slack app setup and configuration |
| [messaging/whatsapp.md](messaging/whatsapp.md) | WhatsApp via Baileys setup |
| [messaging/signal.md](messaging/signal.md) | Signal via signal-cli setup |
| [messaging/email.md](messaging/email.md) | Email (IMAP/SMTP) setup |
| [messaging/matrix.md](messaging/matrix.md) | Matrix via matrix-nio setup |
| [messaging/mattermost.md](messaging/mattermost.md) | Mattermost bot setup |
| [messaging/dingtalk.md](messaging/dingtalk.md) | DingTalk Stream Mode setup |
| [messaging/homeassistant.md](messaging/homeassistant.md) | Home Assistant integration |
| [messaging/open-webui.md](messaging/open-webui.md) | Open WebUI API server mode |
| [messaging/sms.md](messaging/sms.md) | SMS via Twilio setup |

Official documentation: **[hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)**

---

## Community

- Discord: [discord.gg/NousResearch](https://discord.gg/NousResearch)
- Skills Hub: [agentskills.io](https://agentskills.io)
- Issues: [github.com/NousResearch/hermes-agent/issues](https://github.com/NousResearch/hermes-agent/issues)
- Discussions: [github.com/NousResearch/hermes-agent/discussions](https://github.com/NousResearch/hermes-agent/discussions)

---

## License

MIT — see [LICENSE](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).

Built by [Nous Research](https://nousresearch.com).
