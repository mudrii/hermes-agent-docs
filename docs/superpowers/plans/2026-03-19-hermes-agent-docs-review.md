# Hermes Agent Documentation Review & Update Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Comprehensively review all hermes-agent source code and documentation, then produce accurate, fact-checked, complete documentation in ~/src/claw/hermes-agent-docs covering released versions v0.2.0 and v0.3.0 only.

**Architecture:** Six parallel documentation agents cover distinct subsystems (core agent, gateway/messaging, CLI/providers, features/plugins, skills/tools, reference/security). Each reads authoritative source from `~/src/claw/hermes-agent/` (official website docs + source code) and writes to `~/src/claw/hermes-agent-docs/`. All facts are validated against source code — not invented.

**Tech Stack:** Python, asyncio, UV/pip, Telegram/Discord/Slack/WhatsApp/Signal/Email/Matrix/Mattermost/DingTalk/Home Assistant, MCP (Model Context Protocol), ACP (Agent Communication Protocol), Anthropic/OpenAI/OpenRouter APIs, Honcho memory

---

## Source of Truth

- **Released versions:** v0.2.0 (v2026.3.12) and v0.3.0 (v2026.3.17) ONLY
- **Official website docs:** `~/src/claw/hermes-agent/website/docs/`
- **Release notes:** `~/src/claw/hermes-agent/RELEASE_v0.2.0.md`, `~/src/claw/hermes-agent/RELEASE_v0.3.0.md`
- **Source code:** `~/src/claw/hermes-agent/` (all Python modules)
- **Output directory:** `~/src/claw/hermes-agent-docs/`

## File Map

### Files to UPDATE (existing, need review and expansion):
- `README.md` — Project overview, feature summary, quick links
- `architecture.md` — System architecture, component diagram, data flow
- `installation.md` — Install methods, prerequisites, setup commands
- `cli-reference.md` — All CLI commands, flags, options
- `configuration.md` — All config.yaml options, env vars
- `gateway.md` — Gateway architecture and all platforms
- `providers.md` — Provider system, routing, credentials
- `skills.md` — Skills system, categories, activation
- `tools.md` — Tool reference, all built-in tools
- `toolsets.md` — Toolset definitions and configuration
- `acp.md` — ACP server/client, editor integration
- `memory.md` — Honcho memory, session storage
- `security.md` — Security model, PII redaction, approvals
- `contributing.md` — Contribution guide, dev setup
- `changelog.md` — Version history (v0.1.0, v0.2.0, v0.3.0)

### Files to CREATE (missing from existing docs):
- `plugins.md` — Plugin architecture (v0.3.0 feature)
- `streaming.md` — Unified streaming infrastructure (v0.3.0)
- `voice-mode.md` — Voice mode, TTS, STT (v0.3.0)
- `browser.md` — Browser tool, CDP connect (v0.3.0)
- `checkpoints.md` — Filesystem checkpoints & rollback
- `cron.md` — Cron job scheduling
- `mcp.md` — MCP client integration
- `hooks.md` — Hook system for automation
- `skins.md` — CLI theme/skin engine
- `batch-processing.md` — Batch runner for bulk tasks
- `delegation.md` — Task delegation between agents
- `api-server.md` — API server mode
- `messaging/` — Per-platform messaging guides (Telegram, Discord, Slack, WhatsApp, Signal, Email, Matrix, Mattermost, DingTalk, Home Assistant, Open WebUI, SMS)

---

## Agent Assignments

This plan is designed for **6 parallel agents**, each owning a documentation domain:

| Agent | Domain | Files |
|-------|---------|-------|
| Agent A | Core & Architecture | architecture.md, README.md, changelog.md, streaming.md |
| Agent B | Gateway & Messaging | gateway.md, messaging/ (12 platform files) |
| Agent C | CLI, Providers & Config | cli-reference.md, providers.md, configuration.md |
| Agent D | Features & Plugins | plugins.md, hooks.md, browser.md, checkpoints.md, cron.md, voice-mode.md, batch-processing.md, delegation.md, api-server.md, skins.md |
| Agent E | Skills & Tools | skills.md, tools.md, toolsets.md, mcp.md |
| Agent F | Installation, Security & Contributing | installation.md, security.md, contributing.md, acp.md, memory.md |

---

## Task A: Core Architecture & Overview Agent

**Files:**
- Modify: `~/src/claw/hermes-agent-docs/README.md`
- Modify: `~/src/claw/hermes-agent-docs/architecture.md`
- Modify: `~/src/claw/hermes-agent-docs/changelog.md`
- Create: `~/src/claw/hermes-agent-docs/streaming.md`

**Sources to read:**
- `~/src/claw/hermes-agent/website/docs/developer-guide/architecture.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/agent-loop.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/prompt-assembly.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/context-compression-and-caching.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/session-storage.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/trajectory-format.md`
- `~/src/claw/hermes-agent/RELEASE_v0.2.0.md`
- `~/src/claw/hermes-agent/RELEASE_v0.3.0.md`
- `~/src/claw/hermes-agent/README.md`
- `~/src/claw/hermes-agent/run_agent.py`
- `~/src/claw/hermes-agent/agent/` (all files)
- `~/src/claw/hermes-agent/hermes_state.py`
- `~/src/claw/hermes-agent/hermes_time.py`
- `~/src/claw/hermes-agent/hermes_constants.py`

- [ ] **Step A1: Read all source files for core architecture domain**
- [ ] **Step A2: Verify facts against source code (agent loop, context compression, streaming)**
- [ ] **Step A3: Update README.md — accurate project overview, v0.2.0/v0.3.0 highlights, feature list**
- [ ] **Step A4: Rewrite architecture.md — component map, data flow, agent loop, context compression**
- [ ] **Step A5: Create streaming.md — unified streaming infrastructure, token delivery, platform support**
- [ ] **Step A6: Update changelog.md — accurate v0.1.0, v0.2.0, v0.3.0 entries from release notes**
- [ ] **Step A7: Verify all claims against source code before finalizing**

---

## Task B: Gateway & Messaging Agent

**Files:**
- Modify: `~/src/claw/hermes-agent-docs/gateway.md`
- Create: `~/src/claw/hermes-agent-docs/messaging/` directory with 12 platform files

**Sources to read:**
- `~/src/claw/hermes-agent/website/docs/developer-guide/gateway-internals.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/messaging/` (all platform files)
- `~/src/claw/hermes-agent/gateway/` (all Python files)
- `~/src/claw/hermes-agent/gateway/platforms/` (all platform implementations)

**Platform files to create:**
- `messaging/telegram.md`
- `messaging/discord.md`
- `messaging/slack.md`
- `messaging/whatsapp.md`
- `messaging/signal.md`
- `messaging/email.md`
- `messaging/matrix.md`
- `messaging/mattermost.md`
- `messaging/dingtalk.md`
- `messaging/homeassistant.md`
- `messaging/open-webui.md`
- `messaging/sms.md`
- `messaging/README.md` (index)

- [ ] **Step B1: Read gateway source code and all platform implementations**
- [ ] **Step B2: Read all official website messaging docs**
- [ ] **Step B3: Rewrite gateway.md — architecture, session management, approval system, /stop command**
- [ ] **Step B4: Create each platform file with: setup, credentials, config, features, known limitations**
- [ ] **Step B5: Verify credentials and config keys match source code**
- [ ] **Step B6: Create messaging/README.md index**

---

## Task C: CLI, Providers & Configuration Agent

**Files:**
- Modify: `~/src/claw/hermes-agent-docs/cli-reference.md`
- Modify: `~/src/claw/hermes-agent-docs/providers.md`
- Modify: `~/src/claw/hermes-agent-docs/configuration.md`

**Sources to read:**
- `~/src/claw/hermes-agent/website/docs/reference/cli-commands.md`
- `~/src/claw/hermes-agent/website/docs/reference/environment-variables.md`
- `~/src/claw/hermes-agent/website/docs/reference/slash-commands.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/configuration.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/provider-runtime.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/adding-providers.md`
- `~/src/claw/hermes-agent/cli.py`
- `~/src/claw/hermes-agent/hermes_cli/` (all files)
- `~/src/claw/hermes-agent/gateway/config.py`

- [ ] **Step C1: Read CLI source (cli.py, hermes_cli/) and extract all commands/flags**
- [ ] **Step C2: Read provider runtime source — routing logic, credential resolution, fallback**
- [ ] **Step C3: Read configuration source and all env var definitions**
- [ ] **Step C4: Rewrite cli-reference.md — all commands, slash commands, flags, examples**
- [ ] **Step C5: Rewrite providers.md — all providers (OpenAI, Anthropic, OpenRouter, Vercel, custom), credentials, routing**
- [ ] **Step C6: Rewrite configuration.md — full config.yaml reference with all options documented**
- [ ] **Step C7: Cross-validate all flag names, env var names against actual source code**

---

## Task D: Features & Plugins Agent

**Files:**
- Create: `~/src/claw/hermes-agent-docs/plugins.md`
- Create: `~/src/claw/hermes-agent-docs/streaming.md` (coordinate with Agent A)
- Create: `~/src/claw/hermes-agent-docs/hooks.md`
- Create: `~/src/claw/hermes-agent-docs/browser.md`
- Create: `~/src/claw/hermes-agent-docs/checkpoints.md`
- Create: `~/src/claw/hermes-agent-docs/cron.md`
- Create: `~/src/claw/hermes-agent-docs/voice-mode.md`
- Create: `~/src/claw/hermes-agent-docs/batch-processing.md`
- Create: `~/src/claw/hermes-agent-docs/delegation.md`
- Create: `~/src/claw/hermes-agent-docs/api-server.md`
- Create: `~/src/claw/hermes-agent-docs/skins.md`

**Sources to read:**
- `~/src/claw/hermes-agent/website/docs/user-guide/features/plugins.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/hooks.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/browser.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/checkpoints.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/cron.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/voice-mode.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/tts.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/batch-processing.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/delegation.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/api-server.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/skins.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/personality.md`
- `~/src/claw/hermes-agent/website/docs/guides/build-a-hermes-plugin.md`
- `~/src/claw/hermes-agent/cron/` (all files)
- `~/src/claw/hermes-agent/batch_runner.py`
- `~/src/claw/hermes-agent/tools/delegate_tool.py`
- `~/src/claw/hermes-agent/tools/tts_tool.py`
- `~/src/claw/hermes-agent/tools/neutts_synth.py`

- [ ] **Step D1: Read all feature source files and official website docs**
- [ ] **Step D2: Create plugins.md — plugin architecture, discovery, hooks, custom tools/commands**
- [ ] **Step D3: Create hooks.md — hook types, registration, examples**
- [ ] **Step D4: Create browser.md — browser tool, CDP connect, web automation**
- [ ] **Step D5: Create checkpoints.md — filesystem checkpoints, rollback, git worktrees**
- [ ] **Step D6: Create cron.md — cron scheduling, job types, configuration**
- [ ] **Step D7: Create voice-mode.md — voice mode, TTS providers, STT, configuration**
- [ ] **Step D8: Create batch-processing.md — batch runner, datagen, configuration**
- [ ] **Step D9: Create delegation.md — task delegation, agent hierarchy**
- [ ] **Step D10: Create api-server.md — API server mode, endpoints, authentication**
- [ ] **Step D11: Create skins.md — CLI themes, custom skins, banner configuration**

---

## Task E: Skills, Tools & MCP Agent

**Files:**
- Modify: `~/src/claw/hermes-agent-docs/skills.md`
- Modify: `~/src/claw/hermes-agent-docs/tools.md`
- Modify: `~/src/claw/hermes-agent-docs/toolsets.md`
- Create: `~/src/claw/hermes-agent-docs/mcp.md`

**Sources to read:**
- `~/src/claw/hermes-agent/website/docs/user-guide/features/skills.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/tools.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/mcp.md`
- `~/src/claw/hermes-agent/website/docs/reference/skills-catalog.md`
- `~/src/claw/hermes-agent/website/docs/reference/optional-skills-catalog.md`
- `~/src/claw/hermes-agent/website/docs/reference/tools-reference.md`
- `~/src/claw/hermes-agent/website/docs/reference/toolsets-reference.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/creating-skills.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/adding-tools.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/tools-runtime.md`
- `~/src/claw/hermes-agent/website/docs/reference/mcp-config-reference.md`
- `~/src/claw/hermes-agent/skills/` (directory structure)
- `~/src/claw/hermes-agent/tools/` (all tool files)
- `~/src/claw/hermes-agent/toolsets.py`
- `~/src/claw/hermes-agent/toolset_distributions.py`
- `~/src/claw/hermes-agent/tools/mcp_tool.py`
- `~/src/claw/hermes-agent/tools/skills_tool.py`

- [ ] **Step E1: Read all skills, tools, MCP source and website docs**
- [ ] **Step E2: Rewrite skills.md — complete skills catalog (all 70+ skills), categories, activation, SKILL.md format**
- [ ] **Step E3: Rewrite tools.md — all built-in tools with parameters, examples, security notes**
- [ ] **Step E4: Rewrite toolsets.md — toolset system, distributions, per-platform config**
- [ ] **Step E5: Create mcp.md — MCP client setup, transports, server config, sampling, resource discovery**
- [ ] **Step E6: Verify skills count and categories against actual skills/ directory**

---

## Task F: Installation, Security, ACP & Memory Agent

**Files:**
- Modify: `~/src/claw/hermes-agent-docs/installation.md`
- Modify: `~/src/claw/hermes-agent-docs/security.md`
- Modify: `~/src/claw/hermes-agent-docs/contributing.md`
- Modify: `~/src/claw/hermes-agent-docs/acp.md`
- Modify: `~/src/claw/hermes-agent-docs/memory.md`

**Sources to read:**
- `~/src/claw/hermes-agent/website/docs/getting-started/installation.md`
- `~/src/claw/hermes-agent/website/docs/getting-started/quickstart.md`
- `~/src/claw/hermes-agent/website/docs/getting-started/updating.md`
- `~/src/claw/hermes-agent/website/docs/getting-started/learning-path.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/security.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/acp.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/honcho.md`
- `~/src/claw/hermes-agent/website/docs/user-guide/features/memory.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/acp-internals.md`
- `~/src/claw/hermes-agent/website/docs/developer-guide/contributing.md`
- `~/src/claw/hermes-agent/website/docs/reference/faq.md`
- `~/src/claw/hermes-agent/CONTRIBUTING.md`
- `~/src/claw/hermes-agent/setup-hermes.sh`
- `~/src/claw/hermes-agent/pyproject.toml`
- `~/src/claw/hermes-agent/hermes_cli/auth.py`
- `~/src/claw/hermes-agent/hermes_cli/copilot_auth.py`
- `~/src/claw/hermes-agent/hermes_cli/setup.py`
- `~/src/claw/hermes-agent/acp_registry/`
- `~/src/claw/hermes-agent/honcho_integration/`

- [ ] **Step F1: Read all installation, security, ACP, memory source and website docs**
- [ ] **Step F2: Rewrite installation.md — all install methods (pip, uv, git clone), setup wizard, prerequisites**
- [ ] **Step F3: Rewrite security.md — security model, PII redaction, approval system, OAuth, secret handling**
- [ ] **Step F4: Rewrite acp.md — ACP server setup, VS Code/Zed/JetBrains integration, copilot auth**
- [ ] **Step F5: Rewrite memory.md — Honcho integration, recall modes, session titles, multi-user isolation**
- [ ] **Step F6: Rewrite contributing.md — dev setup, test suite, PR process, code conventions**

---

## Final Integration Tasks

### Task G: Update README.md Index

- [ ] **Step G1: Update hermes-agent-docs/README.md** to link all new documents
- [ ] **Step G2: Verify all internal links are correct**
- [ ] **Step G3: Add version badges for v0.2.0 and v0.3.0**

### Task H: Git Commit & Push

- [ ] **Step H1: Stage all changes in hermes-agent-docs**

```bash
cd ~/src/claw/hermes-agent-docs
git add -A
```

- [ ] **Step H2: Create commit**

```bash
git commit -m "docs: comprehensive update for v0.2.0 and v0.3.0 releases"
```

- [ ] **Step H3: Push to GitHub**

```bash
git push origin main
```

---

## Quality Rules for All Agents

1. **Facts only** — every claim must be verifiable in source code or official website docs
2. **Released versions only** — document v0.2.0 and v0.3.0 features only, not unreleased main
3. **No hallucination** — if a feature isn't in source code, don't document it
4. **Exact names** — use exact flag names, config keys, command names from source
5. **Cross-validate** — grep source code to confirm documented behavior
6. **No generic AI filler** — every sentence must convey specific, accurate information
