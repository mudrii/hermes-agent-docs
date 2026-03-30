# Hermes Agent v0.4.0 + v0.5.0 Documentation Update Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Review the full hermes-agent source code and update hermes-agent-docs to fully, accurately cover v0.4.0 (v2026.3.23) and v0.5.0 (v2026.3.28), fact-checking every claim against the source and release notes.

**Architecture:** Six parallel documentation agents each own a topic cluster. Each agent reads the relevant source files in `~/src/claw/hermes-agent`, validates existing doc content against code, updates stale sections, and adds new sections for v0.4.0/v0.5.0 features. All work lands in `~/src/claw/hermes-agent-docs`. One final agent handles commit and push.

**Tech Stack:** Python source (hermes-agent), Markdown docs (hermes-agent-docs), GitHub (NousResearch/hermes-agent source, mudrii/hermes-agent-docs output), online docs at hermes-agent.nousresearch.com/docs.

**Release scope:**
- v0.3.0 = v2026.3.17 — already documented (baseline)
- v0.4.0 = v2026.3.23 — needs full documentation
- v0.5.0 = v2026.3.28 — needs full documentation

**Source repo:** `~/src/claw/hermes-agent` (already at latest main)
**Docs repo:** `~/src/claw/hermes-agent-docs` (current state: v0.3.0)

---

## File Map

### Files to modify (updating existing)
| File | Responsible task |
|------|-----------------|
| `README.md` | Task 1 |
| `changelog.md` | Task 1 |
| `installation.md` | Task 1 |
| `configuration.md` | Task 2 |
| `cli-reference.md` | Task 2 |
| `streaming.md` | Task 2 |
| `checkpoints.md` | Task 2 |
| `delegation.md` | Task 2 |
| `memory.md` | Task 2 |
| `gateway.md` | Task 3 |
| `providers.md` | Task 3 |
| `messaging/telegram.md` | Task 3 |
| `messaging/signal.md` | Task 3 |
| `messaging/discord.md` | Task 3 |
| `messaging/slack.md` | Task 3 |
| `messaging/whatsapp.md` | Task 3 |
| `messaging/email.md` | Task 3 |
| `messaging/mattermost.md` | Task 3 |
| `messaging/matrix.md` | Task 3 |
| `messaging/dingtalk.md` | Task 3 |
| `messaging/sms.md` | Task 3 |
| `tools.md` | Task 4 |
| `toolsets.md` | Task 4 |
| `skills.md` | Task 4 |
| `plugins.md` | Task 4 |
| `hooks.md` | Task 4 |
| `api-server.md` | Task 5 |
| `mcp.md` | Task 5 |
| `acp.md` | Task 5 |
| `cron.md` | Task 5 |
| `batch-processing.md` | Task 5 |
| `security.md` | Task 6 |
| `contributing.md` | Task 6 |
| `voice-mode.md` | Task 6 |
| `browser.md` | Task 6 |
| `architecture.md` | Task 6 |

### Files to create (new in v0.4.0/v0.5.0)
| File | Responsible task |
|------|-----------------|
| `messaging/webhook.md` | Task 3 |
| `messaging/homeassistant.md` (update) | Task 3 |

---

## Task 1: Core Docs — README, Changelog, Installation

**Owner:** Agent 1
**Source files to read:**
- `~/src/claw/hermes-agent/README.md`
- `~/src/claw/hermes-agent/RELEASE_v0.4.0.md`
- `~/src/claw/hermes-agent/RELEASE_v0.5.0.md`
- `~/src/claw/hermes-agent/pyproject.toml`
- `~/src/claw/hermes-agent/scripts/install.sh`
- `~/src/claw/hermes-agent/setup-hermes.sh`
- `~/src/claw/hermes-agent/requirements.txt`
- `~/src/claw/hermes-agent/flake.nix` (for Nix install section)
- `~/src/claw/hermes-agent/cli.py` (for hermes subcommands list)

**Online validation:** Fetch https://hermes-agent.nousresearch.com/docs/ and compare quick-start steps to source.

**Files to update:** `README.md`, `changelog.md`, `installation.md`

- [ ] **Step 1: Read current README.md in docs**

```bash
cat ~/src/claw/hermes-agent-docs/README.md
```

- [ ] **Step 2: Read hermes-agent source README and release notes**

```bash
cat ~/src/claw/hermes-agent/README.md
cat ~/src/claw/hermes-agent/RELEASE_v0.4.0.md
cat ~/src/claw/hermes-agent/RELEASE_v0.5.0.md
```

- [ ] **Step 3: Read pyproject.toml for version and dependencies**

```bash
cat ~/src/claw/hermes-agent/pyproject.toml
```

- [ ] **Step 4: Read install scripts**

```bash
cat ~/src/claw/hermes-agent/scripts/install.sh
cat ~/src/claw/hermes-agent/setup-hermes.sh
```

- [ ] **Step 5: Read flake.nix for Nix installation**

```bash
cat ~/src/claw/hermes-agent/flake.nix | head -150
```

- [ ] **Step 6: Read cli.py for hermes subcommand list**

```bash
cat ~/src/claw/hermes-agent/cli.py | head -200
```

- [ ] **Step 7: Online validation — fetch official docs quickstart**

Use WebFetch to get: https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
Compare install steps to what scripts/install.sh actually does.

- [ ] **Step 8: Update README.md**

Update these sections:
- "Current version" → `v0.5.0 (v2026.3.28)` released March 28, 2026
- "Key Features" table — add Nix flake, HuggingFace provider, API server, new messaging platforms from v0.4.0/v0.5.0
- "Getting Started" commands — add `hermes mcp`, `hermes api-server` if present in cli.py
- Quick Reference table — add new CLI commands from v0.4.0 (`/statusbar`, `/queue`, `/permission`, `/browser`, `/cost`, `/approve`, `/deny`, `/resume`)
- Documentation links table — ensure all links are present and accurate

- [ ] **Step 9: Update changelog.md**

Add complete v0.4.0 and v0.5.0 sections at the top of the file, following the existing format established by the v0.3.0 entry. Each section must:
- Open with version header, date, and tagline
- Cover every subsystem from the release notes with accurate PR links
- Include implementation details (function names, config keys, behavior changes) sourced from actual source code — not just release notes prose

v0.4.0 sections to cover:
- CLI & UX (new commands, streaming default, @ context)
- Gateway & Messaging (new adapters, prompt caching, auto-reconnect)
- New Providers (GitHub Copilot, Alibaba, Kilo, OpenCode)
- Context Compression (structured summaries, token budget)
- MCP (hermes mcp CLI, OAuth 2.1 PKCE)
- API Server (OpenAI-compatible endpoint, /api/jobs)
- Delegation (@ context injection, subagent improvements)
- Memory & Sessions
- Security & Reliability fixes

v0.5.0 sections to cover:
- New Provider: HuggingFace
- /model command overhaul
- Telegram Private Chat Topics
- Native Modal SDK
- Plugin lifecycle hooks
- Tool-use enforcement
- Nix flake
- Supply chain hardening
- Anthropic output limits fix
- Context compression ratio tuning
- Session search improvements
- 50+ bug fixes (grouped by subsystem)

- [ ] **Step 10: Update installation.md**

Update:
- Version references throughout to v0.5.0
- Add Nix installation section (based on flake.nix and README)
  - `nix run github:NousResearch/hermes-agent` syntax
  - NixOS module usage
  - Persistent container mode
- Verify Python requirements match pyproject.toml
- Add supply chain note (litellm removed, uv.lock with hashes)

- [ ] **Step 11: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add README.md changelog.md installation.md
git commit -m "docs: update README, changelog, installation for v0.4.0 and v0.5.0"
```

---

## Task 2: CLI, Configuration, Streaming, Session, Delegation

**Owner:** Agent 2
**Source files to read:**
- `~/src/claw/hermes-agent/cli.py`
- `~/src/claw/hermes-agent/hermes_cli/` (all files)
- `~/src/claw/hermes-agent/hermes_state.py`
- `~/src/claw/hermes-agent/hermes_constants.py`
- `~/src/claw/hermes-agent/run_agent.py` (agent loop, streaming, compression)
- `~/src/claw/hermes-agent/cli-config.yaml.example`
- `~/src/claw/hermes-agent/trajectory_compressor.py`
- `~/src/claw/hermes-agent/agent/` (if exists — delegation)

**Online validation:** Fetch https://hermes-agent.nousresearch.com/docs/user-guide/cli

**Files to update:** `cli-reference.md`, `configuration.md`, `streaming.md`, `checkpoints.md`, `delegation.md`, `memory.md`

- [ ] **Step 1: Read current docs**

```bash
cat ~/src/claw/hermes-agent-docs/cli-reference.md
cat ~/src/claw/hermes-agent-docs/configuration.md
cat ~/src/claw/hermes-agent-docs/streaming.md
cat ~/src/claw/hermes-agent-docs/checkpoints.md
cat ~/src/claw/hermes-agent-docs/delegation.md
cat ~/src/claw/hermes-agent-docs/memory.md
```

- [ ] **Step 2: Read CLI source**

```bash
cat ~/src/claw/hermes-agent/cli.py
ls ~/src/claw/hermes-agent/hermes_cli/
cat ~/src/claw/hermes-agent/hermes_cli/*.py 2>/dev/null || ls ~/src/claw/hermes-agent/hermes_cli/
```

- [ ] **Step 3: Read agent core for streaming and compression**

```bash
cat ~/src/claw/hermes-agent/run_agent.py | head -500
cat ~/src/claw/hermes-agent/hermes_state.py
cat ~/src/claw/hermes-agent/hermes_constants.py
```

- [ ] **Step 4: Read config example**

```bash
cat ~/src/claw/hermes-agent/cli-config.yaml.example
```

- [ ] **Step 5: Online validation**

Use WebFetch: https://hermes-agent.nousresearch.com/docs/user-guide/cli
Compare slash commands listed online to what cli.py actually handles.

- [ ] **Step 6: Update cli-reference.md**

New commands to add (introduced in v0.4.0):
- `/statusbar` — toggle persistent config bar (PR #2240, #1917)
- `/queue [message]` — queue prompt without interrupting current run (PR #2191, #2469)
- `/permission [mode]` — switch approval mode dynamically (PR #2207)
- `/browser` — interactive browser session (PR #2273, #1814)
- `/cost` — live pricing/usage in gateway mode (PR #2180)
- `/approve` and `/deny` — gateway approval commands replacing bare text (PR #2002)

New commands to add (introduced in v0.5.0):
- `/resume` — resume a session (PR #3315)

`hermes` subcommands to verify and update:
- `hermes mcp` — MCP server management (added v0.4.0)
- `hermes model` — extracted from slash command (PR #3080 in v0.5.0 removes `/model` in favor of `hermes model`)
- `hermes api-server` (if present)
- `hermes claw migrate` — OpenClaw migration

For each command, source the exact invocation syntax and options directly from cli.py — do not paraphrase.

@ context references (v0.4.0):
- Document `@file` syntax with tab completion
- Document `@url` syntax
- Show example usage

- [ ] **Step 7: Update configuration.md**

New config keys to document (v0.4.0/v0.5.0 — source from cli-config.yaml.example and hermes_constants.py):
- `streaming.*` — `enabled` now defaults to `true` (was `false` in v0.3.0)
- `compression.target_ratio`, `compression.protect_last_n`, `compression.threshold`
- `agent.tool_use_enforcement` (v0.5.0 — PR #3551)
- `api_server.*` (v0.4.0)
- Provider-level `provider` and `base_url` root-level keys (PR #3112)
- Session config options surfaced on `/new`/`/reset` (PR #3321)
- Third-party session isolation `--source` flag (PR #3255)

For each key: name, type, default value (from source), description, example.

- [ ] **Step 8: Update streaming.md**

Key changes to reflect:
- Streaming now **on by default** as of v0.4.0 (PR #2340) — update "Defaults" section
- Show spinner and tool progress during streaming (PR #2161)
- Reasoning blocks display when `show_reasoning` enabled (PR #2118)
- Context pressure warnings during streaming (PR #2159)
- Persist reasoning across gateway turns with schema v6 columns (PR #2974, v0.5.0)
- `HERMES_STREAMING_ENABLED` env var still works but default changed
- Non-streaming fallback preserved after stream failures (PR #3020)
- Always-prefer-streaming to prevent hung subagents (PR #3120)

- [ ] **Step 9: Update delegation.md**

Key changes:
- Subagents now have independent iteration budgets (PR #3004, v0.5.0)
- Update `api_key` in fallback for subagent auth (PR #3103)
- `--source` flag for session isolation (PR #3255)
- @ context references as delegation input (v0.4.0)

- [ ] **Step 10: Update memory.md**

Key changes:
- Session search "recent sessions mode" — omit query to browse recent sessions with titles, previews, timestamps (PR #2533)
- Session config surfacing on `/new`, `/reset`, auto-reset (PR #3321)
- `/resume` command (PR #3315)
- Session log truncation guard
- `reopen_session` API
- Session search fallback preview on summarization failure (PR #3478)

- [ ] **Step 11: Update checkpoints.md**

Verify existing content is still accurate by reading trajectory_compressor.py.
Add any changes from v0.4.0/v0.5.0 compression overhaul (structured summaries, target_ratio, protect_last_n).

- [ ] **Step 12: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add cli-reference.md configuration.md streaming.md checkpoints.md delegation.md memory.md
git commit -m "docs: update CLI, config, streaming, delegation, memory for v0.4.0 and v0.5.0"
```

---

## Task 3: Gateway, Messaging Platforms, Providers

**Owner:** Agent 3
**Source files to read:**
- `~/src/claw/hermes-agent/gateway/run.py`
- `~/src/claw/hermes-agent/gateway/platforms/` (all platform files)
- `~/src/claw/hermes-agent/gateway/config.py`
- `~/src/claw/hermes-agent/gateway/session.py`
- `~/src/claw/hermes-agent/model_tools.py` (provider list)
- `~/src/claw/hermes-agent/toolset_distributions.py` (provider mappings)
- `~/src/claw/hermes-agent/hermes_constants.py` (SUPPORTED_PROVIDERS or similar)

**Online validation:**
- Fetch https://hermes-agent.nousresearch.com/docs/user-guide/messaging
- Fetch https://hermes-agent.nousresearch.com/docs/user-guide/providers (if exists)

**Files to update:** `gateway.md`, `providers.md`, `messaging/telegram.md`, `messaging/signal.md`, `messaging/discord.md`, `messaging/slack.md`, `messaging/whatsapp.md`, `messaging/email.md`, `messaging/mattermost.md`, `messaging/matrix.md`, `messaging/dingtalk.md`, `messaging/sms.md`, `messaging/homeassistant.md`
**Files to create:** `messaging/webhook.md`

- [ ] **Step 1: Read current gateway and messaging docs**

```bash
cat ~/src/claw/hermes-agent-docs/gateway.md
cat ~/src/claw/hermes-agent-docs/providers.md
cat ~/src/claw/hermes-agent-docs/messaging/README.md
ls ~/src/claw/hermes-agent-docs/messaging/
```

- [ ] **Step 2: Read gateway source**

```bash
ls ~/src/claw/hermes-agent/gateway/
cat ~/src/claw/hermes-agent/gateway/run.py | head -300
ls ~/src/claw/hermes-agent/gateway/platforms/
cat ~/src/claw/hermes-agent/gateway/config.py
```

- [ ] **Step 3: Read each platform adapter**

```bash
for f in ~/src/claw/hermes-agent/gateway/platforms/*.py; do echo "=== $f ==="; head -80 $f; done
```

- [ ] **Step 4: Read model_tools.py for provider list**

```bash
cat ~/src/claw/hermes-agent/model_tools.py | head -400
```

- [ ] **Step 5: Online validation**

Use WebFetch:
- https://hermes-agent.nousresearch.com/docs/user-guide/messaging
- https://hermes-agent.nousresearch.com/docs/user-guide/providers (try this URL)

- [ ] **Step 6: Update gateway.md**

Key changes from v0.4.0:
- Auto-reconnect with exponential backoff for failed platforms (PR #2584)
- Gateway prompt caching — AIAgent instances cached per session, preserving Anthropic prompt cache across turns (PR #2282, #2284, #2361). Explain the caching mechanism.
- `/approve` and `/deny` commands replace bare text approvals (PR #2002)
- `/cost` command for live pricing (PR #2180)
- Exponential backoff reconnection for failed platforms
- Progress thread fallback scoped to Slack only (PR #3488, v0.5.0)
- Include user-local bin paths in systemd unit PATH (PR #3527, v0.5.0)

Key changes from v0.5.0:
- Telegram Private Chat Topics — project-based conversations with skill binding per topic (PR #3163)
- Gateway AIAgent caching, transcript preservation on /compress (PR #3556)
- Prevent AsyncOpenAI/httpx cross-loop deadlock in gateway mode (PR #2701)

- [ ] **Step 7: Update providers.md**

New providers added in v0.4.0:
- **GitHub Copilot** — OAuth flow + token validation (PR #1924)
  - Source the exact config keys from model_tools.py
  - Authentication steps
- **Alibaba Cloud / DashScope** — (PR #1879 by @mchzimm)
  - Config keys, endpoint, model slugs
- **Kilo Code** — (PR #1673)
- **OpenCode Zen/Go** — (PR #1650, #1666)

New providers added in v0.5.0:
- **Hugging Face** — Full HF Inference API integration (PR #3419, #3440)
  - Setup wizard flow
  - Curated model picker mapping to OpenRouter analogues
  - Config keys (auth, endpoint)
  - Note: providers with 8+ curated models skip live /models probe
- **Z.AI GLM update** — glm-5-turbo added (PR #3095)

Provider changes in v0.5.0:
- Preserve `custom` provider, no longer silently remaps to openrouter (PR #2792)
- Root-level `provider` and `base_url` in config.yaml (PR #3112)
- Nous Portal model slug alignment with OpenRouter (PR #3253)
- Alibaba default endpoint and model list fix (PR #3484)
- MiniMax `/v1→/anthropic` auto-correction can now be overridden (PR #3553)
- OAuth token refresh migrated to platform.claude.com (PR #3246)
- Anthropic provider now uses model-native output limits (PR #3426)
- API timeout default increased to 1800s (PR #3431)

For each provider: config YAML snippet, supported model list, auth method, notes.

- [ ] **Step 8: Update messaging/telegram.md**

New in v0.5.0 — Private Chat Topics (PR #3163):
- What topics are and how they map to projects
- How to enable per-topic skill binding
- Fallback behavior when thread_id not found (PR #3390)

Verify existing setup steps against gateway/platforms/telegram.py.

- [ ] **Step 9: Update messaging/signal.md**

Introduced in v0.4.0 (PR #2206). If existing doc is a stub, flesh it out:
- Prerequisites (signal-cli or similar)
- Config keys sourced from the platform adapter
- Setup steps
- Known limitations

- [ ] **Step 10: Update messaging/mattermost.md**

Introduced in v0.4.0 (PR #1683):
- Prerequisites
- Config keys from platform adapter
- Setup steps

- [ ] **Step 11: Update messaging/matrix.md**

Introduced in v0.4.0 (PR #2166):
- Prerequisites
- Config keys from platform adapter
- e2ee access-token hardening (PR #3562, v0.5.0)
- Setup steps

- [ ] **Step 12: Update messaging/dingtalk.md**

Introduced in v0.4.0 (PR #1685):
- Config keys, webhook/robot setup
- Setup steps

- [ ] **Step 13: Update messaging/sms.md (Twilio)**

Introduced in v0.4.0 (PR #1688):
- Twilio prerequisites
- Config keys
- Setup steps

- [ ] **Step 14: Create messaging/webhook.md**

Introduced in v0.4.0 (PR #2584):
- What the webhook adapter is
- Config keys from platform adapter
- Example payload format
- Setup steps

- [ ] **Step 15: Verify and update other messaging docs (email, discord, slack, whatsapp, homeassistant)**

For each, read the platform adapter and verify existing doc is accurate. Add any v0.4.0/v0.5.0 changes.

- [ ] **Step 16: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add gateway.md providers.md messaging/
git commit -m "docs: update gateway, providers, and messaging adapters for v0.4.0 and v0.5.0"
```

---

## Task 4: Tools, Toolsets, Skills, Plugins, Hooks

**Owner:** Agent 4
**Source files to read:**
- `~/src/claw/hermes-agent/tools/` (all tool files)
- `~/src/claw/hermes-agent/toolsets.py`
- `~/src/claw/hermes-agent/toolset_distributions.py`
- `~/src/claw/hermes-agent/skills/` (skill system)
- `~/src/claw/hermes-agent/optional-skills/` (if present)
- `~/src/claw/hermes-agent/gateway/hooks.py`
- `~/src/claw/hermes-agent/gateway/builtin_hooks/`
- Plugin interface: search for `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end` in source

**Online validation:**
- https://hermes-agent.nousresearch.com/docs/user-guide/features/tools
- https://hermes-agent.nousresearch.com/docs/user-guide/features/skills

**Files to update:** `tools.md`, `toolsets.md`, `skills.md`, `plugins.md`, `hooks.md`

- [ ] **Step 1: Read current tool/plugin docs**

```bash
cat ~/src/claw/hermes-agent-docs/tools.md
cat ~/src/claw/hermes-agent-docs/toolsets.md
cat ~/src/claw/hermes-agent-docs/skills.md
cat ~/src/claw/hermes-agent-docs/plugins.md
cat ~/src/claw/hermes-agent-docs/hooks.md
```

- [ ] **Step 2: Read tools source**

```bash
ls ~/src/claw/hermes-agent/tools/
for f in ~/src/claw/hermes-agent/tools/*.py; do echo "=== $f ==="; head -40 $f; done
cat ~/src/claw/hermes-agent/toolsets.py | head -200
cat ~/src/claw/hermes-agent/toolset_distributions.py | head -200
```

- [ ] **Step 3: Read plugin and hook source**

```bash
ls ~/src/claw/hermes-agent/gateway/builtin_hooks/
cat ~/src/claw/hermes-agent/gateway/hooks.py
grep -r "pre_llm_call\|post_llm_call\|on_session_start\|on_session_end" ~/src/claw/hermes-agent/ --include="*.py" -l
```

- [ ] **Step 4: Read skills source**

```bash
ls ~/src/claw/hermes-agent/skills/
head -100 ~/src/claw/hermes-agent/skills/*.py 2>/dev/null | head -200
```

- [ ] **Step 5: Online validation**

Use WebFetch:
- https://hermes-agent.nousresearch.com/docs/user-guide/features/tools
- https://hermes-agent.nousresearch.com/docs/user-guide/features/skills

- [ ] **Step 6: Update plugins.md**

Plugin lifecycle hooks are now fully active (v0.5.0, PR #3542). Update/add:

The four hooks that fire in agent loop and CLI/gateway:
- `pre_llm_call(conversation_history, model_config)` — called before each LLM API call
- `post_llm_call(response, conversation_history)` — called after each LLM API call
- `on_session_start(session_id, config)` — called when a session starts
- `on_session_end(session_id, summary)` — called when a session ends

Source the exact function signatures from gateway/hooks.py or run_agent.py.

Plugin toolset visibility fix (PR #3457, v0.5.0): plugins are now discovered before reading plugin toolsets — document the plugin discovery order.

Show a complete example plugin with all four lifecycle hooks.

- [ ] **Step 7: Update hooks.md**

Add new lifecycle hooks section (v0.5.0):
- Distinguish between gateway builtin hooks (existing in v0.3.0) and new plugin lifecycle hooks
- Document when each hook fires in the execution flow
- Show how to use hooks for logging, metrics, cost tracking

- [ ] **Step 8: Update tools.md**

Verify the tool list is complete and accurate against tools/ directory.
Any new tools added in v0.4.0/v0.5.0 — search git log for "feat(tools)" or "add tool" commits.

Tool-use enforcement (v0.5.0):
- `GPT_TOOL_USE_GUIDANCE` system constant — what it does
- `agent.tool_use_enforcement` config key (PR #3551)
- Budget warning stripping from history (PR #3528)
- Configurable via `agent.tool_use_enforcement: true/false`

Document these in a "Tool Reliability" or "Tool-Use Enforcement" section.

- [ ] **Step 9: Update toolsets.md**

Plugin toolsets now visible in `hermes tools` and standalone processes (PR #3457).
Document how plugin-provided toolsets work alongside built-in toolsets.

- [ ] **Step 10: Update skills.md**

Verify skills system unchanged in v0.4.0/v0.5.0, or document any changes found in source.

- [ ] **Step 11: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add tools.md toolsets.md skills.md plugins.md hooks.md
git commit -m "docs: update tools, plugins, and hooks for v0.4.0 and v0.5.0"
```

---

## Task 5: API Server, MCP, ACP, Cron, Batch Processing

**Owner:** Agent 5
**Source files to read:**
- `~/src/claw/hermes-agent/mcp_serve.py`
- `~/src/claw/hermes-agent/cron/` (all files)
- `~/src/claw/hermes-agent/batch_runner.py`
- `~/src/claw/hermes-agent/acp_adapter/` (all files)
- `~/src/claw/hermes-agent/honcho_integration/` (all files)
- `~/src/claw/hermes-agent/mini_swe_runner.py`
- `~/src/claw/hermes-agent/rl_cli.py`
- Search for OpenAI-compatible API server files (api_server.py or similar)

**Online validation:**
- https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
- https://hermes-agent.nousresearch.com/docs/user-guide/features/cron

**Files to update:** `api-server.md`, `mcp.md`, `acp.md`, `cron.md`, `batch-processing.md`

- [ ] **Step 1: Read current API/MCP/cron docs**

```bash
cat ~/src/claw/hermes-agent-docs/api-server.md
cat ~/src/claw/hermes-agent-docs/mcp.md
cat ~/src/claw/hermes-agent-docs/acp.md
cat ~/src/claw/hermes-agent-docs/cron.md
cat ~/src/claw/hermes-agent-docs/batch-processing.md
```

- [ ] **Step 2: Find and read API server source**

```bash
find ~/src/claw/hermes-agent -name "*.py" | xargs grep -l "chat/completions\|api.server\|api_server" 2>/dev/null | head -10
# Then read the identified files
```

- [ ] **Step 3: Read MCP source**

```bash
cat ~/src/claw/hermes-agent/mcp_serve.py
# Search for hermes mcp subcommand handling
grep -r "hermes mcp\|mcp install\|mcp add\|OAuth" ~/src/claw/hermes-agent --include="*.py" -l | head -10
```

- [ ] **Step 4: Read cron source**

```bash
ls ~/src/claw/hermes-agent/cron/
cat ~/src/claw/hermes-agent/cron/*.py 2>/dev/null | head -300
```

- [ ] **Step 5: Read ACP and batch sources**

```bash
ls ~/src/claw/hermes-agent/acp_adapter/
cat ~/src/claw/hermes-agent/batch_runner.py | head -150
cat ~/src/claw/hermes-agent/mini_swe_runner.py | head -100
```

- [ ] **Step 6: Online validation**

Use WebFetch:
- https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp

- [ ] **Step 7: Update api-server.md**

The OpenAI-compatible API server is a major v0.4.0 feature (PR #1756, #2450, #2456, #2451, #2472). Document:

Starting the server:
```bash
hermes api-server  # (verify exact command from cli.py)
```

Endpoints:
- `POST /v1/chat/completions` — OpenAI-compatible inference
- `GET/POST /api/jobs` — REST API for cron job management (PR #2450)

Security features:
- Input limits
- Field whitelists
- SQLite-backed response persistence
- CORS origin protection

v0.5.0 additions:
- Cancel orphaned agent + true interrupt on SSE disconnect (PR #3427)
- Allow `Idempotency-Key` in CORS headers (PR #3530)
- Auto-repair jobs.json with invalid control characters (PR #3537)

Source the actual request/response schemas and CORS config keys from the server source code.

- [ ] **Step 8: Update mcp.md**

`hermes mcp` subcommands (PR #2465, v0.4.0):
- Source exact subcommand list from cli.py
- `hermes mcp install <server>` — install an MCP server
- `hermes mcp auth <server>` — authenticate via OAuth 2.1 PKCE
- Full OAuth 2.1 PKCE flow walkthrough (source from mcp_serve.py)
- Config file format for MCP servers (source exact YAML/JSON schema)

PKCE OAuth removal for Hermes-native flow (PR #3107, v0.5.0):
- Note: Hermes-native PKCE OAuth removed; MCP uses standard OAuth 2.1

- [ ] **Step 9: Update cron.md**

v0.5.0 fix: prevent recurring job re-fire on gateway crash/restart loop (PR #3396).
v0.4.0: `/api/jobs` REST API for cron management — link to api-server.md.

Verify rest of cron.md is accurate against cron/ source.

- [ ] **Step 10: Update acp.md**

Verify ACP content is accurate against acp_adapter/ source.
Add any v0.4.0/v0.5.0 changes.

- [ ] **Step 11: Update batch-processing.md**

Verify batch_runner.py and mini_swe_runner.py content matches docs.
Note: mini-swe-agent dependency removed in v0.5.0 (PR #2804) — Docker/Modal now inlined.

- [ ] **Step 12: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add api-server.md mcp.md acp.md cron.md batch-processing.md
git commit -m "docs: update API server, MCP, ACP, cron, batch for v0.4.0 and v0.5.0"
```

---

## Task 6: Security, Architecture, Contributing, Voice, Browser, Skins

**Owner:** Agent 6
**Source files to read:**
- `~/src/claw/hermes-agent/CONTRIBUTING.md`
- `~/src/claw/hermes-agent/gateway/pairing.py` (security: DM pairing)
- `~/src/claw/hermes-agent/gateway/status.py`
- `~/src/claw/hermes-agent/uv.lock` (supply chain — check litellm absence)
- `~/src/claw/hermes-agent/pyproject.toml` (dependency pinning)
- `~/src/claw/hermes-agent/tinker-atropos/` (if relevant to architecture)
- Voice: search for voice/TTS/STT in source
- Browser: search for browser/CDP/playwright in source
- `~/src/claw/hermes-agent/website/` (if relevant)

**Online validation:**
- https://hermes-agent.nousresearch.com/docs/user-guide/security

**Files to update:** `security.md`, `contributing.md`, `voice-mode.md`, `browser.md`, `architecture.md`

- [ ] **Step 1: Read current security/architecture/misc docs**

```bash
cat ~/src/claw/hermes-agent-docs/security.md
cat ~/src/claw/hermes-agent-docs/architecture.md
cat ~/src/claw/hermes-agent-docs/contributing.md
cat ~/src/claw/hermes-agent-docs/voice-mode.md
cat ~/src/claw/hermes-agent-docs/browser.md
```

- [ ] **Step 2: Read CONTRIBUTING.md from source**

```bash
cat ~/src/claw/hermes-agent/CONTRIBUTING.md
```

- [ ] **Step 3: Verify supply chain hardening**

```bash
# Verify litellm is absent
grep "litellm" ~/src/claw/hermes-agent/pyproject.toml ~/src/claw/hermes-agent/uv.lock 2>/dev/null | head -5
# Check dependency pinning
head -100 ~/src/claw/hermes-agent/pyproject.toml
```

- [ ] **Step 4: Read pairing/security source**

```bash
cat ~/src/claw/hermes-agent/gateway/pairing.py
```

- [ ] **Step 5: Find and read voice/browser source**

```bash
find ~/src/claw/hermes-agent -name "*.py" | xargs grep -l "voice\|tts\|whisper\|speech" 2>/dev/null | head -5
find ~/src/claw/hermes-agent -name "*.py" | xargs grep -l "browser\|playwright\|cdp\|chrome" 2>/dev/null | head -5
```

- [ ] **Step 6: Online validation**

Use WebFetch: https://hermes-agent.nousresearch.com/docs/user-guide/security

- [ ] **Step 7: Update security.md**

Supply chain hardening section (v0.5.0, PR #2796, #2810, #2812, #2816, #3073):
- Removed compromised `litellm` dependency — what replaced it
- All dependency version ranges pinned
- `uv.lock` with hashes (explain what this means for reproducibility)
- CI workflow scanning PRs for supply chain attack patterns
- CVE fixes (list the CVEs addressed from release notes)

Tirith block verdicts (v0.5.0, PR #3428):
- Verdicts are now approvable instead of hard-blocking
- What Tirith is and how it integrates with approval workflow

Tool-use enforcement as a security control (link to tools.md).

Verify existing DM pairing, command approval, and container isolation sections against source.

- [ ] **Step 8: Update architecture.md**

Key architectural changes from v0.4.0/v0.5.0:
- **Dependency removal**: mini-swe-agent removed (PR #2804), swe-rex replaced with native Modal SDK (PR #3538)
- **Plugin lifecycle hooks** now wired into agent loop (PR #3542) — update plugin architecture diagram/description
- **API server** added as a new entry point alongside CLI and gateway (v0.4.0)
- **Streaming schema v6** — `reasoning`, `reasoning_details`, `codex_reasoning_items` columns (PR #2974)
- **`get_hermes_home()`** consolidated (PR #3062) — note the canonical config path
- **Always-streaming** approach as architectural decision (PR #3120)
- **Plugin toolset discovery** ordering fixed (PR #3457)

- [ ] **Step 9: Update contributing.md**

Sync with CONTRIBUTING.md from source. Add any new contribution guidelines added in v0.4.0/v0.5.0 (supply chain scanning CI, coding standards, etc.).

- [ ] **Step 10: Update voice-mode.md**

Read voice source to verify accuracy. Update any changes from v0.4.0/v0.5.0.

- [ ] **Step 11: Update browser.md**

`/browser` command added in v0.4.0 (PR #2273, #1814). Update:
- Exact `/browser` CLI command documentation
- Interactive browser session workflow
- Underlying mechanism (CDP, Playwright, etc. — source from code)

- [ ] **Step 12: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add security.md architecture.md contributing.md voice-mode.md browser.md
git commit -m "docs: update security, architecture, contributing, voice, browser for v0.4.0 and v0.5.0"
```

---

## Task 7: Final Review, Online Cross-Check, and Push

**Owner:** Main agent (inline after Tasks 1–6 complete)
**Prerequisites:** Tasks 1–6 all committed

- [ ] **Step 1: Verify all docs updated to v0.5.0**

```bash
cd ~/src/claw/hermes-agent-docs
grep -r "v0\.3\.0\|v2026\.3\.17\|Current stable release" . --include="*.md" | grep -v changelog | grep -v "release_evaluation" | grep -v "review_plan" | grep -v "documentation_integrity"
```

Any hits (files still saying v0.3.0) that are NOT the changelog or historical review files should be updated.

- [ ] **Step 2: Cross-check changelog completeness**

```bash
# Count release note entries vs changelog entries
wc -l ~/src/claw/hermes-agent/RELEASE_v0.4.0.md ~/src/claw/hermes-agent/RELEASE_v0.5.0.md
wc -l ~/src/claw/hermes-agent-docs/changelog.md
```

- [ ] **Step 3: Online documentation cross-check**

Fetch the following and compare to updated docs:
- https://hermes-agent.nousresearch.com/docs/getting-started/quickstart
- https://hermes-agent.nousresearch.com/docs/user-guide/configuration
- https://hermes-agent.nousresearch.com/docs/changelog (if exists)

Note any discrepancies and resolve in favor of source code (source code is ground truth).

- [ ] **Step 4: Check for broken internal links**

```bash
cd ~/src/claw/hermes-agent-docs
grep -r "\[.*\](\./" . --include="*.md" | grep -v ".git"
# Manually verify any cross-file links exist
```

- [ ] **Step 5: Git status and final review**

```bash
cd ~/src/claw/hermes-agent-docs
git log --oneline -10
git status
```

- [ ] **Step 6: Push to GitHub**

```bash
cd ~/src/claw/hermes-agent-docs
git push origin main
```

- [ ] **Step 7: Verify push succeeded**

```bash
cd ~/src/claw/hermes-agent-docs
git log --oneline -5
git status
```

---

## Self-Review

### Spec coverage

| Requirement | Covered by |
|-------------|-----------|
| Fetch latest changes from origin | Done (already at main) |
| Review all source files | Tasks 1–6 each read specific source dirs |
| Review all documentation | Tasks 1–6 read all existing doc files |
| Validate/fact-check against source code | Each task reads source before updating docs |
| Add docs for released versions only | v0.4.0 and v0.5.0 only (v0.3.0 baseline) |
| Check online for additional documentation | Each task has online validation step |
| Parallel agent execution | 6 independent agents (Tasks 1–6) |
| Git commit | Each task ends with a granular commit |
| Push to GitHub | Task 7 Step 6 |

### No placeholders scan

All steps have specific file paths, grep commands, specific PR numbers, and concrete update instructions. No "TBD" or "implement later" patterns.

### Type consistency

No types or function signatures defined in early tasks that are referenced in later tasks (documentation tasks are independent).
