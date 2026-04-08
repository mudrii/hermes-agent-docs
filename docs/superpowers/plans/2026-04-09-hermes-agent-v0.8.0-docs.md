# Hermes Agent v0.8.0 Documentation Review & Update — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Perform a release-bounded audit and update of all 84 existing documentation files in `~/src/claw/hermes-agent-docs`, create 3 new content pages and 4 audit files for v0.8.0, and push a single verified commit to GitHub.

**Architecture:** Nine parallel agents (Tasks 1–8 run concurrently, Task 9 runs after all others complete). Each agent owns a domain, reads released source at tag `v2026.4.8`, validates against `RELEASE_v0.8.0.md`, and writes or updates its assigned files. Nothing from `origin/main` beyond `v2026.4.8` is documented as released behavior.

**Tech Stack:** Git (fact-sourcing), Markdown (output), `~/src/claw/hermes-agent` at tag `v2026.4.8` (primary source), `RELEASE_v0.8.0.md` (release notes), GitHub release API (release confirmation).

---

## Source reference constants (used across all tasks)

```
SOURCE_REPO=~/src/claw/hermes-agent
DOCS_REPO=~/src/claw/hermes-agent-docs
RELEASE_TAG=v2026.4.8
PREV_TAG=v2026.4.3
VERSION=0.8.0
RELEASE_DATE=2026-04-08
```

All `git show` commands below use `git -C $SOURCE_REPO show $RELEASE_TAG:<path>`.

---

## Task 1 — Release control & audit scaffolding

**Files:**
- Create: `review_plan_v2026.4.8.md`
- Create: `release_evaluation_v2026.4.8.md`
- Create: `repo_review_v2026.4.8.md`

- [ ] **Step 1.1: Confirm the release tag and metadata**

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8 --stat | head -5
git log v2026.4.3..v2026.4.8 --oneline | wc -l
git diff --stat v2026.4.3..v2026.4.8 | tail -1
```

Expected output:
```
tag v2026.4.8
...
337
 582 files changed, 76163 insertions(+), 6750 deletions(-)
```

- [ ] **Step 1.2: Validate online release metadata**

```bash
curl -s "https://api.github.com/repos/NousResearch/hermes-agent/releases/latest" \
  | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['tag_name'], r['draft'], r['prerelease'], r['published_at'])"
```

Expected: `v2026.4.8 False False 2026-04-08T...`

- [ ] **Step 1.3: Identify commits excluded from release (main-after-tag)**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.8..origin/main --oneline | head -20
git diff --stat v2026.4.8..origin/main | tail -1
```

Record the output — it defines what is NOT in scope for this docs pass.

- [ ] **Step 1.4: Write `review_plan_v2026.4.8.md`**

Create `~/src/claw/hermes-agent-docs/review_plan_v2026.4.8.md`:

```markdown
# Review Plan: Hermes Agent v0.8.0 release-bound audit (2026-04-09)

## Scope and release boundary

This review is restricted to released Hermes Agent artifacts as of April 9, 2026.

- Latest published release: `v2026.4.8` / Hermes Agent `v0.8.0`
- Previous release used for delta sizing: `v2026.4.3` / `v0.7.0`
- Local `origin/main` is ahead of `v2026.4.8` and is treated as unreleased for documentation purposes

## Primary sources

- Local git tag `v2026.4.8` in `~/src/claw/hermes-agent`
- Local release note file `~/src/claw/hermes-agent/RELEASE_v0.8.0.md`
- GitHub latest-release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- GitHub release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.8`
- GitHub compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.3...v2026.4.8`

## Secondary validation sources

Used only after source/release-note confirmation:
- `https://hermes-agent.nousresearch.com/docs/`

## Workstreams

### 1) Release control
- Confirm latest published release tag, metadata, and commit IDs
- Measure `v2026.4.3..v2026.4.8` delta
- Identify unreleased main-after-tag commits to exclude

### 2) Source audit
- Review released source structure across all major subsystems
- Map subsystems to existing docs coverage
- Identify new subsystems with no existing doc page

### 3) Full docs audit
- Review all 84 existing docs files for stale version framing, factual errors, broken cross-references
- Update all files to reflect v0.8.0 released behavior
- Add new pages for: logging, live model switching, notify_on_complete

### 4) External validation
- Validate release against GitHub API
- Cross-check public docs site for facts also present in released source

### 5) Integrity pass
- Aggregate findings from all domain agents
- Produce `documentation_integrity_findings_v2026.4.8.md`
- Update `README.md` index and `changelog.md`

## Deliverables

New files:
- `logging.md`
- `live-model-switching.md`
- `notify-on-complete.md`
- `review_plan_v2026.4.8.md`
- `release_evaluation_v2026.4.8.md`
- `repo_review_v2026.4.8.md`
- `documentation_integrity_findings_v2026.4.8.md`

Updated files: all 84 existing documentation files reviewed; those with stale content updated.
```

- [ ] **Step 1.5: Write `release_evaluation_v2026.4.8.md`**

Create `~/src/claw/hermes-agent-docs/release_evaluation_v2026.4.8.md` with:
- Tag confirmation (from Step 1.2 output)
- Delta stats (from Step 1.1 output)
- Main-after-release exclusion list (from Step 1.3 output, first 10–15 notable commits)
- Online validation summary with URLs
- Key new subsystems in this release:
  - Centralized logging (`hermes_logging.py`, `~/.hermes/logs/`)
  - Live model switching (`/model` command, `gateway/stream_consumer.py`)
  - `notify_on_complete` tool (`tools/process_registry.py`)
  - Google AI Studio provider
  - Supermemory memory provider
  - MCP OAuth 2.1 PKCE
  - Matrix Tier 1 feature parity
  - Plugin session lifecycle hooks (`on_session_finalize`, `on_session_reset`)
  - Config structure validation at startup

- [ ] **Step 1.6: Write `repo_review_v2026.4.8.md`**

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8 --name-only | grep "^[a-z]" | cut -d'/' -f1 | sort -u
git diff --stat v2026.4.3..v2026.4.8 | grep -E "^[a-z]" | awk '{print $1}' | cut -d'/' -f1 | sort | uniq -c | sort -rn | head -20
```

Create `~/src/claw/hermes-agent-docs/repo_review_v2026.4.8.md` documenting:
- Top-level directory structure at `v2026.4.8`
- Highest-impact directories by file change count (from command above)
- Subsystem → existing docs coverage map
- New subsystems with no prior docs page

- [ ] **Step 1.7: Commit audit scaffolding**

```bash
cd ~/src/claw/hermes-agent-docs
git add review_plan_v2026.4.8.md release_evaluation_v2026.4.8.md repo_review_v2026.4.8.md
git commit -m "docs: add v0.8.0 review plan, release evaluation, and repo review"
```

---

## Task 2 — Core architecture & configuration (parallel with Tasks 3–8)

**Files:**
- Modify: `architecture.md`
- Modify: `configuration.md`
- Modify: `installation.md`
- Modify: `environments.md`
- Modify: `getting-started/learning-path.md`
- Modify: `getting-started/updating.md`
- Modify: `getting-started/nix-setup.md`
- Modify: `developer-guide/agent-loop.md`
- Modify: `developer-guide/architecture.md`
- Modify: `developer-guide/context-compression.md`
- Modify: `developer-guide/prompt-assembly.md`
- Create: `logging.md`

- [ ] **Step 2.1: Read source for logging system**

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:run_agent.py | grep -A 20 "Centralized logging"
git show v2026.4.8:hermes_cli/__init__.py | grep -A 30 "logs\|log_cmd\|hermes logs"
```

Key facts confirmed from source:
- `setup_logging()` called from `run_agent.py:AIAgent.__init__`
- Log directory: `~/.hermes/logs/`
- `agent.log`: INFO and above
- `errors.log`: WARNING and above
- Quiet mode suppresses console output but file handlers still capture everything

- [ ] **Step 2.2: Read source for config validation**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "config.*valid\|valid.*config"
git show v2026.4.8:cli.py | grep -n "config.*valid\|validate_config\|startup" | head -20
```

- [ ] **Step 2.3: Read source for prompt assembly changes**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- agent/prompt_builder.py | head -80
```

- [ ] **Step 2.4: Verify no stale version framing in target files**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" architecture.md configuration.md installation.md environments.md getting-started/ developer-guide/agent-loop.md developer-guide/architecture.md developer-guide/context-compression.md developer-guide/prompt-assembly.md 2>/dev/null
```

Update every instance found to reflect v0.8.0 behavior.

- [ ] **Step 2.5: Create `logging.md`**

Create `~/src/claw/hermes-agent-docs/logging.md` with:

```markdown
# Logging

Hermes Agent v0.8.0 introduced centralized structured logging to persistent files under `~/.hermes/logs/`.

## Log files

| File | Level | Contents |
|---|---|---|
| `~/.hermes/logs/agent.log` | INFO and above | All agent activity, tool calls, LLM calls |
| `~/.hermes/logs/errors.log` | WARNING and above | Errors and warnings only |

Logs rotate automatically and are written regardless of quiet mode. Console output is suppressed in quiet mode (the default for interactive CLI), but file handlers always capture everything.

## `hermes logs` command

Tail and filter log output:

```
hermes logs              # tail agent.log
hermes logs --errors     # tail errors.log
hermes logs --follow     # follow in real time (like tail -f)
hermes logs --filter KEYWORD  # filter lines containing KEYWORD
```

## Quiet mode behavior

In interactive CLI mode (`hermes chat`), the following logger namespaces are silenced at console level to avoid cluttering the TUI:

- `tools` (all tool modules)
- `run_agent` (agent runner internals)
- `trajectory_compressor`
- `cron`
- `hermes_cli`

File handlers (`agent.log`, `errors.log`) are unaffected by quiet mode.

## Verbose mode

Pass `--verbose` to enable third-party library debug output:

```
hermes chat --verbose
```

## Configuration

Logging is initialized in `AIAgent.__init__` via `hermes_logging.setup_logging()`. No configuration is required — the log directory is created automatically under `get_hermes_home()`.
```

- [ ] **Step 2.6: Update `configuration.md`**

Read current file, then add/update:
- Config structure validation at startup (new in v0.8.0): malformed YAML now produces actionable error messages before the agent starts
- `reasoning_effort` unified to `config.yaml` only (env var `HERMES_REASONING_EFFORT` removed in v0.8.0)
- Reference to new `logging.md` page

Verify with:
```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:cli.py | grep -n "reasoning_effort\|HERMES_REASONING_EFFORT" | head -10
```

- [ ] **Step 2.7: Update `installation.md` and `getting-started/updating.md`**

Check for stale version numbers and update version references to v0.8.0. Verify `hermes update` behavior:
```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:hermes_cli/__init__.py | grep -A 20 "update"
```

- [ ] **Step 2.8: Update developer guide files**

For `developer-guide/agent-loop.md`: document jittered retry backoff (new in v0.8.0), smart thinking block signature management, and accept-reasoning-only-responses behavior.

For `developer-guide/prompt-assembly.md`: document self-optimized GPT/Codex tool-use guidance and thinking-only prefill continuation.

Verify with:
```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- agent/prompt_builder.py | grep "^+" | head -40
```

- [ ] **Step 2.9: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add architecture.md configuration.md installation.md environments.md \
  getting-started/ developer-guide/agent-loop.md developer-guide/architecture.md \
  developer-guide/context-compression.md developer-guide/prompt-assembly.md \
  logging.md
git commit -m "docs: update core architecture and configuration docs for v0.8.0; add logging.md"
```

---

## Task 3 — Providers & model system (parallel with Tasks 2, 4–8)

**Files:**
- Modify: `providers.md`
- Modify: `provider-routing.md`
- Modify: `fallback-providers.md`
- Modify: `credential-pools.md`
- Modify: `developer-guide/adding-providers.md`
- Modify: `developer-guide/provider-runtime.md`
- Modify: `reference/environment-variables.md`
- Create: `live-model-switching.md`

- [ ] **Step 3.1: Read source for Google AI Studio provider**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "google\|gemini\|studio"
git show v2026.4.8:cli.py | grep -n "google\|gemini\|studio" | head -20
```

- [ ] **Step 3.2: Read source for `/model` command**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "/model\|model.*switch\|live.*model"
git show v2026.4.8:gateway/stream_consumer.py | grep -n "model\|/model" | head -30
```

Key facts to document in `live-model-switching.md`:
- `/model` slash command works in CLI, Telegram, Discord, Slack, and all gateway platforms
- Aggregator-aware: stays on OpenRouter/Nous Portal when possible
- Automatic cross-provider fallback when model not available on current aggregator
- Interactive model pickers on Telegram (paginated with Next/Prev buttons) and Discord (inline buttons)
- Model switch persists for the session
- `/model` with no arguments shows current model

- [ ] **Step 3.3: Read source for credential pool and OAuth changes**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- credential_pool.py 2>/dev/null | grep "^+" | head -40
git log v2026.4.3..v2026.4.8 --oneline | grep -i "oauth\|credential\|codex.*pool\|pool"
```

- [ ] **Step 3.4: Read source for new provider-related environment variables**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- cli.py | grep "^+.*HERMES_\|^+.*environ" | head -30
```

New env vars confirmed in v0.8.0:
- `HERMES_PORTAL_BASE_URL` — override Nous Portal base URL during login
- `Z_AI_ENDPOINT` — Z.AI endpoint override (auto-probed and cached if not set)

- [ ] **Step 3.5: Verify no stale version framing in target files**

```bash
cd ~/src/claw/hermes-agent-docs
grep -n "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" providers.md provider-routing.md fallback-providers.md credential-pools.md reference/environment-variables.md developer-guide/adding-providers.md developer-guide/provider-runtime.md 2>/dev/null
```

- [ ] **Step 3.6: Update `providers.md`**

Add entries for:
- Google AI Studio (Gemini) — native provider, models.dev integration for auto context-length detection
- Xiaomi MiMo v2 Pro — free-tier on Nous Portal for auxiliary tasks (compression, vision, summarization)
- MiniMax — corrected context lengths and model catalog; TTS provider (speech-2.8)
- xAI (Grok) — prompt caching via `x-grok-conv-id` header; added to tool-use enforcement list
- Z.AI — endpoint auto-detect via probe and cache

Update existing entries:
- Nous Portal: free-tier model gating, pricing display, `HERMES_PORTAL_BASE_URL` env var
- OpenRouter: stale OAuth credentials no longer block auto-detect
- Codex: OAuth credential pool disconnect fixed; expired token import fixed; pool entry sync from `~/.codex/auth.json`
- Ollama: Cloud auth added, `/model` switch persistence, alias tab completion

- [ ] **Step 3.7: Create `live-model-switching.md`**

```markdown
# Live Model Switching

Hermes Agent v0.8.0 introduced the `/model` command for switching models and providers mid-session without restarting.

## Usage

```
/model                          # show current model and provider
/model gpt-4o                   # switch to gpt-4o on current provider
/model openrouter/gpt-4o        # switch to gpt-4o via OpenRouter
/model nous/hermes-3-70b        # switch to Hermes 3 70B via Nous Portal
```

## Aggregator-aware resolution

The `/model` command is aggregator-aware. When your current provider is OpenRouter or Nous Portal, the switch stays within the same aggregator when possible. Cross-provider fallback occurs automatically when the requested model is not available through the current aggregator.

## Platform support

| Platform | Model switching | Interactive picker |
|---|---|---|
| CLI | `/model <name>` command | — |
| Telegram | `/model <name>` command | Paginated inline buttons (Next/Prev) |
| Discord | `/model <name>` command | Inline button dropdown |
| Slack | `/model <name>` command | — |
| All gateway platforms | `/model <name>` command | — |

## Session persistence

Model switches persist for the current session. The switch does not modify `config.yaml`. To make a switch permanent, update `model.default` in `~/.hermes/config.yaml`.

## Pricing display

When switching models on OpenRouter or Nous Portal, the picker displays pricing information for each model.
```

- [ ] **Step 3.8: Update `reference/environment-variables.md`**

Add new variables: `HERMES_PORTAL_BASE_URL`, `Z_AI_ENDPOINT`. Verify existing Nous/OpenRouter/Codex entries are accurate.

- [ ] **Step 3.9: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add providers.md provider-routing.md fallback-providers.md credential-pools.md \
  developer-guide/adding-providers.md developer-guide/provider-runtime.md \
  reference/environment-variables.md live-model-switching.md
git commit -m "docs: update provider and model system docs for v0.8.0; add live-model-switching.md"
```

---

## Task 4 — Memory, sessions & persistence (parallel with Tasks 2–3, 5–8)

**Files:**
- Modify: `memory.md`
- Modify: `honcho.md`
- Modify: `checkpoints.md`
- Modify: `profiles.md`
- Modify: `delegation.md`
- Modify: `developer-guide/memory-provider-plugin.md`
- Modify: `developer-guide/session-storage.md`
- Modify: `developer-guide/trajectory-format.md`

- [ ] **Step 4.1: Read source for Supermemory provider**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "supermemory"
git show v2026.4.8:plugins/memory/ 2>/dev/null | head -5 || ls -la ~/src/claw/hermes-agent/plugins/memory/ 2>/dev/null
```

Key facts about Supermemory (from release notes and source):
- New external memory provider
- Supports multi-container configuration
- `search_mode` option
- Identity template support
- Env var override support

- [ ] **Step 4.2: Read source for Hindsight overhaul**

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:plugins/memory/hindsight/__init__.py | head -80
git log v2026.4.3..v2026.4.8 --oneline | grep -i "hindsight"
```

- [ ] **Step 4.3: Read source for mem0 v2 changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "mem0"
```

Key changes: API v2 compatibility, prefetch context fencing, secret redaction. `mem0` env vars now merged with `mem0.json` instead of either/or.

- [ ] **Step 4.4: Read source for shared thread sessions**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "thread\|session\|shared"
```

Key change: shared thread sessions by default — multi-user thread support across gateway platforms. Subagent sessions linked to parent and hidden from session list.

- [ ] **Step 4.5: Verify no stale version framing**

```bash
cd ~/src/claw/hermes-agent-docs
grep -n "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" memory.md honcho.md checkpoints.md profiles.md delegation.md developer-guide/memory-provider-plugin.md developer-guide/session-storage.md developer-guide/trajectory-format.md 2>/dev/null
```

- [ ] **Step 4.6: Update `memory.md`**

Add Supermemory to the memory providers table with: multi-container, search_mode, identity template, env var override. Update Hindsight section for the overhaul in v0.8.0. Update mem0 section for API v2 compat and env var merge behavior. Update RetainDB section for API routes, write queue, dialectic fixes.

- [ ] **Step 4.7: Update `honcho.md`**

Document plugin drift overhaul and plugin CLI registration system (new in v0.8.0). Document holographic prompt and trust score rendering. Fix `recall_mode` (not `memory_mode`) per source.

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:hermes_cli/plugins.py | grep -n "honcho\|recall_mode\|memory_mode" | head -10
```

- [ ] **Step 4.8: Update `profiles.md`**

Add profile-scoped memory isolation and clone support (new in v0.8.0). Document profile-aware service units.

- [ ] **Step 4.9: Update developer guide memory files**

For `developer-guide/memory-provider-plugin.md`: update hook list to include new `on_session_finalize` and `on_session_reset` lifecycle hooks. Document `thread_gateway_user_id` parameter for per-user scoping.

- [ ] **Step 4.10: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add memory.md honcho.md checkpoints.md profiles.md delegation.md \
  developer-guide/memory-provider-plugin.md developer-guide/session-storage.md \
  developer-guide/trajectory-format.md
git commit -m "docs: update memory, sessions, and persistence docs for v0.8.0"
```

---

## Task 5 — Gateway & all messaging platforms (parallel with Tasks 2–4, 6–8)

**Files:**
- Modify: `gateway.md`
- Modify: `messaging/README.md`
- Modify: `messaging/telegram.md`
- Modify: `messaging/discord.md`
- Modify: `messaging/slack.md`
- Modify: `messaging/matrix.md`
- Modify: `messaging/signal.md`
- Modify: `messaging/mattermost.md`
- Modify: `messaging/feishu.md`
- Modify: `messaging/email.md`
- Modify: `messaging/webhook.md`
- Modify: `messaging/dingtalk.md`
- Modify: `messaging/wecom.md`
- Modify: `messaging/homeassistant.md`
- Modify: `messaging/open-webui.md`
- Modify: `messaging/sms.md`
- Modify: `messaging/whatsapp.md`
- Modify: `developer-guide/gateway-internals.md`

- [ ] **Step 5.1: Read source for gateway core changes**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- gateway/run.py | head -80
git diff v2026.4.3..v2026.4.8 -- gateway/stream_consumer.py | head -80
```

Key gateway core changes in v0.8.0:
- Inactivity-based timeout (tracks tool activity, not wall-clock time)
- Approval buttons for Slack & Telegram (native platform buttons instead of typing `/approve`)
- Live-stream `/update` output + interactive prompt forwarding
- Infinite timeout support + periodic notifications
- Duplicate message prevention (dedup + partial stream guard)
- Webhook `delivery_info` persistence + full session id in `/status`
- Tool preview truncation respects `tool_preview_length`
- Typing indicator paused during approval waits
- `MEDIA:` tags stripped from streamed gateway messages

- [ ] **Step 5.2: Read source for Telegram changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "telegram"
git diff v2026.4.3..v2026.4.8 -- gateway/platforms/telegram*.py 2>/dev/null | grep "^+.*def \|^+.*class " | head -20
```

Key Telegram changes: group topics skill binding, emoji reactions for approval status, paginated model picker with Next/Prev, duplicate delivery prevention on send timeout, command name sanitization.

- [ ] **Step 5.3: Read source for Discord changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "discord"
```

Key Discord changes: `ignored_channels` and `no_thread_channels` config options, skills registered as native slash commands, interactive model picker with inline buttons, unnecessary members intent removed.

- [ ] **Step 5.4: Read source for Slack changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "slack"
```

Key Slack changes: auto-respond in bot-started and mentioned threads, `mrkdwn` in `edit_message`, thread replies without @mentions, native approval buttons.

- [ ] **Step 5.5: Read source for Matrix Tier 1**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "matrix"
```

Matrix is a major upgrade in v0.8.0. Key changes: reactions, read receipts, rich formatting, room management, `MATRIX_REQUIRE_MENTION`, `MATRIX_AUTO_THREAD`, encrypted media, auth recovery, cron E2EE, Synapse compat, CJK input, E2EE reconnect fixes.

- [ ] **Step 5.6: Read source for Signal, Mattermost, Feishu, Webhook changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -iE "signal|mattermost|feishu|webhook"
```

Key changes:
- Signal: full MEDIA: tag delivery — `send_image_file`, `send_voice`, `send_video` implemented
- Mattermost: file attachments (DOCUMENT type when post has file attachments)
- Feishu: interactive card approval buttons, reconnect and ACL fixes
- Webhook: `{__raw__}` template token, `thread_id` passthrough for forum topics

- [ ] **Step 5.7: Verify no stale version framing in all messaging files**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" gateway.md messaging/ developer-guide/gateway-internals.md 2>/dev/null
```

- [ ] **Step 5.8: Update `gateway.md`**

Document inactivity-based timeout (explain the difference from wall-clock timeout), approval buttons, duplicate message prevention, profile-aware service units, and thread-safe PairingStore.

- [ ] **Step 5.9: Update `messaging/matrix.md`**

Substantial update — Matrix moved to Tier 1 in v0.8.0. Document all new features:
- Reactions and read receipts
- Rich formatting support
- Room management commands
- `MATRIX_REQUIRE_MENTION` env var
- `MATRIX_AUTO_THREAD` env var
- E2EE (end-to-end encryption) support and cron E2EE
- Synapse compatibility
- Encrypted media handling

- [ ] **Step 5.10: Update `messaging/telegram.md`**

Add: group topics skill binding, emoji reactions for approval status, paginated model picker, `/approve` and `/deny` button-based approval (no longer requires typing), per-platform disabled skills.

- [ ] **Step 5.11: Update `messaging/discord.md`**

Add: `ignored_channels` config option, `no_thread_channels` config option, native slash command registration for skills, interactive model picker.

- [ ] **Step 5.12: Update `messaging/slack.md`**

Add: auto-respond in bot-started threads, approval buttons, thread engagement behavior.

- [ ] **Step 5.13: Update `messaging/signal.md`**

Add: MEDIA: tag delivery support — images, voice, and video now delivered natively.

- [ ] **Step 5.14: Update `messaging/mattermost.md`**

Add: file attachment support (DOCUMENT message type).

- [ ] **Step 5.15: Update `messaging/feishu.md`**

Add: interactive card approval buttons.

- [ ] **Step 5.16: Update `messaging/webhook.md`**

Add: `{__raw__}` template token, `thread_id` passthrough for forum topics, `delivery_info` persistence.

- [ ] **Step 5.17: Review remaining messaging files**

Review `messaging/email.md`, `messaging/dingtalk.md`, `messaging/wecom.md`, `messaging/homeassistant.md`, `messaging/open-webui.md`, `messaging/sms.md`, `messaging/whatsapp.md` for stale content. Update version framing where found. Verify all config options listed are still accurate at `v2026.4.8`.

- [ ] **Step 5.18: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add gateway.md messaging/ developer-guide/gateway-internals.md
git commit -m "docs: update gateway and all messaging platform docs for v0.8.0 (Matrix Tier 1, approval buttons, Signal MEDIA)"
```

---

## Task 6 — CLI, tools & skills (parallel with Tasks 2–5, 7–8)

**Files:**
- Modify: `cli-reference.md`
- Modify: `tools.md`
- Modify: `toolsets.md`
- Modify: `skills.md`
- Modify: `code-execution.md`
- Modify: `batch-processing.md`
- Modify: `streaming.md`
- Modify: `developer-guide/adding-tools.md`
- Modify: `developer-guide/tools-runtime.md`
- Modify: `developer-guide/extending-cli.md`
- Modify: `developer-guide/creating-skills.md`
- Modify: `reference/cli-commands.md`
- Modify: `reference/slash-commands.md`
- Modify: `reference/tools-reference.md`
- Modify: `reference/toolsets-reference.md`
- Modify: `reference/skills-catalog.md`
- Modify: `reference/optional-skills-catalog.md`
- Modify: `reference/profile-commands.md`
- Create: `notify-on-complete.md`

- [ ] **Step 6.1: Read source for `notify_on_complete` tool**

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:tools/process_registry.py | head -100
git log v2026.4.3..v2026.4.8 --oneline | grep -i "notify\|process_registry"
```

Key facts confirmed:
- `notify_on_complete` works with background processes spawned via `terminal(background=true)`
- Process registry: `tools/process_registry.py`
- 200KB rolling output buffer per process
- Status polling, log retrieval, blocking wait, kill, crash recovery via JSON checkpoint
- Session-scoped tracking for gateway reset protection
- Background processes run THROUGH the environment interface (Docker, Singularity, Modal, etc.)

- [ ] **Step 6.2: Read source for tool result persistence changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "budget\|threshold\|persist\|tool.*result"
git show v2026.4.8:cli.py | grep -n "budget\|threshold\|persist" | head -20
```

Key changes: configurable tool result persistence thresholds (previously hardcoded), sandbox-aware tool result persistence.

- [ ] **Step 6.3: Read source for CLI changes**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- cli.py | grep "^+.*def \|^+.*subcommand\|^+.*--" | head -30
git log v2026.4.3..v2026.4.8 --oneline | grep -i "cli\|flag\|--yolo\|allowlist"
```

Key CLI changes:
- `--yolo` and other flags no longer silently dropped before `chat` subcommand
- Permanent command allowlist loaded on startup
- `hermes auth remove` now clears env-seeded credentials permanently
- `hermes update` no longer kills freshly-restarted gateway service
- Subprocess timeouts added to all gateway CLI commands
- Stale `hermes login` replaced with `hermes auth`

- [ ] **Step 6.4: Read source for skills system changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "skill"
```

Key changes: bundled skills synced to all profiles during update, per-platform disabled skills respected in Telegram menu.

- [ ] **Step 6.5: Verify no stale version framing**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" cli-reference.md tools.md toolsets.md skills.md code-execution.md batch-processing.md streaming.md reference/ developer-guide/adding-tools.md developer-guide/tools-runtime.md developer-guide/extending-cli.md developer-guide/creating-skills.md 2>/dev/null
```

- [ ] **Step 6.6: Create `notify-on-complete.md`**

```markdown
# Background Task Notifications (`notify_on_complete`)

Hermes Agent v0.8.0 introduced automatic agent notification when background processes complete.

## Overview

Start a long-running process in the background and the agent is automatically notified when it finishes. The agent can keep working on other tasks while the background process runs.

```
terminal(command="pytest -v", background=true, notify_on_complete=true)
```

## How it works

1. Agent spawns a background process via `terminal(background=true)`
2. The process is tracked by the process registry (`tools/process_registry.py`)
3. On completion, the registry delivers a notification back to the agent's conversation
4. The agent receives the result and can act on it without polling

## Process registry

The process registry tracks all background processes with:
- **200KB rolling output buffer** — last 200KB of output always available
- **Status polling** — check if a process is running, done, or crashed
- **Blocking wait** — `wait(session_id, timeout=N)` blocks until done or timeout
- **Kill support** — terminate a process by session ID
- **Crash recovery** — JSON checkpoint file survives agent restart
- **Session scoping** — processes are tracked per-session; gateway reset isolation

## Environment behavior

Background processes run through the active environment interface. In Docker, Singularity, Modal, Daytona, or SSH environments, the command runs inside the sandbox — not on the host machine. Only `TERMINAL_ENV=local` runs directly on the host.

## Use cases

- AI model training jobs
- Long test suites
- Build pipelines
- Deployments
- Any task where you don't want to block the agent
```

- [ ] **Step 6.7: Update `tools.md` and `reference/tools-reference.md`**

Add `notify_on_complete` parameter to the `terminal` tool documentation. Add process registry section. Document configurable persistence thresholds.

- [ ] **Step 6.8: Update `cli-reference.md` and `reference/cli-commands.md`**

Add/update:
- `hermes logs` command (new in v0.8.0)
- `hermes auth` (replaces `hermes login`)
- `hermes auth remove` behavior
- `--yolo` flag placement fix

- [ ] **Step 6.9: Update `reference/slash-commands.md`**

Add `/model` command documentation (new in v0.8.0). Reference `live-model-switching.md`.

- [ ] **Step 6.10: Update `developer-guide/extending-cli.md`**

Document plugin CLI subcommand registration (new in v0.8.0 — plugins can now register CLI subcommands).

- [ ] **Step 6.11: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add cli-reference.md tools.md toolsets.md skills.md code-execution.md \
  batch-processing.md streaming.md developer-guide/adding-tools.md \
  developer-guide/tools-runtime.md developer-guide/extending-cli.md \
  developer-guide/creating-skills.md reference/ notify-on-complete.md
git commit -m "docs: update CLI, tools, and skills docs for v0.8.0; add notify-on-complete.md"
```

---

## Task 7 — MCP, plugins, security & cron (parallel with Tasks 2–6, 8)

**Files:**
- Modify: `mcp.md`
- Modify: `plugins.md`
- Modify: `security.md`
- Modify: `cron.md`
- Modify: `hooks.md`
- Modify: `acp.md`
- Modify: `developer-guide/cron-internals.md`
- Modify: `reference/mcp-config-reference.md`
- Modify: `reference/faq.md`

- [ ] **Step 7.1: Read source for MCP OAuth 2.1 PKCE**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "oauth\|pkce\|mcp.*auth\|auth.*mcp"
git show v2026.4.8:mcp/ 2>/dev/null | head -10 || find ~/src/claw/hermes-agent -name "*.py" -path "*/mcp/*" | head -5
```

Key facts: Full standards-compliant OAuth 2.1 with PKCE for MCP server authentication. OSV malware scanning of MCP extension packages.

- [ ] **Step 7.2: Read source for plugin system expansion**

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:hermes_cli/plugins.py | grep -n "VALID_HOOKS\|register_command\|on_session\|api_request\|correlation" | head -20
git log v2026.4.3..v2026.4.8 --oneline | grep -i "plugin\|hook"
```

New plugin capabilities in v0.8.0:
- Register CLI subcommands (`register_command`)
- Request-scoped API hooks with correlation IDs (`pre_api_request`, `post_api_request`)
- Prompt for required env vars during install
- Session lifecycle hooks: `on_session_finalize`, `on_session_reset`

- [ ] **Step 7.3: Read source for security hardening**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "security\|ssrf\|timing\|traversal\|sanitize\|cred.*leak\|isolation"
```

Key security changes:
- Consolidated SSRF protections
- Timing attack mitigations
- Tar traversal prevention
- Credential leakage guards
- Cron path traversal hardening
- Cross-session isolation
- Terminal workdir sanitization across all backends
- OSV vulnerability scanning for MCP packages

- [ ] **Step 7.4: Read source for cron changes**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- cron/scheduler.py | head -60
git log v2026.4.3..v2026.4.8 --oneline | grep -i "cron"
```

Key cron changes: inactivity-based timeout (active tasks run indefinitely), delivery failure tracking in job status, MEDIA: tag extraction before delivery.

- [ ] **Step 7.5: Read source for ACP changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "acp\|copilot"
```

Key ACP changes: client-provided MCP servers in ACP, Copilot ACP tool-call bridge.

- [ ] **Step 7.6: Verify no stale version framing**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" mcp.md plugins.md security.md cron.md hooks.md acp.md developer-guide/cron-internals.md reference/mcp-config-reference.md reference/faq.md 2>/dev/null
```

- [ ] **Step 7.7: Update `mcp.md`**

Add MCP OAuth 2.1 PKCE section: explain PKCE flow, which MCP servers require it, how to configure. Add OSV malware scanning section: automatic scanning on install, what happens when a package has a known vulnerability. Add `no_mcp` sentinel to per-platform tool exclusions.

Verify with:
```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:cli.py | grep -n "no_mcp\|OSV\|pkce\|oauth" | head -10
```

- [ ] **Step 7.8: Update `plugins.md`**

Document new plugin capabilities:
- CLI subcommand registration: `ctx.register_command(name, handler)`
- Request-scoped API hooks with correlation IDs
- `prompt_for_env_vars` in plugin manifest (prompts during install)
- Session lifecycle hooks: `on_session_finalize`, `on_session_reset`

Update VALID_HOOKS list to match source (add new hooks).

- [ ] **Step 7.9: Update `security.md`**

Add sections for: SSRF protection consolidation, timing attack mitigations, MCP package OSV scanning, cross-session isolation, terminal workdir sanitization.

- [ ] **Step 7.10: Update `cron.md`**

Replace wall-clock timeout documentation with inactivity-based timeout explanation. Add delivery failure tracking: job status now includes `delivery_failures` field. Document MEDIA: tag extraction behavior.

- [ ] **Step 7.11: Update `hooks.md`**

Add `on_session_finalize` and `on_session_reset` to the hooks reference table with description and arguments.

- [ ] **Step 7.12: Update `reference/faq.md`**

Check for stale version framing and update. Verify any version-specific answers (e.g., minimum Python version, install commands) are accurate at v0.8.0:

```bash
cd ~/src/claw/hermes-agent
git show v2026.4.8:pyproject.toml | grep -E "python_requires|python ="
```

- [ ] **Step 7.13: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add mcp.md plugins.md security.md cron.md hooks.md acp.md \
  developer-guide/cron-internals.md reference/mcp-config-reference.md reference/faq.md
git commit -m "docs: update MCP (OAuth 2.1), plugins, security, and cron docs for v0.8.0"
```

---

## Task 8 — Media, vision, voice & API server (parallel with Tasks 2–7)

**Files:**
- Modify: `vision.md`
- Modify: `tts.md`
- Modify: `voice-mode.md`
- Modify: `image-generation.md`
- Modify: `api-server.md`
- Modify: `browser.md`
- Modify: `context-references.md`
- Modify: `rl-training.md`
- Modify: `skins.md`
- Modify: `personality.md`

- [ ] **Step 8.1: Read source for vision changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "vision\|image"
git show v2026.4.8:cli.py | grep -n "vision\|image" | head -20
```

Key changes: vision auto-detection simplified (tries main provider first, then OpenRouter, then Nous), Google AI Studio supports vision.

- [ ] **Step 8.2: Read source for TTS changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "tts\|speech\|voice\|minimax.*tts"
```

Key change: MiniMax TTS provider added (speech-2.8).

- [ ] **Step 8.3: Read source for API server changes**

```bash
cd ~/src/claw/hermes-agent
git diff v2026.4.3..v2026.4.8 -- gateway/run.py | grep "^+.*def \|^+.*route\|^+.*endpoint" | head -20
git log v2026.4.3..v2026.4.8 --oneline | grep -i "api.server\|api-server\|gateway.*api"
```

- [ ] **Step 8.4: Read source for browser changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "browser\|camofox"
git show v2026.4.8:cli.py | grep -n "browser" | head -10
```

Key change: browser paths with spaces now preserved (agent-browser fix).

- [ ] **Step 8.5: Read source for skins changes**

```bash
cd ~/src/claw/hermes-agent
git log v2026.4.3..v2026.4.8 --oneline | grep -i "skin\|hermes.mod\|visual"
```

Key change: Hermes Mod visual skin editor section added (documented in release notes).

- [ ] **Step 8.6: Verify no stale version framing**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3" vision.md tts.md voice-mode.md image-generation.md api-server.md browser.md context-references.md rl-training.md skins.md personality.md 2>/dev/null
```

- [ ] **Step 8.7: Update `vision.md`**

Update auto-detection logic: v0.8.0 tries main provider first (not OpenRouter first). Add Google AI Studio to the supported providers for vision.

- [ ] **Step 8.8: Update `tts.md`**

Add MiniMax TTS provider (speech-2.8 model). Verify existing provider table is accurate.

- [ ] **Step 8.9: Update `skins.md`**

Add Hermes Mod visual skin editor section. Verify skin-aware banner behavior is documented.

- [ ] **Step 8.10: Update `api-server.md`, `browser.md`, `context-references.md`, `rl-training.md`, `personality.md`, `voice-mode.md`, `image-generation.md`**

For each file: check for stale version framing and update. Verify all documented features exist in released source at `v2026.4.8`. Add Windows native image paste support to relevant docs (new in v0.8.0).

- [ ] **Step 8.11: Commit**

```bash
cd ~/src/claw/hermes-agent-docs
git add vision.md tts.md voice-mode.md image-generation.md api-server.md \
  browser.md context-references.md rl-training.md skins.md personality.md
git commit -m "docs: update media, vision, TTS, voice, and API server docs for v0.8.0"
```

---

## Task 9 — Integrity findings, README & changelog (runs AFTER Tasks 1–8)

**Files:**
- Create: `documentation_integrity_findings_v2026.4.8.md`
- Modify: `README.md`
- Modify: `changelog.md`
- Modify: `contributing.md`

- [ ] **Step 9.1: Scan all files for remaining stale version framing**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.6\|v0\.7\|v2026\.3\|v2026\.4\.3\|Hermes Agent v0\.[67]\b" \
  --include="*.md" \
  --exclude-dir=docs \
  . 2>/dev/null | grep -v "review_plan\|release_eval\|repo_review\|documentation_integrity\|source_valid\|RELEASE_\|changelog"
```

Document every hit in `documentation_integrity_findings_v2026.4.8.md`.

- [ ] **Step 9.2: Check for broken cross-references**

```bash
cd ~/src/claw/hermes-agent-docs
# Find all internal links in markdown files
grep -rh "\[.*\](\.\./\|\./" --include="*.md" . | grep -oP '\(\.\.?/[^)]+\)' | sort -u
```

Verify linked files exist. Document broken links found.

- [ ] **Step 9.3: Check that all new files are referenced**

```bash
cd ~/src/claw/hermes-agent-docs
for f in logging.md live-model-switching.md notify-on-complete.md; do
  echo "=== $f ===" && grep -rl "$f" --include="*.md" . | head -5
done
```

If any new file is unreferenced, add a link from the most relevant existing doc.

- [ ] **Step 9.4: Verify new docs exist**

```bash
ls ~/src/claw/hermes-agent-docs/logging.md \
   ~/src/claw/hermes-agent-docs/live-model-switching.md \
   ~/src/claw/hermes-agent-docs/notify-on-complete.md \
   ~/src/claw/hermes-agent-docs/review_plan_v2026.4.8.md \
   ~/src/claw/hermes-agent-docs/release_evaluation_v2026.4.8.md \
   ~/src/claw/hermes-agent-docs/repo_review_v2026.4.8.md
```

All six files must exist before proceeding.

- [ ] **Step 9.5: Create `documentation_integrity_findings_v2026.4.8.md`**

Create `~/src/claw/hermes-agent-docs/documentation_integrity_findings_v2026.4.8.md`:

```markdown
# Documentation Integrity Findings — Hermes Agent v0.8.0 (2026-04-09)

## Audit scope

Release boundary: `v2026.4.8` (Hermes Agent v0.8.0, released 2026-04-08)
Previous release: `v2026.4.3` (v0.7.0)
Files reviewed: all markdown files in `~/src/claw/hermes-agent-docs`

## Findings

### Stale version framing corrected
[List each file and what was corrected, from Step 9.1 output]

### Broken cross-references corrected
[List from Step 9.2 output]

### New pages added
- `logging.md` — centralized logging system (new in v0.8.0)
- `live-model-switching.md` — `/model` command (new in v0.8.0)
- `notify-on-complete.md` — background task notifications (new in v0.8.0)

### Features documented as released in v0.8.0
- Centralized logging to `~/.hermes/logs/` (agent.log + errors.log)
- Live model switching via `/model` command across all platforms
- `notify_on_complete` tool with process registry
- Google AI Studio (Gemini) native provider
- Supermemory memory provider
- MCP OAuth 2.1 PKCE + OSV malware scanning
- Matrix Tier 1 feature parity (reactions, E2EE, room management)
- Approval buttons on Slack & Telegram
- Inactivity-based agent timeout
- Config structure validation at startup
- Plugin session lifecycle hooks (`on_session_finalize`, `on_session_reset`)
- Self-optimized GPT/Codex tool-use guidance

### Items deferred (unreleased — in origin/main beyond v2026.4.8)
[List any features found in source that are NOT in v2026.4.8]

## Validation method

All facts verified against: released source at `v2026.4.8` tag, `RELEASE_v0.8.0.md`, GitHub release API. Public docs site (`hermes-agent.nousresearch.com/docs`) used only to confirm facts already present in released source.
```

- [ ] **Step 9.6: Update `README.md`**

Read the current README index, then:
1. Add entries for `logging.md`, `live-model-switching.md`, `notify-on-complete.md`
2. Update version references from v0.7.0 to v0.8.0
3. Add links to new audit files

```bash
cd ~/src/claw/hermes-agent-docs
grep -n "v0\.7\|v2026\.4\.3\|logging\|live-model\|notify-on-complete" README.md
```

- [ ] **Step 9.7: Update `changelog.md`**

Add v0.8.0 entry at the top of the changelog. Key sections: Core Agent, Providers, Memory, Gateway/Messaging (grouped by platform), CLI, Tools, MCP/Plugins, Security. Cross-reference `RELEASE_v0.8.0.md` for accurate PR numbers and descriptions.

- [ ] **Step 9.8: Final stale-version check**

```bash
cd ~/src/claw/hermes-agent-docs
grep -rn "v0\.7\.0\|v0\.6\.0\|v2026\.4\.3\b\|v2026\.3\." \
  --include="*.md" \
  --exclude-dir=docs \
  . 2>/dev/null | grep -v "review_plan\|release_eval\|repo_review\|documentation_integrity\|source_valid\|changelog\|RELEASE_"
```

Zero results expected (or only in historical audit files). Fix any remaining hits.

- [ ] **Step 9.9: Commit integrity findings and index updates**

```bash
cd ~/src/claw/hermes-agent-docs
git add documentation_integrity_findings_v2026.4.8.md README.md changelog.md contributing.md
git commit -m "docs: add v0.8.0 integrity findings; update README index and changelog"
```

- [ ] **Step 9.10: Push to GitHub**

```bash
cd ~/src/claw/hermes-agent-docs
git log --oneline origin/main..HEAD
```

Review the commits about to be pushed, then:

```bash
git push origin main
```

Verify push succeeded:

```bash
git log --oneline -5
git status
```

---

## Execution order

```
Tasks 1–8: run in parallel
Task 9: runs after Tasks 1–8 complete
```

Tasks 1–8 each commit independently. Task 9 makes the final integrity pass, then pushes all commits to GitHub in a single `git push`.
