# Changelog

All notable changes to Hermes Agent are documented here.

**Current stable release: v0.3.0** (v2026.3.17, March 17, 2026)

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
