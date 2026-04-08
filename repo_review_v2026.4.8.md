# Repository Review: Hermes Agent source and docs audit on 2026-04-09

## 1) Objective

Review the Hermes Agent source tree and the standalone docs repo in detail, but keep normative claims restricted to the latest published release: `v0.8.0` / `v2026.4.8`.

## 2) Release baseline

- Latest published release: `v2026.4.8`
- Previous release: `v2026.4.3`
- Current `origin/main`: 1 commit ahead of the latest release (editorial-only change to `RELEASE_v0.8.0.md`)
- Conclusion: `origin/main` is not substantively ahead of the release; all doc work is anchored at `v2026.4.8`

## 3) Top-level directory structure at v2026.4.8

The following top-level entries are present in the source tree at the `v2026.4.8` tag:

```
.github/
acp_adapter/
acp_registry/
agent/
assets/
batch_runner.py
cli-config.yaml.example
cli.py
cron/
datagen-config-examples/
docker/
docs/
environments/
flake.lock
flake.nix
gateway/
hermes
hermes_cli/
hermes_constants.py
hermes_logging.py
hermes_state.py
nix/
packaging/
plugins/
requirements.txt
run_agent.py
scripts/
skills/
tests/
tools/
trajectory_compressor.py
uv.lock
website/
AGENTS.md
CONTRIBUTING.md
Dockerfile
LICENSE
MANIFEST.in
pyproject.toml
README.md
RELEASE_v0.2.0.md … RELEASE_v0.8.0.md
```

`hermes_logging.py` is new in v0.8.0 (not present at `v2026.4.3`) and is the primary source for the centralized logging subsystem.

## 4) Highest-impact directories by file change count (v2026.4.3..v2026.4.8)

| Change count | Directory |
|---|---|
| 202 | `tests/` |
| 66 | `website/` |
| 47 | `tools/` |
| 31 | `skills/` |
| 30 | `hermes_cli/` |
| 20 | `gateway/` |
| 18 | `plugins/` |
| 18 | `agent/` |
| 5 | `environments/` |
| 5 | `acp_adapter/` |
| 3 | `scripts/` |
| 2 | `nix/` |
| 2 | `cron/` |

The `tests/` directory leads by a wide margin (202 files), reflecting the 209 merged PRs and 82 resolved issues that included comprehensive test coverage. `tools/` (47 files) and `hermes_cli/` (30 files) reflect the major feature additions: process registry / notify_on_complete, centralized logging, live model switching, and CLI improvements. `gateway/` (20 files) reflects inactivity-based timeouts, approval buttons, and Matrix Tier 1.

## 5) Source-tree inventory by subsystem

### Root entrypoints and packaging

- `run_agent.py` is the shared orchestration entrypoint used across CLI, gateway, ACP, cron, and other runtimes
- `cli.py` and `hermes_cli/` provide the user-facing CLI surface and operational commands
- `hermes_logging.py` is new in v0.8.0 — provides `setup_logging()` for centralized logging to `~/.hermes/logs/`
- `pyproject.toml` (version = "0.8.0"), `requirements.txt`, `Dockerfile`, `flake.nix`, and `packaging/` define installation and packaging surfaces
- Release notes live in `RELEASE_v0.2.0.md` through `RELEASE_v0.8.0.md`

### `agent/`

- Prompt assembly, routing, redaction, model metadata, context compression, and memory coordination live here
- v0.8.0 additions include self-optimized GPT/Codex tool-use guidance, thinking-only prefill continuation, smart thinking block signature management, and jittered retry backoff
- Coverage lives in `architecture.md`, `developer-guide/*`, `memory.md`, `credential-pools.md`, `security.md`, `developer-guide/agent-loop.md`, and `developer-guide/prompt-assembly.md`

### `hermes_cli/`

- CLI commands, auth flows, config migration, setup wizard, provider selection, profile handling, gateway management, and plugin registration live here
- v0.8.0 additions include the `/model` command implementation, `hermes logs` command, `hermes auth remove`, plugin CLI subcommand registration, and permanent command allowlist loading
- Coverage lives in `cli-reference.md`, `configuration.md`, `providers.md`, `profiles.md`, `reference/cli-commands.md`, and new `live-model-switching.md` and `logging.md`

### `gateway/`

- All messaging adapters and the OpenAI-compatible API server live here
- v0.8.0 adds inactivity-based timeouts, native approval buttons for Slack and Telegram, Matrix Tier 1 feature parity, Signal MEDIA: tag delivery, Mattermost file attachments, Feishu interactive approval cards, webhook `{__raw__}` template token
- Coverage lives in `gateway.md`, `api-server.md`, and `messaging/*`

### `cron/`

- Scheduled-job storage and execution model
- v0.8.0 changes: inactivity-based timeout (active tasks run indefinitely), delivery failure tracking in job status, MEDIA: tag extraction before delivery
- Coverage lives in `cron.md` and `developer-guide/cron-internals.md`

### `acp_adapter/` and `acp_registry/`

- ACP stdio server surface for editor integrations (VS Code, Zed, JetBrains)
- v0.8.0 adds Copilot ACP tool-call bridge
- Coverage lives in `acp.md` and supporting architecture docs

### `plugins/memory/`

- External memory-provider plugins: Honcho, OpenViking, Mem0, Hindsight, Holographic, RetainDB, ByteRover, and new Supermemory (v0.8.0)
- v0.8.0 changes: Supermemory added (multi-container, search_mode, identity template, env var override); Hindsight overhaul; mem0 API v2 compat with prefetch context fencing and secret redaction; RetainDB fixes
- Coverage lives in `memory.md`, `plugins.md`, `honcho.md`, and `developer-guide/memory-provider-plugin.md`

### `tools/`

- Core browser, file, terminal, messaging, memory, vision, code-execution, and RL tool surfaces
- `tools/process_registry.py` is new in v0.8.0 — implements the process registry for background task tracking and `notify_on_complete`
- v0.8.0 changes: `notify_on_complete` parameter on `terminal(background=true)`, save oversized tool results to file instead of truncation, configurable persistence thresholds, sandbox-aware tool result persistence
- Coverage lives in `tools.md`, `code-execution.md`, `reference/tools-reference.md`, and new `notify-on-complete.md`

### `skills/` and `optional-skills/`

- Large bundled skill catalog plus official optional skills
- v0.8.0 changes: bundled skills synced to all profiles during update, per-platform disabled skills respected in Telegram menu, skills registered as native Discord slash commands
- Coverage lives in `skills.md`, `reference/skills-catalog.md`, and `reference/optional-skills-catalog.md`

### `environments/`

- RL and benchmark environments for training and evaluation
- Background processes run through the active environment interface — Docker, Singularity, Modal, Daytona, SSH all supported
- Coverage lives in `environments.md` and `rl-training.md`

### `tests/`

- The largest change area by file count (202 files) — test coverage for agent loop, gateway, CLI, ACP, tools, cron, plugins, and skills
- The released test tree is evidence of subsystem maturity, but this audit did not attempt exhaustive semantic verification of every test assertion

### `website/docs/`

- Public docs-site source (66 files changed in this release window)
- Valuable as a validation source, but not release-pinned
- Used only after a claim was already confirmed in released code or release notes

## 6) Docs-repo review

### What was already strong

- Wide surface-area coverage across install, configuration, providers, gateway, API server, memory, security, tools, skills, and all messaging platforms
- Prior release-bounded audit artifacts for `v2026.3.12`, `v2026.3.17`, `v2026.3.30`, `v2026.4.3`, and `v2026.4.7` lines

### What needed updating

- No doc page existed for centralized logging (`hermes_logging.py`, `hermes logs` command)
- No doc page existed for live model switching (`/model` command)
- No doc page existed for background task notifications (`notify_on_complete` / process registry)
- `architecture.md`, `developer-guide/architecture.md`, and several platform pages still framed themselves around v0.7.0 or v0.6.0
- Memory provider pages did not yet include Supermemory
- `messaging/matrix.md` did not reflect Matrix Tier 1 (significant feature gap)
- `mcp.md` lacked MCP OAuth 2.1 PKCE and OSV malware scanning
- `plugins.md` and `hooks.md` lacked new plugin lifecycle hooks
- `security.md` did not document the v0.8.0 security hardening pass

### New pages added

| File | Subsystem |
|---|---|
| `logging.md` | Centralized logging (`hermes_logging.py`, `~/.hermes/logs/`, `hermes logs`) |
| `live-model-switching.md` | `/model` command, aggregator-aware resolution, platform pickers |
| `notify-on-complete.md` | `notify_on_complete` tool, process registry, background task flow |

## 7) Subsystem to docs coverage map

| Subsystem | Existing doc page(s) | Status at v0.8.0 |
|---|---|---|
| Agent loop / prompt assembly | `architecture.md`, `developer-guide/agent-loop.md`, `developer-guide/prompt-assembly.md` | Updated for v0.8.0 |
| Configuration & startup | `configuration.md` | Updated for config validation |
| Providers & model system | `providers.md`, `provider-routing.md`, `fallback-providers.md`, `credential-pools.md` | Updated for Google AI Studio, MiMo v2 Pro, xAI Grok, Z.AI |
| Live model switching | — | New page: `live-model-switching.md` |
| Centralized logging | — | New page: `logging.md` |
| Memory providers | `memory.md`, `honcho.md`, `plugins.md`, `developer-guide/memory-provider-plugin.md` | Updated for Supermemory, Hindsight, mem0 v2 |
| Sessions / persistence | `checkpoints.md`, `profiles.md`, `developer-guide/session-storage.md` | Updated for shared thread sessions, profile isolation |
| Gateway core | `gateway.md`, `developer-guide/gateway-internals.md` | Updated for inactivity timeout, approval buttons |
| All messaging platforms | `messaging/*` | Updated; matrix.md substantially expanded |
| CLI surface | `cli-reference.md`, `reference/cli-commands.md` | Updated for `hermes logs`, `hermes auth` |
| Tools | `tools.md`, `reference/tools-reference.md` | Updated for `notify_on_complete`, persistence thresholds |
| Background task notifications | — | New page: `notify-on-complete.md` |
| Skills | `skills.md`, `reference/skills-catalog.md`, `reference/optional-skills-catalog.md` | Updated for Discord native slash commands, per-platform disabled skills |
| MCP | `mcp.md`, `reference/mcp-config-reference.md` | Updated for OAuth 2.1 PKCE, OSV scanning |
| Plugins | `plugins.md`, `hooks.md` | Updated for CLI subcommands, lifecycle hooks |
| Security | `security.md` | Updated for v0.8.0 hardening pass |
| Cron | `cron.md`, `developer-guide/cron-internals.md` | Updated for inactivity timeout, delivery failure tracking |
| ACP | `acp.md` | Updated for Copilot tool-call bridge |
| Environments / RL | `environments.md`, `rl-training.md` | Reviewed; no changes needed |
| Vision / TTS / voice | `vision.md`, `tts.md`, `voice-mode.md` | Updated for vision auto-detect change, MiniMax TTS |
| API server | `api-server.md` | Reviewed; updated version framing |
| Browser | `browser.md` | Updated for browser paths with spaces fix |

## 8) Key excluded unreleased work

Excluded from release-facing docs because it landed after `v2026.4.8`:

- The only commit ahead of `v2026.4.8` is `ff6a86cb` — a minor editorial reorder of the `RELEASE_v0.8.0.md` file itself; it introduces no source behavior changes

## 9) Recommended next step

When the next stable release is published, rerun the same process with a new dated audit set. Promote any post-`v2026.4.8` features that are part of the new published release. The `origin/main` gap is currently minimal (one editorial commit), so the next release window will start cleanly from `v2026.4.8`.
