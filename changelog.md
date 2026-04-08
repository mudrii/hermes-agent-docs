# Changelog

All notable changes to Hermes Agent are documented here.

**Current stable release: v0.8.0** (v2026.4.8, April 8, 2026)

---

## v0.8.0 -- April 8, 2026

> The intelligence release — background task auto-notifications, free MiMo v2 Pro on Nous Portal, live model switching across all platforms, self-optimized GPT/Codex guidance, native Google AI Studio, smart inactivity timeouts, approval buttons, MCP OAuth 2.1, and 209 merged PRs with 82 resolved issues.

### Highlights

- **Background process auto-notifications (`notify_on_complete`)** — pass `notify_on_complete=True` to `terminal()` with `background=True` and the agent is automatically notified when the process exits. No polling needed. (PR [#5779](https://github.com/NousResearch/hermes-agent/pull/5779))
- **Live model switching (`/model`)** — the `/model` slash command is available again in CLI and all gateway platforms (Telegram, Discord, Slack, and others) for mid-session provider and model switching. (PR [#5181](https://github.com/NousResearch/hermes-agent/pull/5181), [#5742](https://github.com/NousResearch/hermes-agent/pull/5742))
- **Centralized logging (`hermes logs`)** — structured logging to `~/.hermes/logs/` with a new `hermes logs` command for tailing and filtering `agent.log`, `errors.log`, and `gateway.log`. (PR [#5430](https://github.com/NousResearch/hermes-agent/pull/5430), [#5426](https://github.com/NousResearch/hermes-agent/pull/5426))
- **Plugin API-level hooks** — new `pre_api_request`, `post_api_request`, `on_session_finalize`, and `on_session_reset` hooks added to the plugin system, alongside `ctx.register_cli_command()` for top-level subcommands. (PR [#5427](https://github.com/NousResearch/hermes-agent/pull/5427), [#5295](https://github.com/NousResearch/hermes-agent/pull/5295), [#6129](https://github.com/NousResearch/hermes-agent/pull/6129))
- **Google AI Studio (Gemini) native provider** — direct Gemini access via Google's AI Studio API with models.dev context length detection. (PR [#5577](https://github.com/NousResearch/hermes-agent/pull/5577))
- **Inactivity-based agent timeouts** — timeouts now track actual tool activity; actively working agents are never killed mid-task. (PR [#5389](https://github.com/NousResearch/hermes-agent/pull/5389), [#5440](https://github.com/NousResearch/hermes-agent/pull/5440))
- **Approval buttons on Slack and Telegram** — dangerous command approval via native platform buttons instead of typing `/approve`. (PR [#5890](https://github.com/NousResearch/hermes-agent/pull/5890), [#5975](https://github.com/NousResearch/hermes-agent/pull/5975))
- **MCP OAuth 2.1 PKCE + OSV malware scanning** — standards-compliant OAuth for MCP servers plus automatic package malware scanning. (PR [#5420](https://github.com/NousResearch/hermes-agent/pull/5420), [#5305](https://github.com/NousResearch/hermes-agent/pull/5305))
- **Security hardening** — consolidated SSRF protections, timing attack mitigations, tar traversal prevention, credential leakage guards, cron path traversal hardening, and cross-session isolation. (PR [#5944](https://github.com/NousResearch/hermes-agent/pull/5944), [#5613](https://github.com/NousResearch/hermes-agent/pull/5613))
- **Bundled skills synced to all profiles** on `hermes update`. (PR [#5795](https://github.com/NousResearch/hermes-agent/pull/5795))
- **Per-platform disabled skills** respected in Telegram skills menu. (PR [#4799](https://github.com/NousResearch/hermes-agent/pull/4799))
- **`hermes auth remove` clears env-seeded credentials** permanently from `~/.hermes/auth.json`.
- **`--yolo` flag** no longer silently dropped before the `chat` subcommand. (PR [#5145](https://github.com/NousResearch/hermes-agent/pull/5145))
- **New bundled skills** — `creative/popular-web-designs`, `creative/p5js`, `creative/manim-video`, `research/llm-wiki`, `research/research-paper-writing`, `software-development/gitnexus-explorer`.
- **Supermemory memory provider** — new memory plugin with multi-container and per-user scoping. (PR [#5737](https://github.com/NousResearch/hermes-agent/pull/5737))
- **Windows native image paste** support. (PR [#5917](https://github.com/NousResearch/hermes-agent/pull/5917))
- **Matrix Tier 1** — reactions, read receipts, rich formatting, and room management. (PR [#5275](https://github.com/NousResearch/hermes-agent/pull/5275))
- **Jittered backoff for API retries** — exponential backoff with jitter replaces fixed retry intervals, improving resilience under rate limiting. (PR [#6048](https://github.com/NousResearch/hermes-agent/pull/6048))
- **Smart thinking block signature management** — Anthropic thinking block signatures are preserved and managed correctly across multi-turn tool calls. (PR [#6112](https://github.com/NousResearch/hermes-agent/pull/6112))
- **Honcho toolset removed** — Honcho now ships as a memory provider plugin in `plugins/memory/honcho/` rather than as a built-in toolset. Existing users must install via `hermes plugins install honcho`. (PR [#5295](https://github.com/NousResearch/hermes-agent/pull/5295))

---

## v0.7.0 -- April 3, 2026

> The resilience release -- pluggable memory providers, same-provider credential pools, Camofox anti-detection browser support, inline diff previews, API session continuity, gateway hardening, and deep security fixes across 168 PRs and 46 resolved issues.

### Highlights

- **Pluggable memory providers** -- released `MemoryProvider` interface with a built-in provider plus one optional external provider, including Honcho parity and additional provider plugins
- **Same-provider credential pools** -- rotate among multiple credentials for the same provider with `least_used` selection, 401 refresh-and-rotate behavior, and preserved pool state through fallback routing
- **Camofox browser backend** -- local anti-detection browser mode via `CAMOFOX_URL`, with persistent sessions and VNC URL discovery
- **Inline diff previews** -- file writes and patches now emit inline diffs in the activity feed
- **API server session continuity** -- `/v1/chat/completions` now supports optional continuity via `X-Hermes-Session-Id`; `/v1/responses` persists chained responses in the shared SessionDB
- **ACP client-provided MCP servers** -- editor integrations can pass their own MCP servers through ACP so Hermes exposes them as additional tools
- **Gateway hardening** -- approval routing, stuck-session handling, webhook behavior, Discord approvals, and multi-platform reliability all improved
- **Secret exfiltration blocking** -- browser URLs and tool outputs are scanned more aggressively, and additional credential directories are protected

### Core Agent & Architecture

- Released `MemoryProvider` ABC and `MemoryManager` orchestration for built-in memory plus one external provider
- Honcho restored as the reference memory-provider plugin with profile-scoped host and peer resolution
- Same-provider credential pools now support `least_used`, preserved pool state through smart routing, provider reset-window handling, and per-turn primary runtime restoration
- New `developer` role guidance for GPT-5/Codex models
- Improved Anthropic long-context 429 handling, provider recovery, and thinking-block preservation across tool turns
- Context-exceeded errors now include clearer guidance, and compression death-spiral handling was hardened

### Messaging Platforms

**Gateway core:** `/approve` and `/deny` route correctly while an agent is blocked on approval; resumed agents keep the pending tool result instead of losing it.

**Discord:** Button-based approval UI, configurable `discord.reactions`, and unauthorized-user reaction/thread suppression.

**Slack:** `slack.reply_in_thread` config support for threaded replies.

**WhatsApp:** Group-chat `require_mention` enforcement.

**Webhook:** Home-channel prompt skipped and tool-progress output disabled for webhook adapters.

**API server / Open WebUI:** Streaming tool progress and persistent session continuity via `X-Hermes-Session-Id`.

### Tool System

- **Browser:** Camofox local backend, local-backend SSRF bypass, `browser.allow_private_urls`, and VNC discovery
- **File operations:** inline diffs, stale-file detection on write/patch, and refreshed staleness tracking after edits
- **ACP:** client-provided MCP servers from editors are exposed as Hermes tools
- **Skills:** `research-paper-writing` replaces `ml-paper-writing`, and the docs site gained a skills browse/search page

### Security

- Block secret exfiltration through browser URLs and model-visible tool output
- Redact secrets from `execute_code` sandbox output
- Protect additional credential directories: `.docker`, `.azure`, `.config/gh`
- Add GitHub OAuth token patterns and related redaction improvements
- Preserve existing tirith, approval, container, and prompt-injection defenses

### Documentation

- Public docs now cover released memory providers, credential pools, Camofox browser mode, and API/Open WebUI tool-progress streaming
- This docs repo was updated in the `v2026.4.3` audit pass to correct stale version surfaces and fill remaining release-critical gaps

### Contributors

**Core:** @teknium1 -- 168 PRs across the release window.

**Full Changelog:** [v2026.3.30...v2026.4.3](https://github.com/NousResearch/hermes-agent/compare/v2026.3.30...v2026.4.3)

---

## v0.6.0 -- March 30, 2026

> The multi-instance release — Profiles for running isolated agent instances, MCP server mode, Docker container, fallback provider chains, two new messaging platforms (Feishu/Lark and WeCom), Telegram webhook mode, Slack multi-workspace OAuth, 95 PRs and 16 resolved issues in 2 days.

### Highlights

- **Profiles** — Run multiple isolated Hermes instances with separate config, memory, sessions, and gateway service (`hermes profile create/list/use/delete/export/import/rename`)
- **MCP Server Mode** — Expose Hermes sessions to Claude Desktop, Cursor, VS Code via `hermes mcp serve` (released v0.6.0 server transport: stdio)
- **Docker Container** — Official Dockerfile for CLI and gateway modes with volume-mounted config
- **Fallback Provider Chain** — Automatic failover across multiple inference providers via `fallback_providers` in config.yaml
- **Feishu/Lark Support** — Full gateway adapter: event subscriptions, message cards, group chat, attachments, interactive card callbacks
- **WeCom Support** — Enterprise WeChat adapter: text/image/voice messages, group chats, callback verification
- **Slack Multi-Workspace OAuth** — Connect one gateway to multiple Slack workspaces via OAuth token file
- **Telegram Webhook Mode** — Production-grade webhook alternative to polling, plus group mention gating (always / mention-only / regex)
- **Exa Search Backend** — Alternative to Firecrawl/DuckDuckGo with semantic search (`EXA_API_KEY`)
- **Skills & Credentials on Remote Backends** — Mount skill dirs and credential files into Modal/Docker containers
- **Plugin enable/disable** — `hermes plugins enable/disable <name>` without deleting files
- **Plugin inject_message** — Plugins can inject messages into the conversation stream via `ctx.inject_message()`

### Core Agent & Architecture

- Ordered fallback provider chain (`fallback_providers` in config.yaml) with automatic failover on 5xx/429/network errors
- Stop silent OpenRouter fallback — clear error when no provider is configured
- Subagent status reporting: `completed` when summary exists instead of generic failure
- Session log file updated during compression, preventing stale file references
- Omit empty `tools` param (fixes strict provider compatibility)
- Configurable approval timeouts (`approvals.timeout` in config.yaml)

### Messaging Platforms

**New:** Feishu/Lark (#3799, #3817), WeCom/Enterprise WeChat (#3847)

**Telegram:** Webhook mode (#3880), group mention gating (#3870), deleted reply target graceful handling (#3858)

**Discord:** Processing reactions (#3871), DISCORD_IGNORE_NO_MENTION (#3640), deferred "thinking..." cleanup (#3674)

**Slack:** Multi-workspace OAuth token file (#3903)

**WhatsApp:** Persistent aiohttp session (#3818), LID↔phone alias fix (#3830)

**Matrix:** Native MSC3245 voice messages (#3877)

**Mattermost:** Configurable mention behavior (#3664)

**Signal:** URL-encoded phone numbers (#3670) — by @kshitijk4poor

**Email:** Close SMTP/IMAP on error (#3804)

**Gateway Core:** Atomic config.yaml writes (#3800), cron delivery labels (#3860), BOOT.md hook (#3733), TTY guard (#3933)

### Tool System

- MCP Server Mode: `hermes mcp serve` exposes sessions/search to MCP clients (#3795)
- Exa search backend (#3648)
- Mount skill directories and credential files into Modal/Docker containers (#3890, #3671)
- Dynamic MCP tool discovery for Hermes as an MCP client (responds to `notifications/tools/list_changed`) (#3812)
- Terminal: preserve partial output on timeout (#3868)

### Skills & Plugins

- External skill directories via `skills.external_dirs` in config.yaml (#3678)
- Plugin enable/disable: `hermes plugins enable/disable <name>` (#3747)
- Plugin message injection: `ctx.inject_message()` — by @winglian (#3778)
- New skills: memento-flashcards, songwriting-and-ai-music, SiYuan Note, Scrapling, one-three-one-rule

### Security

- Hardened dangerous command detection + file tool path guards for `/etc/`, `/boot/`, Docker socket (#3872)
- Sensitive path write checks in file tools (#3859)
- Expanded secret redaction: ElevenLabs, Tavily, Exa (#3920)
- Vision file type enforcement (#3845)
- Skill category path traversal blocking (#3844)

### Notable Bug Fixes

- OpenClaw migration: fix model config dict overwritten with string — by @0xbyt4 (#3924)
- Telegram: gracefully handle replies to deleted messages (#3858)
- Discord: properly clean up deferred "thinking..." indicator (#3674)
- WhatsApp: LID↔phone alias resolution in allowlists (#3830)
- Signal: URL-encode phone numbers for international formats (#3670)
- Email: close SMTP/IMAP connections on error (#3804)
- Tool schema: ensure `name` field always present, fixes `KeyError: 'name'` (#3811)
- Provider switch: clear stale `api_mode` when switching via `hermes model` (#3857)
- `_safe_print` ValueError: no more crashes on closed stdout (#3843)

### Contributors

**Core:** @teknium1 — 90 PRs

**Community:** @kshitijk4poor (3 PRs: Signal #3670, parallel-cli #3673, status bar #3883), @winglian (1 PR: plugin inject_message #3778), @binhnt92 (1 PR: audio retry #3401), @0xbyt4 (1 PR: migration fix #3924)

**Full Changelog:** [v2026.3.28...v2026.3.30](https://github.com/NousResearch/hermes-agent/compare/v2026.3.28...v2026.3.30)

---

## v0.5.0 (v2026.3.28) -- March 28, 2026

> The hardening release -- Hugging Face provider, /model command overhaul, Telegram Private Chat Topics, native Modal SDK, plugin lifecycle hooks, tool-use enforcement for GPT models, Nix flake, 50+ security and reliability fixes, and a comprehensive supply chain audit.

---

### Core Agent & Architecture

#### New Provider: Hugging Face

First-class Hugging Face Inference API integration with auth, setup wizard, and curated agentic model picker. Providers with 8+ curated models skip the live `/models` probe for faster startup. HF models map to OpenRouter agentic defaults as equivalents. ([#3419](https://github.com/NousResearch/hermes-agent/pull/3419), [#3440](https://github.com/NousResearch/hermes-agent/pull/3440))

#### Provider & Model Improvements

- `/model` command overhaul -- extracted shared `switch_model()` pipeline for CLI and gateway, custom endpoint support, provider-aware routing ([#2795](https://github.com/NousResearch/hermes-agent/pull/2795), [#2799](https://github.com/NousResearch/hermes-agent/pull/2799))
- Removed `/model` slash command from CLI and gateway in favor of `hermes model` subcommand ([#3080](https://github.com/NousResearch/hermes-agent/pull/3080))
- Preserve `custom` provider instead of silently remapping to `openrouter` ([#2792](https://github.com/NousResearch/hermes-agent/pull/2792))
- Read root-level `provider` and `base_url` from config.yaml into model config ([#3112](https://github.com/NousResearch/hermes-agent/pull/3112))
- Align Nous Portal model slugs with OpenRouter naming ([#3253](https://github.com/NousResearch/hermes-agent/pull/3253))
- Nous Portal expanded to 400+ models
- Migrate OAuth token refresh to `platform.claude.com` with fallback ([#3246](https://github.com/NousResearch/hermes-agent/pull/3246))
- Fix Alibaba provider default endpoint and model list ([#3484](https://github.com/NousResearch/hermes-agent/pull/3484))
- Allow MiniMax users to override `/v1` → `/anthropic` auto-correction ([#3553](https://github.com/NousResearch/hermes-agent/pull/3553))

#### Agent Loop & Conversation

- **Improved OpenAI model reliability** -- `GPT_TOOL_USE_GUIDANCE` prevents GPT models from describing actions instead of calling tools; automatic budget warning stripping from history eliminates cross-turn tool avoidance ([#3528](https://github.com/NousResearch/hermes-agent/pull/3528))
- **Surface lifecycle events** -- all retry, fallback, and compression events now surface to the user as formatted messages ([#3153](https://github.com/NousResearch/hermes-agent/pull/3153))
- **Anthropic output limits** -- replaced hardcoded 16K `max_tokens` with per-model native output limits (128K for Opus 4.6, 64K for Sonnet 4.6); fixes "Response truncated" and thinking-budget exhaustion ([#3426](https://github.com/NousResearch/hermes-agent/pull/3426), [#3444](https://github.com/NousResearch/hermes-agent/pull/3444))
- Always prefer streaming for API calls to prevent hung subagents ([#3120](https://github.com/NousResearch/hermes-agent/pull/3120))
- Give subagents independent iteration budgets ([#3004](https://github.com/NousResearch/hermes-agent/pull/3004))
- Update `api_key` in `_try_activate_fallback` for subagent auth ([#3103](https://github.com/NousResearch/hermes-agent/pull/3103))
- Count compression restarts toward retry limit ([#3070](https://github.com/NousResearch/hermes-agent/pull/3070))
- Validate empty user messages to prevent Anthropic API 400 errors ([#3322](https://github.com/NousResearch/hermes-agent/pull/3322))
- Increase API timeout default from 900s to 1800s for slow-thinking models ([#3431](https://github.com/NousResearch/hermes-agent/pull/3431))
- Prevent AsyncOpenAI/httpx cross-loop deadlock in gateway mode ([#2701](https://github.com/NousResearch/hermes-agent/pull/2701)) by @ctlst

#### Plugin Lifecycle Hooks

`pre_llm_call`, `post_llm_call`, `on_session_start`, and `on_session_end` hooks now fire in the agent loop and CLI/gateway, completing the plugin hook system. Plugins can register slash commands and extend the TUI. ([#3542](https://github.com/NousResearch/hermes-agent/pull/3542))

- Fix plugin toolsets invisible in `hermes tools` and standalone processes ([#3457](https://github.com/NousResearch/hermes-agent/pull/3457))

#### Architecture & Dependencies

- **Remove mini-swe-agent dependency** -- inline Docker and Modal backends directly ([#2804](https://github.com/NousResearch/hermes-agent/pull/2804))
- **Replace swe-rex with native Modal SDK** -- `Sandbox.create.aio` + `exec.aio`, eliminating tunnels ([#3538](https://github.com/NousResearch/hermes-agent/pull/3538))
- Consolidate `get_hermes_home()` and `parse_reasoning_effort()` ([#3062](https://github.com/NousResearch/hermes-agent/pull/3062))
- Remove unused Hermes-native PKCE OAuth flow ([#3107](https://github.com/NousResearch/hermes-agent/pull/3107))
- Remove ~100 unused imports across 55 files ([#3016](https://github.com/NousResearch/hermes-agent/pull/3016))

#### Streaming & Reasoning

- **Persist reasoning across gateway session turns** -- new schema v6 columns: `reasoning`, `reasoning_details`, `codex_reasoning_items` ([#2974](https://github.com/NousResearch/hermes-agent/pull/2974))
- Preserve reasoning fields in `rewrite_transcript` ([#3311](https://github.com/NousResearch/hermes-agent/pull/3311))
- Skip duplicate callback for `<think>`-extracted reasoning during streaming ([#3116](https://github.com/NousResearch/hermes-agent/pull/3116))

#### Session & Memory

- **`/resume` CLI handler** -- session log truncation guard, `reopen_session` API ([#3315](https://github.com/NousResearch/hermes-agent/pull/3315))
- **Session search recent sessions mode** -- omit query to browse recent sessions with titles, previews, and timestamps ([#2533](https://github.com/NousResearch/hermes-agent/pull/2533))
- **Session config surfacing** on `/new`, `/reset`, and auto-reset ([#3321](https://github.com/NousResearch/hermes-agent/pull/3321))
- **Third-party session isolation** -- `--source` flag for isolating sessions by origin ([#3255](https://github.com/NousResearch/hermes-agent/pull/3255))
- Surface silent SessionDB failures that cause session data loss ([#2999](https://github.com/NousResearch/hermes-agent/pull/2999))

#### Context Compression

- Replace dead `summary_target_tokens` with ratio-based scaling ([#2554](https://github.com/NousResearch/hermes-agent/pull/2554))
- Expose `compression.target_ratio`, `protect_last_n`, and `threshold` in `DEFAULT_CONFIG`
- Restore sane defaults and cap summary at 12K tokens
- Preserve transcript on `/compress` and hygiene compression ([#3556](https://github.com/NousResearch/hermes-agent/pull/3556))

---

### Nix Flake

Full uv2nix build, NixOS module with persistent container mode, auto-generated config keys from Python source, and suffix PATHs for agent-friendliness. Contributed by @alt-glitch. ([#20](https://github.com/NousResearch/hermes-agent/pull/20), [#3274](https://github.com/NousResearch/hermes-agent/pull/3274), [#3061](https://github.com/NousResearch/hermes-agent/pull/3061))

- Run with: `nix run github:NousResearch/hermes-agent`
- NixOS module: `services.hermes-agent.enable = true` with declarative `settings`, `environmentFiles`, and `mcpServers`
- Two service modes: native systemd (default) and OCI container with persistent writable layer
- Supports `x86_64-linux`, `aarch64-linux`, and `aarch64-darwin`

---

### Messaging Platforms (Gateway)

#### Telegram

- **Private Chat Topics** -- project-based conversations with functional skill binding per topic, enabling isolated workflows within a single Telegram chat ([#3163](https://github.com/NousResearch/hermes-agent/pull/3163))
- Auto-discover fallback IPs via DNS-over-HTTPS when `api.telegram.org` is unreachable ([#3376](https://github.com/NousResearch/hermes-agent/pull/3376))
- Configurable reply threading mode ([#2907](https://github.com/NousResearch/hermes-agent/pull/2907))

#### Discord

- Stop phantom typing indicator after agent turn completes ([#3003](https://github.com/NousResearch/hermes-agent/pull/3003))

#### Slack

- Send tool call progress messages to correct Slack thread ([#3063](https://github.com/NousResearch/hermes-agent/pull/3063))

#### WhatsApp

- Download documents, audio, and video media from messages ([#2978](https://github.com/NousResearch/hermes-agent/pull/2978))

#### Matrix

- Add missing Matrix entry in `PLATFORMS` dict ([#3473](https://github.com/NousResearch/hermes-agent/pull/3473))
- Harden e2ee access-token handling ([#3562](https://github.com/NousResearch/hermes-agent/pull/3562))

#### Gateway Core

- **Config-gated `/verbose` command** -- toggle tool output verbosity from messaging platforms ([#3262](https://github.com/NousResearch/hermes-agent/pull/3262))
- **Background review notifications** delivered to user chat ([#3293](https://github.com/NousResearch/hermes-agent/pull/3293))
- **Retry transient send failures** and notify user on exhaustion ([#3288](https://github.com/NousResearch/hermes-agent/pull/3288))
- Recover from hung agents -- `/stop` hard-kills session lock ([#3104](https://github.com/NousResearch/hermes-agent/pull/3104))
- Thread-safe `SessionStore` -- protect `_entries` with `threading.Lock` ([#3052](https://github.com/NousResearch/hermes-agent/pull/3052))
- Fix gateway token double-counting with cached agents ([#3306](https://github.com/NousResearch/hermes-agent/pull/3306), [#3317](https://github.com/NousResearch/hermes-agent/pull/3317))
- Include user-local bin paths in systemd unit PATH ([#3527](https://github.com/NousResearch/hermes-agent/pull/3527))

---

### CLI & User Experience

- **Configurable busy input mode** + fix `/queue` always working ([#3298](https://github.com/NousResearch/hermes-agent/pull/3298))
- **Preserve user input on multiline paste** ([#3065](https://github.com/NousResearch/hermes-agent/pull/3065))
- **Tool generation callback** -- streaming "preparing terminal..." updates during tool argument generation
- Eliminate "Event loop is closed" / "Press ENTER to continue" during idle sessions ([#3398](https://github.com/NousResearch/hermes-agent/pull/3398))
- Fix status bar shows 26K instead of 260K for token counts with trailing zeros ([#3024](https://github.com/NousResearch/hermes-agent/pull/3024))
- Fix status bar duplicates and degrades during long sessions ([#3291](https://github.com/NousResearch/hermes-agent/pull/3291))
- Fix reasoning box rendering 3x during tool-calling loops ([#3405](https://github.com/NousResearch/hermes-agent/pull/3405))

#### Setup & Configuration

- Harden `hermes update` against diverged history, non-main branches, and gateway edge cases ([#3492](https://github.com/NousResearch/hermes-agent/pull/3492))
- OpenClaw migration overwrites defaults -- fixed; setup wizard now skips imported sections ([#3282](https://github.com/NousResearch/hermes-agent/pull/3282))
- Use `sys.executable` for pip in update commands to fix PEP 668 "externally-managed-environment" error ([#3099](https://github.com/NousResearch/hermes-agent/pull/3099))
- Add macOS Homebrew paths to browser and terminal PATH resolution ([#2713](https://github.com/NousResearch/hermes-agent/pull/2713))
- Add explicit `hermes-api-server` toolset for API server platform ([#3304](https://github.com/NousResearch/hermes-agent/pull/3304))

---

### Tool System

#### API Server

- Idempotency-Key support, body size limit, and OpenAI error envelope ([#2903](https://github.com/NousResearch/hermes-agent/pull/2903))
- Cancel orphaned agent + true interrupt on SSE disconnect ([#3427](https://github.com/NousResearch/hermes-agent/pull/3427))

#### MCP

- MCP toolset resolution for runtime and config ([#3252](https://github.com/NousResearch/hermes-agent/pull/3252))
- Add MCP tool name collision protection ([#3077](https://github.com/NousResearch/hermes-agent/pull/3077))

---

### Skills Ecosystem

- **Env var passthrough** for skills and user config -- skills can declare environment variables to pass through ([#2807](https://github.com/NousResearch/hermes-agent/pull/2807))
- Cache skills prompt with shared `skill_utils` module for faster TTFT ([#3421](https://github.com/NousResearch/hermes-agent/pull/3421))
- Use Git Trees API to prevent silent subdirectory loss during install ([#2995](https://github.com/NousResearch/hermes-agent/pull/2995))
- **New skills:** G0DM0D3 godmode jailbreaking skill ([#3157](https://github.com/NousResearch/hermes-agent/pull/3157)), Docker management ([#3060](https://github.com/NousResearch/hermes-agent/pull/3060)), OpenClaw migration v2 with 17 new modules ([#2906](https://github.com/NousResearch/hermes-agent/pull/2906))

---

### Security & Reliability

#### Supply Chain Hardening

- **Removed compromised `litellm`/`typer`/`platformdirs` dependencies** ([#2796](https://github.com/NousResearch/hermes-agent/pull/2796))
- **Pinned all dependency version ranges** ([#2810](https://github.com/NousResearch/hermes-agent/pull/2810))
- **Regenerated `uv.lock` with hashes**, use lockfile in setup ([#2812](https://github.com/NousResearch/hermes-agent/pull/2812))
- Bumped dependencies to fix CVEs + regenerated `uv.lock` ([#3073](https://github.com/NousResearch/hermes-agent/pull/3073))
- Supply chain audit CI workflow for PR scanning ([#2816](https://github.com/NousResearch/hermes-agent/pull/2816))

#### Security

- SSRF protection added to `browser_navigate` ([#3058](https://github.com/NousResearch/hermes-agent/pull/3058))
- Restrict subagent toolsets to parent's enabled set ([#3269](https://github.com/NousResearch/hermes-agent/pull/3269))
- Prevent zip-slip path traversal in self-update ([#3250](https://github.com/NousResearch/hermes-agent/pull/3250))
- Normalize input before dangerous command detection ([#3260](https://github.com/NousResearch/hermes-agent/pull/3260))

#### Reliability

- SQLite WAL write-lock contention causing 15-20s TUI freeze -- fixed ([#3385](https://github.com/NousResearch/hermes-agent/pull/3385))
- SQLite concurrency hardening + session transcript integrity ([#3249](https://github.com/NousResearch/hermes-agent/pull/3249))
- Prevent recurring cron job re-fire on gateway crash/restart loop ([#3396](https://github.com/NousResearch/hermes-agent/pull/3396))

---

### Performance

- TTFT startup optimizations ([#3395](https://github.com/NousResearch/hermes-agent/pull/3395))
- Cache skills prompt with shared `skill_utils` module ([#3421](https://github.com/NousResearch/hermes-agent/pull/3421))

---

### Contributors

**Core:** @teknium1 -- 157 PRs covering the full scope of this release.

**Community contributors:** @alt-glitch (Nix flake, uv2nix build, NixOS module, persistent container mode, auto-generated config keys, suffix PATHs), @ctlst (AsyncOpenAI/httpx cross-loop deadlock fix), @memosr (send_message_tool HTTP timeouts), @mehmoodosman (Discord docs fix for Public Bot setting).

**Full changelog:** [v2026.3.23...v2026.3.28](https://github.com/NousResearch/hermes-agent/compare/v2026.3.23...v2026.3.28)

---

## v0.4.0 (v2026.3.23) -- March 23, 2026

> The platform expansion release -- OpenAI-compatible API server, 6 new messaging adapters, 4 new inference providers, MCP server management with OAuth 2.1, @ context references, gateway prompt caching, streaming enabled by default, and a sweeping reliability pass with 200+ bug fixes.

---

### Core Agent & Architecture

#### New Providers

- **GitHub Copilot** -- full OAuth auth, API routing, token validation, and 400k context ([#1924](https://github.com/NousResearch/hermes-agent/pull/1924), [#1896](https://github.com/NousResearch/hermes-agent/pull/1896), [#1879](https://github.com/NousResearch/hermes-agent/pull/1879) by @mchzimm, [#2507](https://github.com/NousResearch/hermes-agent/pull/2507))
- **Alibaba Cloud / DashScope** -- full integration with DashScope v1 runtime, model dot preservation, 401 auth fixes ([#1673](https://github.com/NousResearch/hermes-agent/pull/1673), [#2332](https://github.com/NousResearch/hermes-agent/pull/2332), [#2459](https://github.com/NousResearch/hermes-agent/pull/2459))
- **Kilo Code** -- first-class inference provider ([#1666](https://github.com/NousResearch/hermes-agent/pull/1666))
- **OpenCode Zen and OpenCode Go** -- new provider backends ([#1650](https://github.com/NousResearch/hermes-agent/pull/1650), [#2393](https://github.com/NousResearch/hermes-agent/pull/2393) by @0xbyt4)
- **NeuTTS** -- local TTS provider backend with built-in setup flow, replacing the old optional skill ([#1657](https://github.com/NousResearch/hermes-agent/pull/1657), [#1664](https://github.com/NousResearch/hermes-agent/pull/1664))

#### Provider Improvements

- **Eager fallback** to backup model on rate-limit errors ([#1730](https://github.com/NousResearch/hermes-agent/pull/1730))
- **Context length detection overhaul** -- models.dev integration, provider-aware resolution, fuzzy matching for custom endpoints, `/v1/props` for llama.cpp ([#2158](https://github.com/NousResearch/hermes-agent/pull/2158), [#2051](https://github.com/NousResearch/hermes-agent/pull/2051), [#2403](https://github.com/NousResearch/hermes-agent/pull/2403))
- **Model catalog updates** -- gpt-5.4-mini, gpt-5.4-nano, healer-alpha, haiku-4.5, minimax-m2.7, claude 4.6 at 1M context ([#1913](https://github.com/NousResearch/hermes-agent/pull/1913), [#1915](https://github.com/NousResearch/hermes-agent/pull/1915), [#1900](https://github.com/NousResearch/hermes-agent/pull/1900), [#2155](https://github.com/NousResearch/hermes-agent/pull/2155), [#2474](https://github.com/NousResearch/hermes-agent/pull/2474))
- **`custom_models.yaml`** for user-managed model additions ([#2214](https://github.com/NousResearch/hermes-agent/pull/2214))
- **`model.base_url` in config.yaml** plus `api_mode` override for Responses API ([#2330](https://github.com/NousResearch/hermes-agent/pull/2330))
- Fix: prevent Anthropic token leaking to third-party `anthropic_messages` providers ([#2389](https://github.com/NousResearch/hermes-agent/pull/2389))
- Fix: case-insensitive model family matching ([#2350](https://github.com/NousResearch/hermes-agent/pull/2350))

#### Agent Loop

- **Gateway prompt caching** -- cache AIAgent instances per session, preserving Anthropic prompt cache across turns ([#2282](https://github.com/NousResearch/hermes-agent/pull/2282), [#2284](https://github.com/NousResearch/hermes-agent/pull/2284), [#2361](https://github.com/NousResearch/hermes-agent/pull/2361))
- **Context compression overhaul** -- structured summaries with iterative updates, token-budget tail protection, configurable `summary_base_url` and fallback model support ([#2323](https://github.com/NousResearch/hermes-agent/pull/2323), [#1727](https://github.com/NousResearch/hermes-agent/pull/1727), [#2224](https://github.com/NousResearch/hermes-agent/pull/2224))
- **Pre-call sanitization and post-call tool guardrails** ([#1732](https://github.com/NousResearch/hermes-agent/pull/1732))
- **Auto-recover** from provider-rejected `tool_choice` by retrying without ([#2174](https://github.com/NousResearch/hermes-agent/pull/2174))
- **Background memory/skill review** replaces inline nudges ([#2235](https://github.com/NousResearch/hermes-agent/pull/2235))
- **SOUL.md as primary agent identity** instead of hardcoded default ([#1922](https://github.com/NousResearch/hermes-agent/pull/1922))
- Fix: prevent silent tool result loss during context compression ([#1993](https://github.com/NousResearch/hermes-agent/pull/1993))
- Fix: prevent stuck agent loop on malformed tool calls ([#2114](https://github.com/NousResearch/hermes-agent/pull/2114))
- Fix: `compression_attempts` resets each iteration -- allowed unlimited compressions ([#1723](https://github.com/NousResearch/hermes-agent/pull/1723))
- Fix: compressor summary role violated consecutive-role constraint ([#1720](https://github.com/NousResearch/hermes-agent/pull/1720), [#1743](https://github.com/NousResearch/hermes-agent/pull/1743))
- Fix: strip ANSI at the source before output reaches the model ([#2115](https://github.com/NousResearch/hermes-agent/pull/2115))

#### Session & Memory

- **Session search** and management slash commands ([#2198](https://github.com/NousResearch/hermes-agent/pull/2198))
- **Auto session titles** and `.hermes.md` project config ([#1712](https://github.com/NousResearch/hermes-agent/pull/1712))
- Fix: concurrent memory writes silently drop entries -- added file locking ([#1726](https://github.com/NousResearch/hermes-agent/pull/1726))
- Fix: prevent `session_search` crash when no sessions exist ([#2194](https://github.com/NousResearch/hermes-agent/pull/2194))
- Fix: reset token counters on new session for accurate usage display ([#2101](https://github.com/NousResearch/hermes-agent/pull/2101) by @InB4DevOps)
- Fix: quiet mode with `--resume` now passes conversation_history ([#2357](https://github.com/NousResearch/hermes-agent/pull/2357))

#### Honcho Memory

- Honcho config fixes and @ context reference integration ([#2343](https://github.com/NousResearch/hermes-agent/pull/2343))
- Self-hosted / Docker configuration documentation ([#2475](https://github.com/NousResearch/hermes-agent/pull/2475))

---

### Messaging Platforms (Gateway)

#### New Platform Adapters

- **Signal Messenger** -- full adapter with attachment handling, group message filtering, and Note to Self echo-back protection ([#2206](https://github.com/NousResearch/hermes-agent/pull/2206), [#2400](https://github.com/NousResearch/hermes-agent/pull/2400), [#2297](https://github.com/NousResearch/hermes-agent/pull/2297))
- **DingTalk** -- adapter with gateway wiring and setup docs ([#1685](https://github.com/NousResearch/hermes-agent/pull/1685), [#1690](https://github.com/NousResearch/hermes-agent/pull/1690), [#1692](https://github.com/NousResearch/hermes-agent/pull/1692))
- **SMS (Twilio)** ([#1688](https://github.com/NousResearch/hermes-agent/pull/1688))
- **Mattermost** -- with @-mention-only channel filter ([#1683](https://github.com/NousResearch/hermes-agent/pull/1683), [#2443](https://github.com/NousResearch/hermes-agent/pull/2443))
- **Matrix** -- with vision support and image caching ([#1683](https://github.com/NousResearch/hermes-agent/pull/1683), [#2520](https://github.com/NousResearch/hermes-agent/pull/2520))
- **Webhook** -- platform adapter for external event triggers ([#2166](https://github.com/NousResearch/hermes-agent/pull/2166))
- **OpenAI-compatible API server** -- `/v1/chat/completions` endpoint with `/api/jobs` cron management, SQLite-backed response persistence, CORS origin protection, and input limits ([#1756](https://github.com/NousResearch/hermes-agent/pull/1756), [#2450](https://github.com/NousResearch/hermes-agent/pull/2450), [#2456](https://github.com/NousResearch/hermes-agent/pull/2456), [#2451](https://github.com/NousResearch/hermes-agent/pull/2451), [#2472](https://github.com/NousResearch/hermes-agent/pull/2472))

#### Gateway Core

- **Auto-reconnect** failed platforms with exponential backoff ([#2584](https://github.com/NousResearch/hermes-agent/pull/2584))
- **Notify users when session auto-resets** ([#2519](https://github.com/NousResearch/hermes-agent/pull/2519))
- **Reply-to message context** for out-of-session replies ([#1662](https://github.com/NousResearch/hermes-agent/pull/1662))
- Fix: `/reset` in thread-mode resets global session instead of thread ([#2254](https://github.com/NousResearch/hermes-agent/pull/2254))
- Fix: cap interrupt recursion depth to prevent resource exhaustion ([#1659](https://github.com/NousResearch/hermes-agent/pull/1659))
- Fix: prevent systemd restart storm on gateway connection failure ([#2327](https://github.com/NousResearch/hermes-agent/pull/2327))
- Fix: media delivery fails for file paths containing spaces ([#2621](https://github.com/NousResearch/hermes-agent/pull/2621))
- Fix: duplicate session-key collision in multi-platform gateway ([#2171](https://github.com/NousResearch/hermes-agent/pull/2171))
- Fix: PII redaction config never read -- missing yaml import ([#1701](https://github.com/NousResearch/hermes-agent/pull/1701))

#### Telegram Improvements

- MarkdownV2 support -- strikethrough, spoiler, blockquotes, escape parentheses/braces/backslashes/backticks ([#2199](https://github.com/NousResearch/hermes-agent/pull/2199), [#2200](https://github.com/NousResearch/hermes-agent/pull/2200) by @llbn, [#2386](https://github.com/NousResearch/hermes-agent/pull/2386))
- Telegram group vision support + thread-based sessions ([#2153](https://github.com/NousResearch/hermes-agent/pull/2153))
- Auto-reconnect polling after network interruption ([#2517](https://github.com/NousResearch/hermes-agent/pull/2517))

#### Discord Improvements

- Discord DM vision -- inline images + attachment analysis ([#2186](https://github.com/NousResearch/hermes-agent/pull/2186))
- Document caching and text-file injection ([#2503](https://github.com/NousResearch/hermes-agent/pull/2503))
- Fix: gateway crash on non-ASCII guild names ([#2302](https://github.com/NousResearch/hermes-agent/pull/2302))
- Fix: graceful WebSocket reconnection ([#2127](https://github.com/NousResearch/hermes-agent/pull/2127))

---

### CLI & User Experience

#### New Commands & Interactions

- **@ context completions** -- tab-completable `@file`/`@url` references that inject file content or web pages into the conversation ([#2482](https://github.com/NousResearch/hermes-agent/pull/2482), [#2343](https://github.com/NousResearch/hermes-agent/pull/2343))
- **`/statusbar`** -- toggle a persistent config bar showing model + provider info in the prompt ([#2240](https://github.com/NousResearch/hermes-agent/pull/2240), [#1917](https://github.com/NousResearch/hermes-agent/pull/1917))
- **`/queue`** -- queue prompts for the agent without interrupting the current run ([#2191](https://github.com/NousResearch/hermes-agent/pull/2191), [#2469](https://github.com/NousResearch/hermes-agent/pull/2469))
- **`/permission`** -- switch approval mode dynamically during a session ([#2207](https://github.com/NousResearch/hermes-agent/pull/2207))
- **`/browser`** -- interactive browser sessions from the CLI ([#2273](https://github.com/NousResearch/hermes-agent/pull/2273), [#1814](https://github.com/NousResearch/hermes-agent/pull/1814))
- **`/cost`** -- live pricing and usage tracking in gateway mode ([#2180](https://github.com/NousResearch/hermes-agent/pull/2180))
- **`/approve` and `/deny`** -- replaced bare text approval in gateway with explicit commands ([#2002](https://github.com/NousResearch/hermes-agent/pull/2002))

#### Streaming & Display

- **Streaming enabled by default** in CLI ([#2340](https://github.com/NousResearch/hermes-agent/pull/2340))
- Show spinners and tool progress during streaming mode ([#2161](https://github.com/NousResearch/hermes-agent/pull/2161))
- Show reasoning/thinking blocks when `show_reasoning` enabled ([#2118](https://github.com/NousResearch/hermes-agent/pull/2118))
- Context pressure warnings for CLI and gateway ([#2159](https://github.com/NousResearch/hermes-agent/pull/2159))
- Fix: streaming chunks concatenated without whitespace ([#2258](https://github.com/NousResearch/hermes-agent/pull/2258))
- Fix: suppress spinner animation in non-TTY environments ([#2216](https://github.com/NousResearch/hermes-agent/pull/2216))

#### Configuration

- **`${ENV_VAR}` substitution** in config.yaml ([#2684](https://github.com/NousResearch/hermes-agent/pull/2684))
- **Real-time config reload** -- config.yaml changes apply without restart ([#2210](https://github.com/NousResearch/hermes-agent/pull/2210))
- **Priority-based context file selection** + CLAUDE.md support ([#2301](https://github.com/NousResearch/hermes-agent/pull/2301))
- **Merge nested YAML sections** instead of replacing on config update ([#2213](https://github.com/NousResearch/hermes-agent/pull/2213))
- Fix: config.yaml provider key silently overrides env var ([#2272](https://github.com/NousResearch/hermes-agent/pull/2272))
- Fix: disabled toolsets re-enable themselves after `hermes tools` ([#2268](https://github.com/NousResearch/hermes-agent/pull/2268))
- Fix: add zprofile fallback and create zshrc on fresh macOS installs ([#2320](https://github.com/NousResearch/hermes-agent/pull/2320))

---

### Tool System

#### MCP Enhancements

- **MCP server management CLI** + OAuth 2.1 PKCE auth ([#2465](https://github.com/NousResearch/hermes-agent/pull/2465))
- **Expose MCP servers as standalone toolsets** ([#1907](https://github.com/NousResearch/hermes-agent/pull/1907))
- **Interactive MCP tool configuration** in `hermes tools` ([#1694](https://github.com/NousResearch/hermes-agent/pull/1694))
- Fix: MCP-OAuth port mismatch, path traversal, and shared handler state ([#2552](https://github.com/NousResearch/hermes-agent/pull/2552))
- Fix: preserve MCP tool registrations across session resets ([#2124](https://github.com/NousResearch/hermes-agent/pull/2124))

#### Web Tool Backends

- **Tavily** as web search/extract/crawl backend ([#1731](https://github.com/NousResearch/hermes-agent/pull/1731))
- **Parallel** as alternative web search/extract backend ([#1696](https://github.com/NousResearch/hermes-agent/pull/1696))
- **Configurable web backend** -- Firecrawl/BeautifulSoup/Playwright selection ([#2256](https://github.com/NousResearch/hermes-agent/pull/2256))

#### New Tools

- **IMAP email** reading and sending ([#2173](https://github.com/NousResearch/hermes-agent/pull/2173))
- **STT (speech-to-text)** tool using Whisper API ([#2072](https://github.com/NousResearch/hermes-agent/pull/2072))
- **Route-aware pricing estimates** ([#1695](https://github.com/NousResearch/hermes-agent/pull/1695))

---

### Plugin System Enhancements

- **TUI extension hooks** -- build custom CLIs on top of Hermes ([#2333](https://github.com/NousResearch/hermes-agent/pull/2333))
- **`hermes plugins install/remove/list`** commands ([#2337](https://github.com/NousResearch/hermes-agent/pull/2337))
- **Slash command registration** for plugins ([#2359](https://github.com/NousResearch/hermes-agent/pull/2359))
- **`session:end` lifecycle event** hook ([#1725](https://github.com/NousResearch/hermes-agent/pull/1725))

---

### Skills Ecosystem

**New skills:** OCR-and-documents (PDF/DOCX/XLS/PPTX/image OCR with optional GPU) ([#2236](https://github.com/NousResearch/hermes-agent/pull/2236)), Huggingface-hub bundled skill ([#1921](https://github.com/NousResearch/hermes-agent/pull/1921)), Sherlock OSINT username search ([#1671](https://github.com/NousResearch/hermes-agent/pull/1671)), Meme-generation with Pillow ([#2344](https://github.com/NousResearch/hermes-agent/pull/2344)), Bioinformatics gateway skill ([#2387](https://github.com/NousResearch/hermes-agent/pull/2387)), Base blockchain ([#1643](https://github.com/NousResearch/hermes-agent/pull/1643)), 3D-model-viewer ([#2226](https://github.com/NousResearch/hermes-agent/pull/2226)), FastMCP ([#2113](https://github.com/NousResearch/hermes-agent/pull/2113)), Inference.sh ([#1686](https://github.com/NousResearch/hermes-agent/pull/1686)), Hermes-agent-setup ([#1905](https://github.com/NousResearch/hermes-agent/pull/1905)).

**Skills system improvements:**
- Agent-created skills -- caution-level findings allowed, dangerous skills ask instead of block ([#1840](https://github.com/NousResearch/hermes-agent/pull/1840), [#2446](https://github.com/NousResearch/hermes-agent/pull/2446))
- `--yes` flag to bypass confirmation in `/skills install` and uninstall ([#1647](https://github.com/NousResearch/hermes-agent/pull/1647))
- Fix: agent-created skills keep working after session reset ([#2121](https://github.com/NousResearch/hermes-agent/pull/2121))
- Fix: skills hub inspect/resolve -- 4 bugs in inspect, redirects, discovery, tap list ([#2447](https://github.com/NousResearch/hermes-agent/pull/2447))

---

### Security & Reliability

#### Security

- SSRF protection for `vision_tools` and `web_tools` ([#2679](https://github.com/NousResearch/hermes-agent/pull/2679))
- Shell injection prevention in `_expand_path` via `~user` path suffix ([#2685](https://github.com/NousResearch/hermes-agent/pull/2685))
- Block untrusted browser-origin API server access ([#2451](https://github.com/NousResearch/hermes-agent/pull/2451))
- Block sandbox backend credentials from subprocess env ([#1658](https://github.com/NousResearch/hermes-agent/pull/1658))
- Block @ references from reading secrets outside workspace ([#2601](https://github.com/NousResearch/hermes-agent/pull/2601) by @Gutslabs)
- Malicious code pattern pre-exec scanner for `terminal_tool` ([#2245](https://github.com/NousResearch/hermes-agent/pull/2245))
- Eliminate SQL string formatting in `execute()` calls ([#2061](https://github.com/NousResearch/hermes-agent/pull/2061) by @dusterbloom)
- Harden jobs API -- input limits, field whitelist, startup check ([#2456](https://github.com/NousResearch/hermes-agent/pull/2456))

#### Reliability

- Thread locks on 4 SessionDB methods ([#1704](https://github.com/NousResearch/hermes-agent/pull/1704))
- File locking for concurrent memory writes ([#1726](https://github.com/NousResearch/hermes-agent/pull/1726))
- API server: persist ResponseStore to SQLite across restarts ([#2472](https://github.com/NousResearch/hermes-agent/pull/2472))

#### Cron System

- `[SILENT]` response -- cron agents can suppress delivery ([#1833](https://github.com/NousResearch/hermes-agent/pull/1833))
- Scale missed-job grace window with schedule frequency ([#2449](https://github.com/NousResearch/hermes-agent/pull/2449))
- Fix: normalize `repeat<=0` to None -- jobs deleted after first run when LLM passes -1 ([#2612](https://github.com/NousResearch/hermes-agent/pull/2612) by @Mibayy)
- Fix: naive ISO timestamps without timezone -- jobs fire at wrong time ([#1729](https://github.com/NousResearch/hermes-agent/pull/1729))
- Fix: stop injecting cron outputs into gateway session history ([#2313](https://github.com/NousResearch/hermes-agent/pull/2313))

---

### Contributors

**Core:** @teknium1 -- 280 PRs.

**Community contributors:** @mchzimm (GitHub Copilot provider), @jquesnelle (per-thread persistent event loops), @llbn (Telegram MarkdownV2), @dusterbloom (SQL injection prevention, local server context window querying), @0xbyt4 (tool_calls None guard, OpenCode-Go config fix), @sai-samartha (WhatsApp send_message routing, systemd node path), @Gutslabs (block @ references from reading secrets), @Mibayy (cron repeat normalization), @ten-jampa (gateway /title fix), @cutepawss (file tools search pagination), @hanai (OpenAI TTS base_url), @rovle (Daytona sandbox API migration), @buntingszn (Matrix cron delivery), @InB4DevOps (token counter reset), @JiwaniZakir (missing file in wheel fix), @ygd58 (delegate tool fix).

**Full changelog:** [v2026.3.17...v2026.3.23](https://github.com/NousResearch/hermes-agent/compare/v2026.3.17...v2026.3.23)

---

## v0.3.0 (v2026.3.17) -- March 17, 2026

> The streaming, plugins, and provider release -- unified real-time token delivery, first-class plugin architecture, rebuilt provider system with Vercel AI Gateway, native Anthropic provider, smart approvals, live Chrome CDP browser connect, ACP IDE integration, Honcho memory, voice mode, persistent shell, and 50+ bug fixes across every platform.

---

### Core Agent & Architecture

#### Unified Streaming Infrastructure

Real-time token-by-token delivery in CLI and all gateway platforms. Responses stream as they are generated instead of arriving as a single block. ([#1538](https://github.com/NousResearch/hermes-agent/pull/1538))

- `AIAgent.__init__` accepts `stream_delta_callback: callable = None` (persistent callback for display)
- `run_conversation()` accepts `stream_callback: callable = None` (per-call callback for TTS)
- `_interruptible_streaming_api_call()` handles all three API modes with graceful fallback to non-streaming on error
- `_fire_stream_delta()` fires both `stream_delta_callback` and `_stream_callback` for each text token
- Thread-safe `queue.Queue()` bridges the agent thread and each consumer (CLI, gateway, API server)
- End-of-stream signaled via `_fire_stream_delta(None)`; consumers remove cursors and finalize display
- Per-platform message-edit throttling respects API rate limits (Telegram ~1.5s, Discord ~1.2s, Slack ~1.5s)
- `min_tokens: 20` threshold prevents premature partial previews
- WhatsApp, Signal, Email, and Home Assistant fall back to non-streaming automatically (no edit API)
- `HERMES_STREAMING_ENABLED=true` env var and `streaming.*` config section; master switch defaults to off
- See [streaming.md](./streaming.md) for full documentation

#### Concurrent Tool Execution

Multiple independent tool calls now run in parallel via `ThreadPoolExecutor` (max 8 workers), significantly reducing latency for multi-tool turns. Parallelization decided by `_should_parallelize_tool_batch()` checking tool safety classification. Message and result ordering is preserved when reinserting tool responses into conversation history. ([#1152](https://github.com/NousResearch/hermes-agent/pull/1152))

#### Plugin Architecture

Drop Python files into `~/.hermes/plugins/` to extend Hermes with custom tools, commands, and hooks. No forking or patching required. ([#1544](https://github.com/NousResearch/hermes-agent/pull/1544), [#1555](https://github.com/NousResearch/hermes-agent/pull/1555))

#### Persistent Shell Mode

Local and SSH terminal backends can maintain shell state across tool calls -- `cd`, environment variables, and shell aliases persist within a session. Contributed by @alt-glitch. ([#1067](https://github.com/NousResearch/hermes-agent/pull/1067), [#1483](https://github.com/NousResearch/hermes-agent/pull/1483))

#### Context Compression Improvements

- Session hygiene threshold tuned to 50% for more proactive compression ([#1096](https://github.com/NousResearch/hermes-agent/pull/1096), [#1161](https://github.com/NousResearch/hermes-agent/pull/1161))
- Improved compaction handoff summaries -- compressor preserves more actionable state ([#1273](https://github.com/NousResearch/hermes-agent/pull/1273))
- Session ID synced after mid-run context compression ([#1160](https://github.com/NousResearch/hermes-agent/pull/1160))
- Anthropic Context Editing API support ([#1147](https://github.com/NousResearch/hermes-agent/pull/1147))
- `--pass-session-id` flag includes session ID in system prompt ([#1040](https://github.com/NousResearch/hermes-agent/pull/1040))

---

### Provider & Model Support

#### Native Anthropic Provider

Direct Anthropic API calls with Claude Code credential auto-discovery, OAuth PKCE flows, and native prompt caching. No OpenRouter middleman required. Auth supports regular API keys (`sk-ant-api*`), OAuth setup-tokens (`sk-ant-oat*`), and Claude Code credentials. ([#1097](https://github.com/NousResearch/hermes-agent/pull/1097))

- Anthropic OAuth: auto-run `claude setup-token`, reauthentication, PKCE state persistence, identity fingerprinting ([#1132](https://github.com/NousResearch/hermes-agent/pull/1132), [#1360](https://github.com/NousResearch/hermes-agent/pull/1360), [#1396](https://github.com/NousResearch/hermes-agent/pull/1396), [#1597](https://github.com/NousResearch/hermes-agent/pull/1597))
- Native Anthropic auxiliary vision -- use Claude's native vision API directly ([#1377](https://github.com/NousResearch/hermes-agent/pull/1377))
- Retry Anthropic 429/529 errors and surface details to users -- by @0xbyt4 ([#1585](https://github.com/NousResearch/hermes-agent/pull/1585))
- Fix Anthropic adapter max_tokens, fallback crash, proxy base_url -- by @0xbyt4 ([#1121](https://github.com/NousResearch/hermes-agent/pull/1121))
- Fix Anthropic cache markers through adapter -- by @brandtcormorant ([#1216](https://github.com/NousResearch/hermes-agent/pull/1216))
- Fix adaptive thinking without `budget_tokens` for Claude 4.6 models -- by @ASRagab ([#1128](https://github.com/NousResearch/hermes-agent/pull/1128))

#### Centralized Provider Router (Rebuilt)

Rebuilt provider system with `call_llm` API, unified `/model` command, auto-detect provider on model switch, and direct endpoint overrides for auxiliary and delegation clients. ([#1003](https://github.com/NousResearch/hermes-agent/pull/1003), [#1506](https://github.com/NousResearch/hermes-agent/pull/1506), [#1375](https://github.com/NousResearch/hermes-agent/pull/1375))

#### Vercel AI Gateway

Route Hermes through Vercel's AI Gateway (`https://ai-gateway.vercel.sh/v1`) for access to their model catalog and infrastructure. ([#1628](https://github.com/NousResearch/hermes-agent/pull/1628))

#### Other Provider Improvements

- Auto-detect provider when switching models via `/model` ([#1506](https://github.com/NousResearch/hermes-agent/pull/1506))
- Direct endpoint overrides for auxiliary and delegation clients ([#1375](https://github.com/NousResearch/hermes-agent/pull/1375))
- Fix DeepSeek V3 parser dropping multiple parallel tool calls -- by @mr-emmett-one ([#1365](https://github.com/NousResearch/hermes-agent/pull/1365), [#1300](https://github.com/NousResearch/hermes-agent/pull/1300))
- Accept unlisted models with warning instead of rejecting ([#1047](https://github.com/NousResearch/hermes-agent/pull/1047), [#1102](https://github.com/NousResearch/hermes-agent/pull/1102))
- Skip reasoning params for unsupported OpenRouter models ([#1485](https://github.com/NousResearch/hermes-agent/pull/1485))
- MiniMax Anthropic API compatibility fix ([#1623](https://github.com/NousResearch/hermes-agent/pull/1623))
- Custom endpoint `/models` verification and `/v1` base URL suggestion ([#1480](https://github.com/NousResearch/hermes-agent/pull/1480))
- Resolve delegation providers from `custom_providers` config ([#1328](https://github.com/NousResearch/hermes-agent/pull/1328))
- Kimi model additions and User-Agent fix ([#1039](https://github.com/NousResearch/hermes-agent/pull/1039))
- Strip `call_id`/`response_item_id` for Mistral compatibility ([#1058](https://github.com/NousResearch/hermes-agent/pull/1058))

---

### Memory & Sessions

#### Honcho Memory Integration

Async memory writes, configurable recall modes, session title integration, and multi-user isolation in gateway mode. Contributed by @erosika. ([#736](https://github.com/NousResearch/hermes-agent/pull/736))

- Keep Honcho recall out of the cached system prefix -- injected via `_inject_honcho_turn_context()` into the current-turn user message ([#1201](https://github.com/NousResearch/hermes-agent/pull/1201))
- Isolate Honcho session routing for multi-user gateway ([#1500](https://github.com/NousResearch/hermes-agent/pull/1500))
- Improve memory prioritization -- user preferences and corrections weighted above procedural knowledge ([#1548](https://github.com/NousResearch/hermes-agent/pull/1548))
- Tighter memory and session recall guidance in system prompts ([#1329](https://github.com/NousResearch/hermes-agent/pull/1329))
- Persist CLI token counts to session DB for `/insights` ([#1498](https://github.com/NousResearch/hermes-agent/pull/1498))
- Correct `seed_ai_identity` to use `session.add_messages()` ([#1475](https://github.com/NousResearch/hermes-agent/pull/1475))

---

### Messaging Gateway

#### Gateway Core

- System gateway service mode -- run as a system-level systemd service ([#1371](https://github.com/NousResearch/hermes-agent/pull/1371))
- Gateway install scope prompts -- choose user vs system scope during setup ([#1374](https://github.com/NousResearch/hermes-agent/pull/1374))
- Reasoning hot reload -- change reasoning settings without restarting ([#1275](https://github.com/NousResearch/hermes-agent/pull/1275))
- Default group sessions to per-user isolation ([#1495](https://github.com/NousResearch/hermes-agent/pull/1495), [#1417](https://github.com/NousResearch/hermes-agent/pull/1417))
- Harden gateway restart recovery ([#1310](https://github.com/NousResearch/hermes-agent/pull/1310))
- Cancel active runs during shutdown ([#1427](https://github.com/NousResearch/hermes-agent/pull/1427))
- SSL certificate auto-detection for NixOS and non-standard systems ([#1494](https://github.com/NousResearch/hermes-agent/pull/1494))
- Auto-detect D-Bus session bus for `systemctl --user` on headless servers ([#1601](https://github.com/NousResearch/hermes-agent/pull/1601))
- Auto-enable systemd linger during gateway install on headless servers ([#1334](https://github.com/NousResearch/hermes-agent/pull/1334))
- Fix dual gateways on macOS launchd after `hermes update` ([#1567](https://github.com/NousResearch/hermes-agent/pull/1567))
- Remove recursive ExecStop from systemd units ([#1530](https://github.com/NousResearch/hermes-agent/pull/1530))
- Prevent logging handler accumulation in gateway mode ([#1251](https://github.com/NousResearch/hermes-agent/pull/1251))
- Restart on retryable startup failures -- by @jplew ([#1517](https://github.com/NousResearch/hermes-agent/pull/1517))
- Backfill model on gateway sessions after agent runs ([#1306](https://github.com/NousResearch/hermes-agent/pull/1306))
- PID-based gateway kill and deferred config write ([#1499](https://github.com/NousResearch/hermes-agent/pull/1499))
- Fall back to module entrypoint when `hermes` is not on PATH ([#1355](https://github.com/NousResearch/hermes-agent/pull/1355))

#### Platform-Specific

**Telegram:**
- Buffer media groups to prevent self-interruption from photo bursts ([#1341](https://github.com/NousResearch/hermes-agent/pull/1341), [#1422](https://github.com/NousResearch/hermes-agent/pull/1422))
- Retry on transient TLS failures ([#1535](https://github.com/NousResearch/hermes-agent/pull/1535))
- Harden polling conflict handling ([#1339](https://github.com/NousResearch/hermes-agent/pull/1339))
- Escape chunk indicators and inline code in MarkdownV2 ([#1478](https://github.com/NousResearch/hermes-agent/pull/1478), [#1626](https://github.com/NousResearch/hermes-agent/pull/1626))
- Check updater/app state before disconnect ([#1389](https://github.com/NousResearch/hermes-agent/pull/1389))

**Discord:**
- `/thread` command with `auto_thread` config and media metadata fixes ([#1178](https://github.com/NousResearch/hermes-agent/pull/1178))
- Auto-thread on @mention, skip mention text in bot threads ([#1438](https://github.com/NousResearch/hermes-agent/pull/1438))
- Retry without reply reference for system messages ([#1385](https://github.com/NousResearch/hermes-agent/pull/1385))
- Preserve native document and video attachment support ([#1392](https://github.com/NousResearch/hermes-agent/pull/1392))
- Defer discord adapter annotations to avoid optional import crashes ([#1314](https://github.com/NousResearch/hermes-agent/pull/1314))

**Slack:**
- Thread handling overhaul -- progress messages, responses, and session isolation all respect threads ([#1103](https://github.com/NousResearch/hermes-agent/pull/1103))
- Formatting, reactions, user resolution, and command improvements ([#1106](https://github.com/NousResearch/hermes-agent/pull/1106))
- Fix MAX_MESSAGE_LENGTH 3900 -> 39000 ([#1117](https://github.com/NousResearch/hermes-agent/pull/1117))
- File upload fallback preserves thread context -- by @0xbyt4 ([#1122](https://github.com/NousResearch/hermes-agent/pull/1122))
- Improve setup guidance ([#1387](https://github.com/NousResearch/hermes-agent/pull/1387))

**Email:**
- Fix IMAP UID tracking and SMTP TLS verification ([#1305](https://github.com/NousResearch/hermes-agent/pull/1305))
- Add `skip_attachments` option via config.yaml ([#1536](https://github.com/NousResearch/hermes-agent/pull/1536))

**Home Assistant:**
- Event filtering closed by default ([#1169](https://github.com/NousResearch/hermes-agent/pull/1169))

---

### CLI & User Experience

- **Smart Approvals + `/stop` command** -- Codex-inspired approval system that learns which commands are safe and remembers preferences. `/stop` kills the current agent run immediately. ([#1543](https://github.com/NousResearch/hermes-agent/pull/1543))
- **Persistent CLI status bar** -- always-visible model, provider, and token counts ([#1522](https://github.com/NousResearch/hermes-agent/pull/1522))
- **File path autocomplete** in the input prompt ([#1545](https://github.com/NousResearch/hermes-agent/pull/1545))
- **`/plan` command** -- generate implementation plans from specs ([#1372](https://github.com/NousResearch/hermes-agent/pull/1372), [#1381](https://github.com/NousResearch/hermes-agent/pull/1381))
- **Major `/rollback` improvements** -- richer checkpoint history, clearer UX ([#1505](https://github.com/NousResearch/hermes-agent/pull/1505))
- **Preload CLI skills on launch** -- skills are ready before the first prompt ([#1359](https://github.com/NousResearch/hermes-agent/pull/1359))
- **Centralized slash command registry** -- all commands defined once, consumed everywhere ([#1603](https://github.com/NousResearch/hermes-agent/pull/1603))
- `/bg` alias for `/background` ([#1590](https://github.com/NousResearch/hermes-agent/pull/1590))
- Prefix matching for slash commands -- `/mod` resolves to `/model` ([#1320](https://github.com/NousResearch/hermes-agent/pull/1320))
- `/new`, `/reset`, `/clear` now start genuinely fresh sessions ([#1237](https://github.com/NousResearch/hermes-agent/pull/1237))
- Accept session ID prefixes for session actions ([#1425](https://github.com/NousResearch/hermes-agent/pull/1425))
- TUI prompt and accent output now respect active skin ([#1282](https://github.com/NousResearch/hermes-agent/pull/1282))
- Centralize tool emoji metadata in registry + skin integration ([#1484](https://github.com/NousResearch/hermes-agent/pull/1484))
- "View full command" option in dangerous command approval -- by @teknium1 ([#887](https://github.com/NousResearch/hermes-agent/pull/887))
- Non-blocking startup update check and banner deduplication ([#1386](https://github.com/NousResearch/hermes-agent/pull/1386))
- Verbose mode shows full untruncated output ([#1472](https://github.com/NousResearch/hermes-agent/pull/1472))
- Fix `/status` to report live state and tokens ([#1476](https://github.com/NousResearch/hermes-agent/pull/1476))
- Seed a default global SOUL.md ([#1311](https://github.com/NousResearch/hermes-agent/pull/1311))
- OpenClaw migration during first-time setup -- by @kshitijk4poor ([#981](https://github.com/NousResearch/hermes-agent/pull/981))
- `hermes claw migrate` command + migration docs ([#1059](https://github.com/NousResearch/hermes-agent/pull/1059))

---

### Tool System

#### Voice Mode

Push-to-talk in CLI, voice notes in Telegram and Discord, Discord voice channel support, and local Whisper transcription via faster-whisper. ([#1299](https://github.com/NousResearch/hermes-agent/pull/1299), [#1185](https://github.com/NousResearch/hermes-agent/pull/1185), [#1429](https://github.com/NousResearch/hermes-agent/pull/1429))

- Free local Whisper transcription via faster-whisper ([#1185](https://github.com/NousResearch/hermes-agent/pull/1185))
- Discord voice channel reliability fixes ([#1429](https://github.com/NousResearch/hermes-agent/pull/1429))
- Restore local STT fallback for gateway voice notes ([#1490](https://github.com/NousResearch/hermes-agent/pull/1490))
- Honor `stt.enabled: false` across gateway transcription ([#1394](https://github.com/NousResearch/hermes-agent/pull/1394))

#### Browser

- **`/browser connect` via CDP** -- attach browser tools to a live Chrome instance through Chrome DevTools Protocol ([#1549](https://github.com/NousResearch/hermes-agent/pull/1549))
- Improve browser cleanup, local browser PATH setup, and screenshot recovery ([#1333](https://github.com/NousResearch/hermes-agent/pull/1333))

#### MCP

- Selective tool loading with utility policies -- filter which MCP tools are available ([#1302](https://github.com/NousResearch/hermes-agent/pull/1302))
- Auto-reload MCP tools when `mcp_servers` config changes without restart ([#1474](https://github.com/NousResearch/hermes-agent/pull/1474))
- Resolve npx stdio connection failures ([#1291](https://github.com/NousResearch/hermes-agent/pull/1291))
- Preserve MCP toolsets when saving platform tool config ([#1421](https://github.com/NousResearch/hermes-agent/pull/1421))

#### Terminal & Execution

- **Persistent shell mode** for local and SSH backends -- maintain shell state across tool calls -- by @alt-glitch ([#1067](https://github.com/NousResearch/hermes-agent/pull/1067), [#1483](https://github.com/NousResearch/hermes-agent/pull/1483))
- **Tirith pre-exec command scanning** -- security layer that analyzes commands before execution ([#1256](https://github.com/NousResearch/hermes-agent/pull/1256))
- Strip Hermes provider env vars from all subprocess environments ([#1157](https://github.com/NousResearch/hermes-agent/pull/1157), [#1172](https://github.com/NousResearch/hermes-agent/pull/1172), [#1399](https://github.com/NousResearch/hermes-agent/pull/1399), [#1419](https://github.com/NousResearch/hermes-agent/pull/1419)) -- initial fix by @eren-karakus0
- SSH preflight check ([#1486](https://github.com/NousResearch/hermes-agent/pull/1486))
- Docker backend: make cwd workspace mount explicit opt-in ([#1534](https://github.com/NousResearch/hermes-agent/pull/1534))
- Add project root to PYTHONPATH in execute_code sandbox ([#1383](https://github.com/NousResearch/hermes-agent/pull/1383))
- Eliminate execute_code progress spam on gateway platforms ([#1098](https://github.com/NousResearch/hermes-agent/pull/1098))

#### Cron

- Compress cron management into one `cronjob` tool replacing multiple commands ([#1343](https://github.com/NousResearch/hermes-agent/pull/1343))
- Suppress duplicate cron sends to auto-delivery targets ([#1357](https://github.com/NousResearch/hermes-agent/pull/1357))
- Persist cron sessions to SQLite ([#1255](https://github.com/NousResearch/hermes-agent/pull/1255))
- Per-job runtime overrides (provider, model, base_url) ([#1398](https://github.com/NousResearch/hermes-agent/pull/1398))
- Atomic write in `save_job_output` to prevent data loss on crash ([#1173](https://github.com/NousResearch/hermes-agent/pull/1173))
- Preserve thread context for `deliver=origin` ([#1437](https://github.com/NousResearch/hermes-agent/pull/1437))

#### Patch Tool

- Avoid corrupting pipe chars in V4A patch apply ([#1286](https://github.com/NousResearch/hermes-agent/pull/1286))
- Permissive `block_anchor` thresholds and unicode normalization ([#1539](https://github.com/NousResearch/hermes-agent/pull/1539))

#### Vision

- Unify vision backend gating ([#1367](https://github.com/NousResearch/hermes-agent/pull/1367))
- Surface actual error reason instead of generic message ([#1338](https://github.com/NousResearch/hermes-agent/pull/1338))
- Make Claude image handling work end-to-end ([#1408](https://github.com/NousResearch/hermes-agent/pull/1408))

#### Delegation

- Add observability metadata to subagent results (model, tokens, duration, tool trace) ([#1175](https://github.com/NousResearch/hermes-agent/pull/1175))

---

### ACP (IDE Integration)

VS Code, Zed, and JetBrains can now connect to Hermes as an agent backend, with full slash command support. The `hermes-acp` entry point runs the ACP server. ([#1254](https://github.com/NousResearch/hermes-agent/pull/1254), [#1532](https://github.com/NousResearch/hermes-agent/pull/1532))

---

### PII Redaction

When `privacy.redact_pii` is enabled, personally identifiable information is automatically scrubbed before sending context to LLM providers. ([#1542](https://github.com/NousResearch/hermes-agent/pull/1542))

---

### Skills Ecosystem

**New skills:**
- Linear project management ([#1230](https://github.com/NousResearch/hermes-agent/pull/1230))
- X/Twitter via x-cli ([#1285](https://github.com/NousResearch/hermes-agent/pull/1285))
- Telephony -- Twilio, SMS, and AI calls ([#1289](https://github.com/NousResearch/hermes-agent/pull/1289))
- 1Password -- by @arceus77-7 ([#883](https://github.com/NousResearch/hermes-agent/pull/883), [#1179](https://github.com/NousResearch/hermes-agent/pull/1179))
- NeuroSkill BCI integration ([#1135](https://github.com/NousResearch/hermes-agent/pull/1135))
- Blender MCP for 3D modeling ([#1531](https://github.com/NousResearch/hermes-agent/pull/1531))
- OSS Security Forensics ([#1482](https://github.com/NousResearch/hermes-agent/pull/1482))
- Parallel CLI research skill ([#1301](https://github.com/NousResearch/hermes-agent/pull/1301))
- OpenCode CLI skill ([#1174](https://github.com/NousResearch/hermes-agent/pull/1174))
- ASCII Video skill refactored -- by @SHL0MS ([#1213](https://github.com/NousResearch/hermes-agent/pull/1213), [#1598](https://github.com/NousResearch/hermes-agent/pull/1598))

**Skills system improvements:**
- Integrate skills.sh as a hub source alongside ClawHub ([#1303](https://github.com/NousResearch/hermes-agent/pull/1303))
- Secure skill env setup on load ([#1153](https://github.com/NousResearch/hermes-agent/pull/1153))
- Honor policy table for dangerous verdicts ([#1330](https://github.com/NousResearch/hermes-agent/pull/1330))
- Harden ClawHub skill search exact matches ([#1400](https://github.com/NousResearch/hermes-agent/pull/1400))
- Fix ClawHub skill install -- use `/download` ZIP endpoint ([#1060](https://github.com/NousResearch/hermes-agent/pull/1060))
- Avoid mislabeling local skills as builtin -- by @arceus77-7 ([#862](https://github.com/NousResearch/hermes-agent/pull/862))

---

### Security & Reliability

- **Tirith pre-exec command scanning** -- static analysis of terminal commands before execution ([#1256](https://github.com/NousResearch/hermes-agent/pull/1256))
- **PII redaction** when `privacy.redact_pii` is enabled ([#1542](https://github.com/NousResearch/hermes-agent/pull/1542))
- Strip Hermes provider/gateway/tool env vars from all subprocess environments ([#1157](https://github.com/NousResearch/hermes-agent/pull/1157), [#1172](https://github.com/NousResearch/hermes-agent/pull/1172), [#1399](https://github.com/NousResearch/hermes-agent/pull/1399), [#1419](https://github.com/NousResearch/hermes-agent/pull/1419))
- Docker cwd workspace mount now explicit opt-in -- never auto-mount host directories ([#1534](https://github.com/NousResearch/hermes-agent/pull/1534))
- Escape parens and braces in fork bomb regex pattern ([#1397](https://github.com/NousResearch/hermes-agent/pull/1397))
- Harden `.worktreeinclude` path containment ([#1388](https://github.com/NousResearch/hermes-agent/pull/1388))
- Use description as `pattern_key` to prevent approval collisions ([#1395](https://github.com/NousResearch/hermes-agent/pull/1395))
- Guard init-time stdio writes ([#1271](https://github.com/NousResearch/hermes-agent/pull/1271))
- Session log writes reuse shared atomic JSON helper ([#1280](https://github.com/NousResearch/hermes-agent/pull/1280))
- Atomic temp cleanup protected on interrupts ([#1401](https://github.com/NousResearch/hermes-agent/pull/1401))

---

### RL Training

- **Agentic On-Policy Distillation (OPD)** environment -- new RL training environment for agent policy distillation ([#1149](https://github.com/NousResearch/hermes-agent/pull/1149))
- Make tinker-atropos RL training fully optional ([#1062](https://github.com/NousResearch/hermes-agent/pull/1062))

---

### Testing

- Cover empty cached Anthropic tool-call turns ([#1222](https://github.com/NousResearch/hermes-agent/pull/1222))
- Fix stale CI assumptions in parser and quick-command coverage ([#1236](https://github.com/NousResearch/hermes-agent/pull/1236))
- Fix gateway async tests without implicit event loop ([#1278](https://github.com/NousResearch/hermes-agent/pull/1278))
- Make gateway async tests xdist-safe ([#1281](https://github.com/NousResearch/hermes-agent/pull/1281))
- Cross-timezone naive timestamp regression for cron ([#1319](https://github.com/NousResearch/hermes-agent/pull/1319))
- Isolate codex provider tests from local env ([#1335](https://github.com/NousResearch/hermes-agent/pull/1335))
- Lock retry replacement semantics ([#1379](https://github.com/NousResearch/hermes-agent/pull/1379))
- Improve error logging in session search tool -- by @aydnOktay ([#1533](https://github.com/NousResearch/hermes-agent/pull/1533))

---

### Notable Bug Fixes

- `/status` always showing 0 tokens -- now reports live state ([#1476](https://github.com/NousResearch/hermes-agent/pull/1476))
- Custom model endpoints not working -- restored config-saved endpoint resolution ([#1373](https://github.com/NousResearch/hermes-agent/pull/1373))
- MCP tools not visible until restart -- auto-reload on config change ([#1474](https://github.com/NousResearch/hermes-agent/pull/1474))
- `hermes tools` removing MCP tools -- preserve MCP toolsets when saving ([#1421](https://github.com/NousResearch/hermes-agent/pull/1421))
- Terminal subprocesses inheriting `OPENAI_BASE_URL` breaking external tools ([#1399](https://github.com/NousResearch/hermes-agent/pull/1399))
- Background process lost on gateway restart -- improved recovery
- Cron jobs not persisting state -- now stored in SQLite ([#1255](https://github.com/NousResearch/hermes-agent/pull/1255))
- Cronjob `deliver: origin` not preserving thread context ([#1437](https://github.com/NousResearch/hermes-agent/pull/1437))
- Gateway systemd service failing to auto-restart when browser processes orphaned
- Remaining hardcoded `~/.hermes` paths -- all now respect `HERMES_HOME` ([#1233](https://github.com/NousResearch/hermes-agent/pull/1233))
- Model switching not taking effect ([#1183](https://github.com/NousResearch/hermes-agent/pull/1183))
- `hermes doctor` reporting cronjob as unavailable ([#1180](https://github.com/NousResearch/hermes-agent/pull/1180))
- Setup wizard hanging on headless SSH ([#1274](https://github.com/NousResearch/hermes-agent/pull/1274))
- Log handler accumulation degrading gateway performance ([#1251](https://github.com/NousResearch/hermes-agent/pull/1251))
- Gateway NULL model in DB ([#1306](https://github.com/NousResearch/hermes-agent/pull/1306))
- Delegate tool not working with custom inference providers ([#1328](https://github.com/NousResearch/hermes-agent/pull/1328))
- Skills Guard blocking official skills ([#1330](https://github.com/NousResearch/hermes-agent/pull/1330))
- `GatewayConfig.get()` AttributeError crashing all message handling ([#1287](https://github.com/NousResearch/hermes-agent/pull/1287))
- Image analysis failing silently ([#1338](https://github.com/NousResearch/hermes-agent/pull/1338))
- Slash commands requiring exact full name -- now uses prefix matching ([#1320](https://github.com/NousResearch/hermes-agent/pull/1320))

---

### Contributors

**Core:** @teknium1 -- 220+ PRs spanning every area of the codebase.

**Top community contributors:** @0xbyt4 (Anthropic adapter fixes, Slack, setup), @erosika (Honcho memory), @SHL0MS (ASCII video), @alt-glitch (persistent shell, packaging), @arceus77-7 (1Password skill), @kshitijk4poor (OpenClaw migration), @ASRagab (Claude 4.6 adaptive thinking), @eren-karakus0 (subprocess env isolation), @mr-emmett-one (DeepSeek V3 parser), @jplew (gateway restart), @brandtcormorant (Anthropic cache control), @aydnOktay (session search logging), @austinpickett (landing page).

Full list: @0xbyt4, @alt-glitch, @arceus77-7, @ASRagab, @austinpickett, @aydnOktay, @brandtcormorant, @eren-karakus0, @erosika, @JackTheGit, @jplew, @kshitijk4poor, @mr-emmett-one, @SHL0MS, @teknium1.

**Full changelog:** [v2026.3.12...v2026.3.17](https://github.com/NousResearch/hermes-agent/compare/v2026.3.12...v2026.3.17)

---

## v0.2.0 (v2026.3.12) -- March 12, 2026

> First tagged release since v0.1.0. In just over two weeks, Hermes Agent went from a small internal project to a full-featured AI agent platform. This release covers **216 merged pull requests** from **63 contributors**, resolving **119 issues**.

---

### Core Agent & Architecture

#### Centralized Provider Router

Unified `call_llm()` / `async_call_llm()` API replaces scattered provider logic across vision, summarization, compression, and trajectory saving. All auxiliary consumers route through `resolve_provider_client()` with automatic credential resolution. ([#1003](https://github.com/NousResearch/hermes-agent/pull/1003))

- Per-consumer auxiliary model configuration
- Fallback model for provider resilience ([#740](https://github.com/NousResearch/hermes-agent/pull/740))
- Live API validation for model switching (against live API, not hardcoded lists)

#### Agent Loop & Conversation

- Shared `IterationBudget` across parent and subagent delegation -- prevents runaway recursion
- Iteration budget pressure via tool result injection (70% caution, 90% warning thresholds)
- Configurable subagent provider/model with full credential resolution
- Handle 413 payload-too-large via compression instead of aborting ([#153](https://github.com/NousResearch/hermes-agent/pull/153))
- Retry with rebuilt payload after compression ([#616](https://github.com/NousResearch/hermes-agent/pull/616))
- Auto-compress pathologically large gateway sessions ([#628](https://github.com/NousResearch/hermes-agent/issues/628))
- Tool call repair middleware -- auto-lowercase and invalid tool handler
- Detect and block file re-read/search loops after context compression ([#705](https://github.com/NousResearch/hermes-agent/pull/705))
- Reasoning effort configuration and `/reasoning` command ([#921](https://github.com/NousResearch/hermes-agent/pull/921))

#### Session & Memory

- Session naming with unique titles, auto-lineage, rich listing, and resume by name ([#720](https://github.com/NousResearch/hermes-agent/pull/720))
- Interactive session browser with search filtering ([#733](https://github.com/NousResearch/hermes-agent/pull/733))
- Honcho AI-native cross-session user modeling -- by @erosika ([#38](https://github.com/NousResearch/hermes-agent/pull/38))
- Proactive async memory flush on session expiry
- Smart context length probing with persistent caching and banner display
- `/resume` command for switching to named sessions in gateway
- Session reset policy for messaging platforms (daily/idle/both/none)

---

### Provider & Model Support

- Nous Portal as first-class provider
- OpenAI Codex (Responses API) with ChatGPT subscription support -- by @grp06 ([#43](https://github.com/NousResearch/hermes-agent/pull/43))
- Codex OAuth vision support and multimodal content adapter
- Self-hosted Firecrawl support -- by @caentzminger ([#460](https://github.com/NousResearch/hermes-agent/pull/460))
- Kimi Code API support -- by @christomitov ([#635](https://github.com/NousResearch/hermes-agent/pull/635))
- z.ai/GLM, Kimi/Moonshot, MiniMax, Azure OpenAI as first-class providers
- OpenRouter provider routing configuration (`provider_preferences`)
- Nous credential refresh on 401 errors ([#571](https://github.com/NousResearch/hermes-agent/pull/571))
- Unified `/model` and `/provider` into single view

---

### Messaging Gateway (7 Platforms)

#### Multi-Platform Messaging Gateway

Telegram, Discord, Slack, WhatsApp, Signal, Email (IMAP/SMTP), and Home Assistant platforms with unified session management, media attachments, and per-platform tool configuration.

**Telegram:**
- Native file attachments: `send_document` + `send_video`
- Document file processing for PDF, text, and Office files
- Forum topic session isolation ([#766](https://github.com/NousResearch/hermes-agent/pull/766))
- Browser screenshot sharing via MEDIA: protocol ([#657](https://github.com/NousResearch/hermes-agent/pull/657))
- Location support for find-nearby skill
- Italic regex newline fix + 43 format tests -- by @0xbyt4 ([#204](https://github.com/NousResearch/hermes-agent/pull/204))

**Discord:**
- Channel topic included in session context -- by @Bartok9 ([#248](https://github.com/NousResearch/hermes-agent/pull/248))
- `DISCORD_ALLOW_BOTS` config for bot message filtering ([#758](https://github.com/NousResearch/hermes-agent/pull/758))
- Document and video support ([#784](https://github.com/NousResearch/hermes-agent/pull/784))

**Slack:**
- Socket Mode integration, threading, documents/video ([#784](https://github.com/NousResearch/hermes-agent/pull/784))
- Structured logging replacing print statements

**WhatsApp:**
- Baileys Node.js bridge, native media sending -- images, videos, documents -- by @satelerd ([#292](https://github.com/NousResearch/hermes-agent/pull/292))
- Multi-user session isolation -- by @satelerd ([#75](https://github.com/NousResearch/hermes-agent/pull/75))

**Signal:**
- Full Signal messenger gateway via signal-cli-rest-api ([#405](https://github.com/NousResearch/hermes-agent/issues/405))
- Media URL support in message events ([#871](https://github.com/NousResearch/hermes-agent/pull/871))

**Email:**
- New email gateway platform (IMAP/SMTP) -- by @0xbyt4

**Home Assistant:**
- REST tools + WebSocket gateway integration -- by @0xbyt4 ([#184](https://github.com/NousResearch/hermes-agent/pull/184))
- Service discovery and enhanced setup

**Gateway Core:**
- `edit_message()` for Telegram/Discord/Slack with fallback
- `/compress`, `/usage`, `/update` slash commands
- Eliminated 3x SQLite message duplication in gateway sessions ([#873](https://github.com/NousResearch/hermes-agent/pull/873))
- Stabilize system prompt across gateway turns for cache hits ([#754](https://github.com/NousResearch/hermes-agent/pull/754))
- Configurable background process watcher notifications ([#840](https://github.com/NousResearch/hermes-agent/pull/840))
- Session reset policy (daily/idle/both/none) with per-platform and per-type overrides

---

### MCP (Model Context Protocol) Client

Native MCP client with stdio and HTTP transports, reconnection, resource/prompt discovery, and sampling (server-initiated LLM requests). ([#291](https://github.com/NousResearch/hermes-agent/pull/291) -- @0xbyt4, [#301](https://github.com/NousResearch/hermes-agent/pull/301), [#753](https://github.com/NousResearch/hermes-agent/pull/753))

- Automatic reconnection on disconnect
- Tool prefix: `mcp_<server>_<tool_name>`
- Per-session MCP server injection via ACP
- `/reload-mcp` command
- `hermes tools` UI integration
- Resource and prompt discovery

---

### ACP Server

VS Code, Zed, and JetBrains editor integration via the Agent Communication Protocol standard. ([#949](https://github.com/NousResearch/hermes-agent/pull/949))

- Full ACP protocol: initialize, authenticate, new/load/resume/fork/list sessions, prompt, cancel
- Streaming: thinking, tool_start, tool_end, step, message events
- Multi-session via `ThreadPoolExecutor(4)`
- `hermes-acp` toolset (excludes audio/TTS/clarify)

---

### CLI & User Experience

#### CLI Skin/Theme Engine

Data-driven visual customization: banners, spinners, colors, branding. 7 built-in skins (`default`, `ares`, `mono`, `slate`, `poseidon`, `sisyphus`, `charizard`) plus custom YAML skins.

#### New Commands and Features

- **Git Worktree Isolation** -- `hermes -w` launches isolated agent sessions in git worktrees for safe parallel work ([#654](https://github.com/NousResearch/hermes-agent/pull/654))
- **Filesystem Checkpoints & Rollback** -- automatic snapshots before destructive operations, `/rollback` to restore ([#824](https://github.com/NousResearch/hermes-agent/pull/824))
- `/personality` command with custom personality and disable support -- by @teyrebaz33 ([#773](https://github.com/NousResearch/hermes-agent/pull/773))
- User-defined quick commands that bypass the agent loop -- by @teyrebaz33 ([#746](https://github.com/NousResearch/hermes-agent/pull/746))
- `/reasoning` command for effort level and display toggle ([#921](https://github.com/NousResearch/hermes-agent/pull/921))
- `/insights` command -- usage analytics, cost estimation, activity patterns ([#552](https://github.com/NousResearch/hermes-agent/pull/552))
- `/background` command for managing background processes
- `--quiet/-Q` flag for programmatic single-query mode
- `--fuck-it-ship-it` flag to bypass all approval prompts -- by @dmahan93 ([#724](https://github.com/NousResearch/hermes-agent/pull/724))
- Clipboard image paste (Alt+V / Ctrl+V)
- Up/down arrow history navigation
- Bell-on-complete -- terminal bell when agent finishes ([#738](https://github.com/NousResearch/hermes-agent/pull/738))
- `hermes doctor` for health checks across all configured providers
- `hermes tools` -- per-platform tool enable/disable with curses UI
- Config migration system (currently v7)

---

### Skills Ecosystem (70+ Skills)

Per-platform skill enable/disable, conditional activation based on tool availability, and prerequisite validation. Compatible with the agentskills.io open standard. ([#743](https://github.com/NousResearch/hermes-agent/pull/743), [#785](https://github.com/NousResearch/hermes-agent/pull/785))

---

### Security Hardening (20+ Fixes)

- Path traversal fix in skill_view -- prevented reading arbitrary files -- by @Farukest
- Shell injection prevention in sudo password piping -- by @leonsgithub ([#65](https://github.com/NousResearch/hermes-agent/pull/65))
- Dangerous command detection: multiline bypass fix -- by @Farukest ([#233](https://github.com/NousResearch/hermes-agent/pull/233))
- Symlink boundary check fix in skills_guard -- by @Farukest ([#386](https://github.com/NousResearch/hermes-agent/pull/386))
- Multi-word prompt injection bypass prevention -- by @0xbyt4 ([#192](https://github.com/NousResearch/hermes-agent/pull/192))
- Cron prompt injection scanner bypass fix -- by @0xbyt4 ([#63](https://github.com/NousResearch/hermes-agent/pull/63))
- Enforce 0600/0700 file permissions on sensitive files ([#757](https://github.com/NousResearch/hermes-agent/pull/757))
- FTS5 query sanitization + DB connection leak fix -- by @0xbyt4 ([#565](https://github.com/NousResearch/hermes-agent/pull/565))
- Atomic writes for all critical files: sessions.json, cron jobs, .env config, process checkpoints, batch runner, skill files
- In-memory permanent allowlist to prevent data leak -- by @alireza78a ([#600](https://github.com/NousResearch/hermes-agent/pull/600))

---

### Testing

**3,289 tests** across agent loop, tool calling, gateway (all 7 platforms), cron, CLI, security, sessions, ACP, skills, honcho, and trajectory format. Parallelized with pytest-xdist. ([#802](https://github.com/NousResearch/hermes-agent/pull/802))

---

### Notable Bug Fixes

- Fix DeepSeek V3 tool call parser silently dropping multi-line JSON -- by @PercyDikec ([#444](https://github.com/NousResearch/hermes-agent/pull/444))
- Fix gateway transcript losing 1 message per turn -- by @PercyDikec ([#395](https://github.com/NousResearch/hermes-agent/pull/395))
- Fix /retry command silently discarding the agent's final response -- by @PercyDikec ([#441](https://github.com/NousResearch/hermes-agent/pull/441))
- Fix SQLite session transcript accumulating duplicate messages -- 3-4x token inflation ([#860](https://github.com/NousResearch/hermes-agent/issues/860))
- Fix OPENROUTER_API_KEY resolution order across all code paths -- by @0xbyt4 ([#295](https://github.com/NousResearch/hermes-agent/pull/295))
- Fix Anthropic "prompt is too long" 400 not detected as context length error ([#813](https://github.com/NousResearch/hermes-agent/issues/813))
- Fix provider selection not persisting when switching via hermes model ([#881](https://github.com/NousResearch/hermes-agent/pull/881))
- Fix Honcho auto-enable when API key is present -- by @Bartok9 ([#243](https://github.com/NousResearch/hermes-agent/pull/243))

---

### Contributors

**Core:** @teknium1 -- 43 PRs: project lead, core architecture, provider router, sessions, skills, CLI, documentation.

**Top community contributors:** @0xbyt4 (40 PRs), @Farukest (16 PRs), @aydnOktay (11 PRs), @Bartok9 (9 PRs), @PercyDikec (7 PRs), @teyrebaz33 (5 PRs), @alireza78a (5 PRs), @shitcoinsherpa (3 PRs), @Himess (3 PRs).

63 total contributors.

**Full changelog:** [v0.1.0...v2026.3.12](https://github.com/NousResearch/hermes-agent/compare/v0.1.0...v2026.3.12)

---

## v0.1.0 -- February 24, 2026

Initial internal release.

### Core

- `AIAgent` class in `run_agent.py` with ReAct loop
- OpenAI-compatible provider via OpenRouter
- Anthropic native provider
- SQLite session storage (WAL mode, FTS5 full-text search)
- Local terminal backend
- Basic CLI (`hermes` command)
- Telegram gateway
- Discord gateway
- Slack gateway
- First bundled skills (GitHub, web research, systematic-debugging)
- `session_search` tool
- `memory` tool
- Context compression (basic)

---

*Pre-release internal prototypes from approximately February 10-23, 2026 are not documented here.*
