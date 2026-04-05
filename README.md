# Hermes Agent

**The self-improving AI agent built by [Nous Research](https://nousresearch.com).**

Hermes Agent is an autonomous AI agent with a built-in learning loop. It creates skills from experience, improves them during use, searches its own past conversations for context, and builds a deepening model of who you are across sessions. It runs on a $5 VPS, a GPU cluster, or serverless infrastructure -- not tied to your laptop.

Current version: **v0.7.0 (v2026.4.3)** -- released April 3, 2026.

---

## What Hermes Agent Is

Hermes Agent is not a coding copilot or a chatbot wrapper. It is a multi-platform, multi-provider autonomous agent that:

- Executes tool calls sequentially or concurrently via `ThreadPoolExecutor` (up to 8 parallel workers)
- Manages long-running conversations with context compression and session lineage
- Persists memory, skills, and session history in SQLite across restarts
- Streams responses token-by-token from the model to the user interface (v0.3.0)
- Runs as a CLI, a messaging gateway (Telegram, Discord, Slack, WhatsApp, Signal, DingTalk, SMS, Mattermost, Matrix, Webhook, Email, Home Assistant, Feishu/Lark, WeCom), or an IDE integration (VS Code, Zed, JetBrains via ACP)
- Exposes an OpenAI-compatible `/v1/chat/completions` API server with REST cron job management (v0.4.0)
- Delegates work to isolated subagents and spawns scheduled cron jobs
- Exports training trajectories for SFT data generation and RL fine-tuning

The agent core is `AIAgent` in `run_agent.py`. All other subsystems -- gateway, CLI, ACP server, cron scheduler -- use this single agent core, so behavior is consistent across all platforms.

---

## Key Features

### Unified Streaming (v0.3.0)

Real-time token-by-token delivery in the CLI and all gateway platforms. Responses stream as they are generated instead of arriving as a single block. Supported on Telegram, Discord, Slack, and the interactive CLI. WhatsApp, Signal, Email, and Home Assistant fall back to non-streaming automatically (no message-edit API). Implemented in `_interruptible_streaming_api_call()` with graceful fallback to non-streaming on any error.

### Closed Learning Loop

- **Persistent memory** -- agent-curated facts written to `MEMORY.md` with periodic nudges to persist durable knowledge
- **Pluggable memory providers** -- the built-in memory layer can now be extended with released memory-provider plugins such as Honcho and other provider backends (v0.7.0)
- **Autonomous skill creation** -- after complex tasks (5+ tool calls), the agent creates reusable skill documents
- **Skill self-improvement** -- skills are patched during use when they are outdated, incomplete, or wrong
- **FTS5 session search** -- full-text search across all past sessions with LLM summarization for cross-session recall
- **Honcho integration** -- dialectic user modeling that builds a persistent model of who you are across sessions

### Multi-Platform Messaging Gateway

Telegram, Discord, Slack, WhatsApp, Signal, DingTalk, SMS (Twilio), Mattermost, Matrix, Webhook, Email (IMAP/SMTP), Home Assistant, Feishu/Lark, and WeCom -- all from a single gateway process. Six new adapters were added in v0.4.0; Feishu/Lark and WeCom were added in v0.6.0. Unified session management, media attachments, voice transcription, and per-platform tool configuration. The gateway auto-reconnects failed platforms with exponential backoff (v0.4.0).

### Plugin Architecture (v0.3.0)

Drop Python files into `~/.hermes/plugins/` to extend Hermes with custom tools, commands, and hooks. No forking required.

### Provider Flexibility

Use any model without code changes. Supported providers:

- **Nous Portal** -- first-class provider; 400+ models via Nous inference (`https://inference-api.nousresearch.com/v1`)
- **OpenRouter** -- 200+ models (`https://openrouter.ai/api/v1`)
- **Anthropic** (native, v0.3.0) -- direct API with Claude Code credential auto-discovery, OAuth PKCE flows, and native prompt caching
- **OpenAI** -- GPT-5 variants via chat completions or Codex Responses API
- **Vercel AI Gateway** (v0.3.0) -- access Vercel's model catalog and infrastructure (`https://ai-gateway.vercel.sh/v1`)
- **GitHub Copilot** (v0.4.0) -- OAuth auth, 400k context, token validation
- **Alibaba Cloud / DashScope** (v0.4.0) -- DashScope v1 runtime
- **Kilo Code** (v0.4.0) -- direct API-key provider
- **OpenCode Zen / OpenCode Go** (v0.4.0) -- provider backends
- **Hugging Face** (v0.5.0) -- HF Inference API with curated agentic model picker and live `/models` probe
- **DeepSeek** -- direct API-key provider (`https://api.deepseek.com/v1`)
- **z.ai/GLM**, **Kimi/Moonshot**, **MiniMax** -- direct API-key providers
- **Custom endpoints** -- any OpenAI-compatible API

Switch with `hermes model` -- no code changes, no lock-in.

### OpenAI-Compatible API Server (v0.4.0)

Expose Hermes as an `/v1/chat/completions` and `/v1/responses` endpoint with a `/api/jobs` REST API for cron job management. Hardened with input limits, field whitelists, SQLite-backed response persistence across restarts, CORS origin protection, and `Idempotency-Key` support (v0.5.0). In v0.7.0 the API server added real-time tool progress streaming plus optional session continuity for chat-completions clients via the `X-Hermes-Session-Id` header. The API server exposes its own `hermes-api-server` toolset.

### Credential Pools (v0.7.0)

Register multiple credentials for the same provider and let Hermes rotate within that provider before falling back elsewhere. Released v0.7.0 adds `least_used` pool selection, 401 refresh-and-rotate behavior, and smarter recovery that preserves pool state through fallback routing and restores the primary runtime on later turns.

### Camofox Browser Backend (v0.7.0)

Hermes can route browser tools through a local Camofox anti-detection backend by setting `CAMOFOX_URL`. This adds persistent local browser sessions, VNC URL discovery for visual debugging, and local-backend browsing without the normal remote-backend private-URL restrictions.

### @ Context References (v0.4.0)

Claude Code-style `@file` and `@url` context injection with tab completions in the CLI. Files and web pages are inserted directly into the conversation. Access is restricted to the workspace -- reading secrets outside the workspace is blocked.

### MCP Server Management CLI (v0.4.0)

`hermes mcp` commands for installing, configuring, and authenticating MCP servers, with a full OAuth 2.1 PKCE flow for remote MCP servers. MCP servers are exposed as standalone toolsets and are configurable interactively in `hermes tools`.

### Plugin Lifecycle Hooks (v0.5.0)

`pre_llm_call`, `post_llm_call`, `on_session_start`, and `on_session_end` hooks now fire in the agent loop and CLI/gateway, completing the plugin hook system. Plugins can also register slash commands and extend the TUI.

### Nix Flake (v0.5.0)

Full uv2nix build, NixOS module with persistent container mode, auto-generated config keys from Python source, and suffix PATHs for agent-friendliness. See [installation.md](installation.md) for Nix installation details. Contributed by @alt-glitch.

### Profiles — Multi-Instance Hermes (v0.6.0)

Run multiple isolated Hermes instances from the same installation. Each profile gets its own config, memory, sessions, skills, and gateway service. Create with `hermes profile create`, target a single command with `hermes -p <name> ...`, or set the sticky default with `hermes profile use <name>`. Token locks prevent credential collisions between profiles. See [configuration.md](configuration.md) for details.

### MCP Server Mode (v0.6.0)

Expose Hermes sessions to MCP-compatible clients (Claude Desktop, Cursor, VS Code, etc.) via `hermes mcp serve`. In the released v0.6.0 surface, Hermes serves MCP over stdio; Streamable HTTP is part of Hermes' MCP client support, not the `serve` command.

### Docker Container (v0.6.0)

Official Dockerfile for running Hermes Agent in a container. Supports both CLI and gateway modes with volume-mounted config.

### Fallback Provider Chain (v0.6.0)

Configure multiple inference providers with automatic failover via `fallback_providers` in config.yaml. When the primary provider returns errors or is unreachable, Hermes automatically tries the next provider in the chain.

### Concurrent Tool Execution (v0.3.0)

Multiple independent tool calls run in parallel via `ThreadPoolExecutor` (max 8 workers), significantly reducing latency for multi-tool turns. Interactive tools (e.g. `clarify`) force sequential execution. Path-scoped tools (`read_file`, `write_file`, `patch`) run concurrently only when targeting independent paths. Message and result ordering is preserved when reinserting tool responses into conversation history.

### Context Compression

`ContextCompressor` monitors token usage and compresses context when approaching the model's context limit (default threshold: 50% of the context window). The algorithm protects the first `protect_first_n` turns (default: 3) and at least the last `protect_last_n` messages (default: 20), summarizes the middle section via an auxiliary model call (`call_llm(task="compression")`), and sanitizes orphaned tool-call/result pairs. Session lineage is preserved via `parent_session_id` chains in the SQLite state store.

### Six Terminal Backends

Local, Docker, SSH, Daytona, Singularity, and Modal. Daytona and Modal offer serverless persistence -- the environment hibernates when idle and wakes on demand, costing nearly nothing between sessions.

### Persistent Shell Mode (v0.3.0)

Local and SSH terminal backends maintain shell state across tool calls -- `cd`, environment variables, and aliases persist within a session.

### Smart Approvals (v0.3.0)

Codex-inspired approval system that learns which commands are safe and remembers your preferences. `/stop` kills the current agent run immediately.

### Voice Mode (v0.3.0)

Push-to-talk in the CLI, voice notes in Telegram and Discord, Discord voice channel support, and local Whisper transcription via `faster-whisper` when the `voice` extra is installed. Configurable STT backends use the `stt.enabled` config flag.

### MCP Integration

Native MCP client with stdio and HTTP transports, selective tool loading with utility policies, sampling (server-initiated LLM requests), and auto-reload when `mcp_servers` config changes. Tools are prefixed as `mcp_<server>_<tool_name>`.

### ACP Integration (v0.3.0)

VS Code, Zed, and JetBrains connect to Hermes as an agent backend over stdio/JSON-RPC. Full slash command support. The `hermes-acp` entry point runs the ACP server.

### PII Redaction (v0.3.0)

When `privacy.redact_pii` is enabled, personally identifiable information is automatically scrubbed before sending context to LLM providers.

### Secret Redaction

`agent/redact.py` provides regex-based secret redaction for logs and tool output. It masks API keys (OpenAI, GitHub, Slack, Google, AWS, Stripe, SendGrid, HuggingFace, and others), env variable assignments containing secret names, JSON secret fields, Authorization headers, Telegram bot tokens, private key blocks, database connection string passwords, and E.164 phone numbers. Short tokens (< 18 chars) are fully masked; longer tokens preserve the first 6 and last 4 characters. Controlled via `security.redact_secrets` in config.yaml or `HERMES_REDACT_SECRETS` env var. A `RedactingFormatter` log handler applies redaction to all log output.

### Research-Ready

Batch trajectory generation, trajectory compression for training datasets (`trajectory_compressor.py`), and Atropos RL environments including WebResearchEnv, YC-Bench, and Agentic On-Policy Distillation (OPD, v0.3.0). The pipeline supports SFT data generation and GRPO/PPO RL fine-tuning.

### Skills Ecosystem

118 skills (96 bundled + 22 optional) across 26+ categories. Compatible with the [agentskills.io](https://agentskills.io) open standard. Skills support per-platform enable/disable, conditional activation based on tool availability, and prerequisite validation. The Skills Hub integrates with both ClawHub and skills.sh (v0.3.0).

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
| `pydantic>=2.12.5` | Data validation |
| `firecrawl-py>=4.16.0` | Web content extraction |
| `edge-tts>=7.2.7` | Free text-to-speech (no API key needed) |

Optional extras (install with `pip install "hermes-agent[extra]"`):

| Extra | Contents |
|-------|---------|
| `messaging` | Core gateway dependencies for Telegram, Discord, Slack, and shared `aiohttp`-based adapters |
| `voice` | Push-to-talk CLI and voice note transcription (faster-whisper, sounddevice) |
| `mcp` | Model Context Protocol client |
| `honcho` | Honcho AI user modeling |
| `modal` / `daytona` | Serverless sandbox backends |
| `rl` | Atropos RL training integration |
| `acp` | ACP IDE integration server |
| `cron` | Cron job scheduler |
| `tts-premium` | ElevenLabs premium TTS |
| `slack` | Slack Bolt + SDK (standalone, without full `messaging`) |
| `matrix` | Matrix with E2E encryption (matrix-nio) |
| `pty` | PTY terminal backend (ptyprocess / pywinpty) |
| `homeassistant` | Home Assistant integration (aiohttp) |
| `sms` | SMS via Twilio (aiohttp) |
| `dingtalk` | DingTalk enterprise messaging (dingtalk-stream) |
| `feishu` | Feishu/Lark enterprise messaging (lark-oapi) |
| `yc-bench` | YC-Bench evaluation harness (Python 3.12+) |
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
hermes              # Interactive CLI -- start a conversation
hermes model        # Choose your LLM provider and model
hermes tools        # Configure which tools are enabled (curses UI)
hermes setup        # Run the full setup wizard (configures everything at once)
hermes gateway      # Start the messaging gateway (Telegram, Discord, etc.)
hermes mcp          # Install, configure, and authenticate MCP servers (v0.4.0)
hermes claw migrate # Migrate settings and memories from OpenClaw
hermes doctor       # Diagnose configuration issues across all providers
hermes update       # Update to the latest version (with auto-restart for gateway)
```

---

## CLI vs Messaging Quick Reference

Hermes has two entry points: start the terminal UI with `hermes`, or run the gateway and talk to it from Telegram, Discord, Slack, WhatsApp, Signal, or other platforms. Once you are in a conversation, many slash commands are shared across both interfaces.

| Action | CLI | Messaging platforms |
|---------|-----|---------------------|
| Start chatting | `hermes` | Run `hermes gateway setup` + `hermes gateway start`, then send the bot a message |
| Start fresh conversation | `/new` or `/reset` | `/new` or `/reset` |
| Resume a named session | `/resume [session]` (v0.5.0) | `/resume [session]` (v0.5.0) |
| Change model | `hermes model` | `hermes model` |
| Set a personality | `/personality [name]` | `/personality [name]` |
| Retry or undo the last turn | `/retry`, `/undo` | `/retry`, `/undo` |
| Compress context / check usage | `/compress`, `/usage`, `/insights [--days N]` | `/compress`, `/usage`, `/insights [days]` |
| Toggle config status bar | `/statusbar` (v0.4.0) | -- |
| Queue a prompt without interrupting | `/queue [prompt]` (v0.4.0) | `/queue [prompt]` (v0.4.0) |
| Switch approval mode | `/permission [mode]` (v0.4.0) | `/permission [mode]` (v0.4.0) |
| Open an interactive browser session | `/browser` (v0.4.0) | -- |
| Show live cost / usage in gateway | -- | `/cost` (v0.4.0) |
| Approve / deny a pending command | -- | `/approve`, `/deny` (v0.4.0) |
| Toggle tool output verbosity | -- | `/verbose` (v0.5.0) |
| Browse skills | `/skills` or `/<skill-name>` | `/skills` or `/<skill-name>` |
| Interrupt current work | `Ctrl+C` or send a new message | `/stop` or send a new message |
| Platform-specific status | `/platforms` | `/status`, `/sethome` |

---

## Documentation Index

### Core

| Document | Contents |
|----------|----------|
| [architecture.md](architecture.md) | Component diagram, agent loop, context compression, session storage, internal design |
| [changelog.md](changelog.md) | v0.6.0, v0.5.0, v0.4.0, v0.3.0 and v0.2.0 release notes with full feature lists |
| [streaming.md](streaming.md) | Unified streaming infrastructure, token delivery, platform support (v0.3.0) |

### Setup & Configuration

| Document | Contents |
|----------|----------|
| [installation.md](installation.md) | Install methods (one-liner, manual, Windows), prerequisites, setup wizard |
| [configuration.md](configuration.md) | Complete config.yaml reference, env vars, precedence rules |
| [cli-reference.md](cli-reference.md) | All CLI commands, slash commands, flags, environment variables |
| [providers.md](providers.md) | All 17 LLM providers, credential resolution, routing, OAuth flows |
| [profiles.md](profiles.md) | Multi-instance profiles, creation, switching, gateway per-profile (v0.6.0) |

### Features

| Document | Contents |
|----------|----------|
| [plugins.md](plugins.md) | Plugin architecture, discovery, registration, custom tools (v0.3.0) |
| [hooks.md](hooks.md) | Gateway and plugin lifecycle hooks, event types |
| [browser.md](browser.md) | Browser automation, Browserbase, Browser Use, CDP connect |
| [checkpoints.md](checkpoints.md) | Filesystem checkpoints, rollback, git worktree isolation |
| [cron.md](cron.md) | Scheduled task system, schedule formats, job management |
| [voice-mode.md](voice-mode.md) | Voice interaction, STT/TTS providers, Discord voice channels (v0.3.0) |
| [batch-processing.md](batch-processing.md) | Batch trajectory generation for SFT/RL training data |
| [delegation.md](delegation.md) | Subagent spawning, task isolation, parallel workstreams |
| [api-server.md](api-server.md) | OpenAI-compatible HTTP API, endpoints, compatible frontends |
| [skins.md](skins.md) | CLI themes, custom skins, banner configuration |

### Skills & Tools

| Document | Contents |
|----------|----------|
| [skills.md](skills.md) | Skills system, SKILL.md format, 96 bundled + 22 optional skills catalog |
| [tools.md](tools.md) | All 49 built-in tools reference with parameters and toolsets |
| [toolsets.md](toolsets.md) | Toolset definitions, compositions, per-platform configuration |
| [mcp.md](mcp.md) | MCP client integration, stdio/HTTP transports, sampling, tool filtering |

### Security & Memory

| Document | Contents |
|----------|----------|
| [security.md](security.md) | Five-layer security model, PII redaction, approvals, tirith scanning, OAuth |
| [memory.md](memory.md) | MEMORY.md/USER.md system, Honcho integration, session storage |
| [acp.md](acp.md) | ACP server for IDE integration (VS Code, Zed, JetBrains) |

### Gateway & Messaging

| Document | Contents |
|----------|----------|
| [gateway.md](gateway.md) | Gateway architecture, session management, authorization, streaming |
| [messaging/](messaging/) | Per-platform setup guides for all 16 messaging platforms |
| [messaging/telegram.md](messaging/telegram.md) | Telegram Bot API setup, forum topics, voice notes |
| [messaging/discord.md](messaging/discord.md) | Discord bot setup, voice channels, threads |
| [messaging/slack.md](messaging/slack.md) | Slack Bolt + Socket Mode setup |
| [messaging/whatsapp.md](messaging/whatsapp.md) | WhatsApp Baileys bridge setup |
| [messaging/signal.md](messaging/signal.md) | Signal via signal-cli-rest-api |
| [messaging/email.md](messaging/email.md) | Email via IMAP/SMTP |
| [messaging/matrix.md](messaging/matrix.md) | Matrix with E2E encryption |
| [messaging/mattermost.md](messaging/mattermost.md) | Mattermost team chat |
| [messaging/dingtalk.md](messaging/dingtalk.md) | DingTalk enterprise messaging |
| [messaging/homeassistant.md](messaging/homeassistant.md) | Home Assistant smart home |
| [messaging/feishu.md](messaging/feishu.md) | Feishu/Lark enterprise messaging (v0.6.0) |
| [messaging/wecom.md](messaging/wecom.md) | WeCom enterprise messaging (v0.6.0) |
| [messaging/webhook.md](messaging/webhook.md) | Webhook adapter |
| [messaging/open-webui.md](messaging/open-webui.md) | Open WebUI / API server frontend |
| [messaging/sms.md](messaging/sms.md) | SMS via Twilio |

### Feature Deep Dives

| Document | Contents |
|----------|----------|
| [features/code-execution.md](features/code-execution.md) | Sandboxed code execution, Docker/Modal/Daytona backends |
| [features/context-references.md](features/context-references.md) | @file and @url context injection, tab completion |
| [features/credential-pools.md](features/credential-pools.md) | Multi-key credential pooling, rotation, exhaustion tracking |
| [features/fallback-providers.md](features/fallback-providers.md) | Provider failover chains, retry logic, health checks |
| [features/honcho.md](features/honcho.md) | Honcho dialectic user modeling, session mapping, identity |
| [features/image-generation.md](features/image-generation.md) | Image generation via fal.ai, DALL-E, provider routing |
| [features/vision.md](features/vision.md) | Vision analysis, multi-modal input, image attachments |
| [features/tts.md](features/tts.md) | Text-to-speech: Edge TTS, ElevenLabs, NeuTTS |
| [features/personality.md](features/personality.md) | Personality system, SOUL.md, per-profile personas |
| [features/provider-routing.md](features/provider-routing.md) | Smart model routing, prefix matching, provider detection |
| [features/rl-training.md](features/rl-training.md) | Atropos RL integration, environments, trajectory export |
| [features/environments.md](features/environments.md) | RL environments: WebResearchEnv, YC-Bench, OPD |

### Getting Started

| Document | Contents |
|----------|----------|
| [getting-started/quickstart.md](getting-started/quickstart.md) | First conversation in 5 minutes |
| [getting-started/first-config.md](getting-started/first-config.md) | Initial provider and model setup |
| [getting-started/gateway-setup.md](getting-started/gateway-setup.md) | Setting up the messaging gateway |

### Developer Guide

| Document | Contents |
|----------|----------|
| [developer-guide/architecture-deep-dive.md](developer-guide/architecture-deep-dive.md) | Detailed agent internals, call flow, state machine |
| [developer-guide/adding-a-tool.md](developer-guide/adding-a-tool.md) | How to create a new built-in tool |
| [developer-guide/adding-a-platform.md](developer-guide/adding-a-platform.md) | How to create a new gateway platform adapter |
| [developer-guide/adding-a-provider.md](developer-guide/adding-a-provider.md) | How to add a new inference provider |
| [developer-guide/adding-a-skill.md](developer-guide/adding-a-skill.md) | SKILL.md authoring guide, categories, activation |
| [developer-guide/testing.md](developer-guide/testing.md) | Test suite, pytest markers, integration tests |
| [developer-guide/plugin-development.md](developer-guide/plugin-development.md) | Plugin API, hooks, custom commands |
| [developer-guide/mcp-server-development.md](developer-guide/mcp-server-development.md) | Building MCP servers for Hermes |
| [developer-guide/acp-protocol.md](developer-guide/acp-protocol.md) | ACP wire protocol, message types, IDE integration |
| [developer-guide/context-compression.md](developer-guide/context-compression.md) | Compression algorithm, session lineage, tuning |
| [developer-guide/session-storage.md](developer-guide/session-storage.md) | SQLite schema, FTS5 search, session lifecycle |
| [developer-guide/security-model.md](developer-guide/security-model.md) | Approval tiers, PII redaction, secret scanning |
| [developer-guide/batch-trajectories.md](developer-guide/batch-trajectories.md) | Trajectory format, SFT data generation |
| [developer-guide/nix-packaging.md](developer-guide/nix-packaging.md) | Nix flake, NixOS module, uv2nix build |
| [developer-guide/memory-provider-plugin.md](developer-guide/memory-provider-plugin.md) | Memory provider plugin framework (planned, not in v0.6.0) |

### Reference

| Document | Contents |
|----------|----------|
| [reference/config-keys.md](reference/config-keys.md) | Complete config.yaml key reference |
| [reference/env-vars.md](reference/env-vars.md) | All environment variables |
| [reference/slash-commands.md](reference/slash-commands.md) | Complete slash command reference |
| [reference/tool-schemas.md](reference/tool-schemas.md) | JSON schemas for all 49 tools |
| [reference/skill-categories.md](reference/skill-categories.md) | All skill categories and their skills |
| [reference/gateway-events.md](reference/gateway-events.md) | Gateway event types and hook payloads |
| [reference/sqlite-schema.md](reference/sqlite-schema.md) | Session store SQLite schema reference |
| [reference/api-endpoints.md](reference/api-endpoints.md) | API server endpoint reference |
| [reference/mcp-tools.md](reference/mcp-tools.md) | MCP tool naming and filtering reference |
| [reference/provider-aliases.md](reference/provider-aliases.md) | Provider ID aliases and routing |

### Contributing

| Document | Contents |
|----------|----------|
| [contributing.md](contributing.md) | Dev setup, contribution guide, PR process, priorities |

Official documentation: **[hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)**

---

## Community

- Discord: [discord.gg/NousResearch](https://discord.gg/NousResearch)
- Skills Hub: [agentskills.io](https://agentskills.io)
- Issues: [github.com/NousResearch/hermes-agent/issues](https://github.com/NousResearch/hermes-agent/issues)
- Discussions: [github.com/NousResearch/hermes-agent/discussions](https://github.com/NousResearch/hermes-agent/discussions)

---

## License

MIT -- see [LICENSE](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).

Built by [Nous Research](https://nousresearch.com).
