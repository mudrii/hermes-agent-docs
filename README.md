# Hermes Agent

> The self-improving AI agent built by Nous Research. The only agent with a built-in learning loop — it creates skills from experience, improves them during use, nudges itself to persist knowledge, searches its own past conversations, and builds a deepening model of who you are across sessions.

**Current stable release: v0.2.0** (2026-03-12) — [GitHub](https://github.com/NousResearch/hermes-agent)

---

## What Is Hermes Agent?

Hermes Agent is an open-source, self-hosted AI agent platform that runs as a persistent server process. Unlike IDE-bound copilots or basic chatbot wrappers, Hermes lives on your server, remembers what it learns, and gets more capable the longer it runs.

It sits between a Claude Code-style CLI and a messaging platform agent like OpenClaw, combining the terminal-first experience with a multi-platform messaging gateway.

**Positioning (NousResearch's own words):** "It's built in Python, so it's easy for developers to extend. It also powers our agentic RL pipeline."

---

## Key Features

| Feature | Detail |
|---------|--------|
| **Closed learning loop** | Creates skills from experience, improves them during use, searches past conversations, builds user model |
| **Real terminal interface** | Full TUI: multiline editing, slash-command autocomplete, streaming tool output, conversation history |
| **Multi-platform messaging** | Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant — from one gateway process |
| **48 built-in tools** | Web, terminal, browser automation, files, vision, media, scheduling, RL training, smart home |
| **70+ bundled skills** | MLOps, GitHub, productivity, research, creative, security, and more |
| **6 terminal backends** | Local, Docker, SSH, Daytona, Singularity, Modal |
| **Subagent delegation** | Isolated subagents for parallel workstreams; Python RPC |
| **Scheduled automations** | Built-in cron scheduler with delivery to any platform |
| **MCP integration** | Model Context Protocol client (stdio + HTTP) |
| **Editor integration** | VS Code, Zed, JetBrains via ACP protocol |
| **Research-ready** | Batch trajectory generation, Atropos RL environments, trajectory compression |

---

## Three-Level Memory Architecture

Hermes implements a three-tier memory system:

1. **Short-term inference memory** — standard context window for the current session
2. **Procedural Skill Documents** — persistent Markdown files (agentskills.io format) capturing step-by-step solutions to recurring tasks, created autonomously by the agent
3. **Contextual Persistence Layer** — searchable FTS5 index of past sessions, plus optional Honcho user modeling across sessions

---

## Quick Start

### Install

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.zshrc    # or ~/.bashrc
hermes             # start chatting
```

Supports Linux, macOS, and WSL2. Windows native is not supported.

### First Run

```bash
hermes             # Interactive CLI (choose model on first run)
hermes setup       # Full interactive setup wizard
hermes model       # Change LLM provider and model
hermes gateway     # Start messaging platform gateway
```

---

## Providers

Hermes supports multiple LLM providers via [OpenRouter](https://openrouter.ai/) and native integrations:

- **Nous Portal** (recommended) — Claude Opus/Sonnet, GPT-5.4, Gemini 3 Pro
- **OpenRouter** — 200+ models
- **Anthropic** — Claude native API
- **OpenAI** — GPT-5 variants
- **z.ai** — GLM-5, GLM-4.7
- **Kimi/Moonshot** — Kimi K2.5 coding models
- **MiniMax** — M2.5 models
- **Custom endpoints** — Any OpenAI-compatible API

Default model: `anthropic/claude-opus-4.6` via OpenRouter.

---

## v0.2.0 Highlights (2026-03-12)

- 216 PRs merged from **63 contributors**
- **3,289 tests**
- Multi-platform messaging gateway (7 platforms)
- MCP client with automatic reconnection
- Skills ecosystem: 70+ bundled + Skills Hub
- ACP server (VS Code, Zed, JetBrains)
- CLI skin/theme engine (7 skins + custom)
- Git worktree isolation (`hermes -w`)
- Filesystem checkpoints + `/rollback`
- Centralized provider router

---

## Repository Structure

```
hermes-agent/
├── run_agent.py          # AIAgent class — core orchestration loop
├── cli.py                # Interactive terminal UI
├── model_tools.py        # Tool discovery and registry
├── toolsets.py           # Tool grouping by scenario
├── hermes_state.py       # SQLite session state (FTS5)
├── hermes_constants.py   # API endpoint constants
├── hermes_time.py        # Timezone-aware clock
├── batch_runner.py       # RL trajectory generation
├── trajectory_compressor.py  # Training data optimization
├── agent/                # Core agent internals
│   ├── prompt_builder.py
│   ├── auxiliary_client.py
│   ├── context_compressor.py
│   ├── display.py
│   ├── anthropic_adapter.py
│   └── ...
├── tools/                # 48 tool implementations
│   ├── registry.py       # Central tool registry
│   ├── web_tools.py
│   ├── terminal_tool.py
│   ├── browser_tool.py
│   └── environments/     # 6 terminal backends
├── skills/               # 70+ bundled skills
│   └── <category>/<skill-name>/SKILL.md
├── optional-skills/      # 9 official optional skills
├── gateway/              # Messaging platform adapters
├── cron/                 # Scheduled task system
├── honcho_integration/   # Cross-session user memory
├── acp_adapter/          # Editor protocol (ACP)
├── hermes_cli/           # CLI commands, config, auth, setup
├── environments/         # RL/benchmark environments
├── website/docs/         # 74 documentation files
└── tests/                # 3,289 tests
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [Installation](installation.md) | All install methods, setup wizard, first-run guide |
| [CLI Reference](cli-reference.md) | All commands, flags, and slash commands |
| [Configuration](configuration.md) | Full config file reference |
| [Architecture](architecture.md) | System design, agent loop, subsystems |
| [Tools](tools.md) | All 48 built-in tools with parameters |
| [Toolsets](toolsets.md) | Toolset system and platform configurations |
| [Skills](skills.md) | 70+ bundled skills, skill authoring, Skills Hub |
| [Gateway](gateway.md) | Multi-platform messaging setup |
| [LLM Providers](providers.md) | Provider setup, model catalog, routing |
| [Memory](memory.md) | Session storage, Honcho integration, cron |
| [Security](security.md) | Security model and hardening features |
| [ACP Integration](acp.md) | Editor integration (VS Code, Zed, JetBrains) |
| [Contributing](contributing.md) | Development setup, contributing guide |
| [Changelog](changelog.md) | Full release history |

---

## Community

- **GitHub:** https://github.com/NousResearch/hermes-agent
- **Website:** https://hermes-agent.nousresearch.com
- **Docs:** https://hermes-agent.nousresearch.com/docs
- **License:** MIT
