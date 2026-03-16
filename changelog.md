# Changelog

All notable changes to Hermes Agent are documented here.

**Current stable release: v0.2.0** (2026-03-12)

---

## [0.2.0] — 2026-03-12

Major public release. 216 merged pull requests from 63 contributors, resolving 119 issues.

**"In just over two weeks, Hermes Agent went from a small internal project to a full-featured AI agent platform — thanks to an explosion of community contributions."**

### Core Platform

**Agent & Orchestration**
- `AIAgent` class: full orchestration engine with provider selection, prompt construction, tool execution, retries, callbacks, and persistence
- ReAct loop (Reason → Act → Observe) with up to 8 parallel tool workers
- Shared `IterationBudget` across parent and subagents — prevents runaway recursion
- `delegate_task` tool for spawning isolated subagents
- `execute_code` tool for Python execution with tool access
- `mixture_of_agents` tool: 5 LLM calls (4 reference + 1 aggregator) for maximum quality

**Centralized Provider Router**
- Unified `call_llm()` / `async_call_llm()` for all LLM consumers
- Per-consumer auxiliary model configuration
- Fallback model for resilience
- Live API validation for model switching
- Per-job model override in cron scheduler

**Context Management**
- Compression at 50% context threshold (configurable)
- Emergency trimming at 70%/90%
- Session lineage via `parent_session_id` chains
- Loop detection for file re-read after compression
- 413 Payload Too Large → automatic compression trigger
- Anthropic prompt caching (cache_control headers)

### Messaging Gateway (7 Platforms)

- **Telegram** — long polling, forum topics, documents (PDF/text/Office), location, voice transcription
- **Discord** — bot intents, channel topics, documents/video, bot filtering, auto-threading
- **Slack** — Socket Mode, threading, documents/video
- **WhatsApp** — Baileys Node.js bridge, native media, multi-user isolation
- **Signal** — signal-cli HTTP daemon, E164 phone numbers, stories filtering
- **Email** — IMAP polling + SMTP sending, MIME multipart, HTML support (new in this release)
- **Home Assistant** — REST tools + WebSocket events, domain/entity filtering

**Gateway infrastructure:**
- `SessionSource` extraction for all platforms
- Session reset policy (daily/idle/both/none) with per-platform and per-type overrides
- `/new` and `/reset` triggers
- `group_sessions_per_user` mode
- Channel directory cache (refreshed every 5 minutes)
- Delivery target routing (origin / local / platform / specific channel)
- Voice message transcription via faster-whisper (configurable STT backends)
- Quick commands (`/standup`, `/brief`, etc.)

### Skills Ecosystem (70+ Bundled Skills)

**New categories added:**
- Apple/macOS (4 skills: Notes, Reminders, FindMy, iMessage)
- Autonomous AI Agents (4 skills: Claude Code, Codex, Hermes spawning, OpenCode)
- Creative (3 skills: ASCII Art, ASCII Video, Excalidraw)
- MLOps training (14 skills: Axolotl, TRL, GRPO, Unsloth, PEFT, etc.)
- MLOps inference (8 skills: vLLM, TensorRT-LLM, llama.cpp, etc.)
- Vision/multimodal (6 skills: Whisper, CLIP, SAM, Stable Diffusion, etc.)
- Vector databases (4 skills: Chroma, FAISS, Pinecone, Qdrant)
- Research (6 skills: ArXiv, domain-intel, blogwatcher, etc.)
- GitHub (6 skills: PR workflow, code review, issues, etc.)
- Productivity (5 skills: Google Workspace, Notion, PDF, PowerPoint)
- Gaming (2 skills: Minecraft, Pokemon player)
- Media (4 skills: GIF search, music generation, YouTube)

**New optional skills (9):**
- `solana` — Blockchain queries
- `agentmail` — Agent-owned email
- `blackbox` — Blackbox AI CLI
- `neuroskill-bci` — BCI wearable integration
- `openclaw-migration` — Migrate from OpenClaw
- `qmd` — Personal knowledge base search
- `1password` — 1Password CLI
- `oss-forensics` — Security forensics

**Skills Hub:**
- Multi-source browsing (official, skills-sh, community, github, well-known)
- Security scanning with trust levels
- Enable/disable without removing
- Platform-specific skills (macOS-only, Linux-only, etc.)
- Conditional activation (`fallback_for_toolsets`, `requires_toolsets`)

### MCP (Model Context Protocol) Client

- Native stdio and HTTP (SSE) transport
- Automatic reconnection on disconnect
- Tool prefix: `mcp_<server>_<tool_name>`
- Per-session MCP server injection (via ACP)
- `native-mcp` skill for guided setup
- `mcporter` skill for CLI bridge

### ACP Server (Editor Integration)

- Full ACP protocol: initialize, authenticate, new/load/resume/fork/list sessions, prompt, cancel
- Streaming: thinking, tool_start, tool_end, step, message events
- Multi-session: ThreadPoolExecutor(4)
- VS Code, Zed, JetBrains compatible
- `hermes-acp` toolset (excludes audio/TTS/clarify)

### CLI Improvements

**New slash commands:**
- `/reasoning [level]` — Set reasoning effort
- `/rollback` — Restore filesystem snapshot
- `/title <name>` — Name/rename current session
- `/platforms` — Show connected messaging platforms

**New features:**
- Skin/theme engine: 7 built-in skins + custom YAML skins
- Git worktree isolation: `hermes -w`
- Filesystem checkpoints + rollback
- `hermes sessions` subcommand with full history management
- Shell completions (bash, zsh, fish)
- `hermes claw migrate` command

### Honcho Integration

- Hybrid/context/tools memory modes
- Session strategy: per-session/per-repo/per-directory/global
- Dialectic reasoning levels (minimal → max)
- Write frequency: async/turn/session/N
- `hermes honcho` management commands
- Linked hosts for cross-device memory

### Security Hardening (20+ fixes)

- Path traversal fix for all file tools
- Shell injection prevention via direct exec
- Dangerous command detection and warning
- Symlink boundary checks
- Prompt injection scanning before loading context files
- FTS5 query sanitization
- `.env` and auth file permissions enforced (0600/0700)
- Atomic writes for all critical files (temp + fsync + os.replace)
- Secret redaction patterns + in-memory allowlist
- File locking for concurrent auth store access

### Testing

- **3,289 tests** total
- Parallelized with pytest-xdist
- Test coverage: agent loop, tool calling, gateway (all 7 platforms), cron, CLI, security, sessions, ACP, skills, honcho, trajectory format

### Performance

- Smart context length probing with persistent caching
- Tool call repair middleware
- Lazy tool loading (import only enabled toolsets)
- FTS5 indexing for session search
- SQLite WAL mode for concurrent gateway access

### Documentation

- Full docs site: 74 Markdown files
- Developer guide: architecture, agent loop, adding tools, adding providers, ACP internals, trajectory format
- Feature guides: 23 feature-specific pages
- Platform guides: 8 platform setup pages
- Reference: complete CLI commands, env vars, tools, toolsets, skills

---

## [0.1.0] — 2026-02-24

Initial internal release.

### Core

- `AIAgent` class with ReAct loop
- OpenAI-compatible provider (OpenRouter)
- Anthropic native provider
- SQLite session storage (WAL, FTS5)
- Terminal backend (local)
- Basic CLI (`hermes` command)
- Telegram gateway
- Discord gateway
- Slack gateway
- First bundled skills (GitHub, web research, systematic-debugging)
- `session_search` tool
- `memory` tool
- Context compression (basic)

---

*Earlier pre-release versions (internal prototypes, ~Feb 10–23, 2026) are not documented here.*

---

## Top Contributors (v0.2.0)

| Contributor | PRs | Areas |
|-------------|-----|-------|
| `@teknium1` | 43 | Core architecture, provider router, sessions, skills, CLI, docs |
| `@0xbyt4` | 40 | MCP client, Home Assistant, security (6 batches), ascii-art, testing |
| `@Farukest` | 16 | Security hardening, Windows compatibility, WhatsApp |
| `@aydnOktay` | 11 | Atomic writes, error handling across Telegram/Discord/TTS/skills |
| `@Bartok9` | 9 | Contributing guide, slash commands, Discord channel topics |
| `@PercyDikec` | 7 | DeepSeek parser fix, /retry, gateway transcript |
| `@teyrebaz33` | 5 | Skills enable/disable, quick commands, personality |
| `@alireza78a` | 5 | Atomic writes (cron, sessions), fd leak prevention |
| `@shitcoinsherpa` | 3 | Windows support (pywinpty, UTF-8, auth store lock) |

63 total contributors.
