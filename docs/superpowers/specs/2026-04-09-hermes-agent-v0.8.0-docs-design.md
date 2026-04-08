# Design: Hermes Agent v0.8.0 Documentation Review & Update

**Date:** 2026-04-09  
**Release:** Hermes Agent v0.8.0 (v2026.4.8, released 2026-04-08)  
**Previous release:** v0.7.0 (v2026.4.3)  
**Docs repo:** `~/src/claw/hermes-agent-docs` → `https://github.com/mudrii/hermes-agent-docs`  
**Source repo:** `~/src/claw/hermes-agent` → `https://github.com/NousResearch/hermes-agent`

---

## 1. Scope

A release-bounded documentation audit and update pass covering all 84 existing documentation files plus new pages for subsystems introduced in v0.8.0. Every fact must be traceable to one of:

- Released source at tag `v2026.4.8`
- `RELEASE_v0.8.0.md` in the source repo
- GitHub release API confirming `draft: false`, `prerelease: false`

Nothing from `origin/main` beyond the `v2026.4.8` tag is documented as released behavior.

**Release window size:** `v2026.4.3..v2026.4.8` — 337 commits, 582 files changed, 76,163 insertions, 6,750 deletions.

---

## 2. Architecture: Parallel Domain Agents

Nine independent agents execute in parallel. Each agent owns a domain, reads the relevant source, validates against the release notes and the public docs site (for confirmation only), and writes or updates its assigned files. Agent 9 additionally cross-references the outputs of all other agents for integrity.

### Coordination rules

- Each agent is self-contained: it reads source, release notes, and existing docs independently.
- No agent invents unreleased features. If source is ambiguous, agents document what is confirmed and flag what is unclear.
- Prior version framing (`v0.7.0`, `v0.6.0`, older) is updated wherever found.
- The public docs site (`hermes-agent.nousresearch.com/docs`) is used only to confirm facts already present in released source — never as the primary source.
- All agents produce a short fact-check note at the top of changed files (in a comment or frontmatter) to assist Agent 9's integrity pass.

---

## 3. Domain Assignments

### Agent 1 — Release control & audit scaffolding
**Reads:** `v2026.4.8` tag, `RELEASE_v0.8.0.md`, GitHub release API  
**Creates:**
- `review_plan_v2026.4.8.md` — scope statement, primary sources, workstream breakdown
- `release_evaluation_v2026.4.8.md` — tag confirmation, delta sizing, main-after-release exclusion
- `repo_review_v2026.4.8.md` — source structure audit, subsystem → docs coverage map

### Agent 2 — Core architecture & configuration
**Reads:** `cli.py`, `run_agent.py`, `agent/`, `hermes_cli/`, config schema, `RELEASE_v0.8.0.md` §Setup  
**Updates:** `architecture.md`, `configuration.md`, `installation.md`, `getting-started/updating.md`, `getting-started/learning-path.md`, `getting-started/nix-setup.md`, `environments.md`  
**Developer guide updates:** `developer-guide/agent-loop.md`, `developer-guide/architecture.md`, `developer-guide/context-compression.md`, `developer-guide/prompt-assembly.md`  
**Creates:** `logging.md` — centralized logging to `~/.hermes/logs/`, `hermes logs` command, agent.log + errors.log, log levels

### Agent 3 — Providers & model system
**Reads:** `agent/`, provider modules, `RELEASE_v0.8.0.md` §Provider & Model Support  
**Updates:** `providers.md`, `provider-routing.md`, `fallback-providers.md`, `credential-pools.md`, `reference/environment-variables.md`  
**Developer guide updates:** `developer-guide/adding-providers.md`, `developer-guide/provider-runtime.md`  
**Creates:** `live-model-switching.md` — `/model` command, aggregator-aware resolution, interactive pickers on Telegram/Discord, cross-provider fallback

### Agent 4 — Memory, sessions & persistence
**Reads:** `plugins/memory/`, `RELEASE_v0.8.0.md` §Memory & Sessions  
**Updates:** `memory.md`, `honcho.md`, `checkpoints.md`, `profiles.md`, `delegation.md`  
**Developer guide updates:** `developer-guide/memory-provider-plugin.md`, `developer-guide/session-storage.md`, `developer-guide/trajectory-format.md`

### Agent 5 — Gateway & all messaging platforms
**Reads:** `gateway/`, platform adapter modules, `RELEASE_v0.8.0.md` §Messaging Platforms  
**Updates:** `gateway.md`, `messaging/README.md`, `messaging/telegram.md`, `messaging/discord.md`, `messaging/slack.md`, `messaging/matrix.md`, `messaging/signal.md`, `messaging/mattermost.md`, `messaging/feishu.md`, `messaging/email.md`, `messaging/webhook.md`, `messaging/dingtalk.md`, `messaging/wecom.md`, `messaging/homeassistant.md`, `messaging/open-webui.md`, `messaging/sms.md`, `messaging/whatsapp.md`  
**Developer guide updates:** `developer-guide/gateway-internals.md`

### Agent 6 — CLI, tools & skills
**Reads:** `cli.py`, `tools/`, `tools/process_registry.py`, skills system, `RELEASE_v0.8.0.md` §CLI and §Tools  
**Updates:** `cli-reference.md`, `tools.md`, `toolsets.md`, `skills.md`, `code-execution.md`, `batch-processing.md`, `streaming.md`  
**Developer guide updates:** `developer-guide/adding-tools.md`, `developer-guide/tools-runtime.md`, `developer-guide/extending-cli.md`, `developer-guide/creating-skills.md`  
**Reference updates:** `reference/cli-commands.md`, `reference/slash-commands.md`, `reference/tools-reference.md`, `reference/toolsets-reference.md`, `reference/skills-catalog.md`, `reference/optional-skills-catalog.md`, `reference/profile-commands.md`  
**Creates:** `notify-on-complete.md` — `notify_on_complete` tool, process registry, background task auto-notifications

### Agent 7 — MCP, plugins, security & cron
**Reads:** `cron/`, plugin system, `RELEASE_v0.8.0.md` §MCP, §Plugins, §Security, §Cron  
**Updates:** `mcp.md`, `plugins.md`, `security.md`, `cron.md`, `hooks.md`, `acp.md`  
**Developer guide updates:** `developer-guide/cron-internals.md`  
**Reference updates:** `reference/mcp-config-reference.md`, `reference/faq.md`

### Agent 8 — Media, vision, voice & API server
**Reads:** vision/TTS/voice modules, `gateway/run.py`, browser adapter, skins system  
**Updates:** `vision.md`, `tts.md`, `voice-mode.md`, `image-generation.md`, `api-server.md`, `browser.md`, `context-references.md`, `rl-training.md`, `skins.md`, `personality.md`

### Agent 9 — Integrity, README & changelog
**Runs after agents 1–8.** Uses `RELEASE_v0.8.0.md`, the source tag, and the updated docs files to cross-reference completeness and consistency.  
**Creates:** `documentation_integrity_findings_v2026.4.8.md` — stale version framing, broken cross-references, undocumented released features, contradictions between files  
**Updates:** `README.md` (index of all docs files, including new ones), `changelog.md`, `contributing.md`

---

## 4. New Files Created

| File | Content |
|---|---|
| `logging.md` | Centralized logging system (`~/.hermes/logs/`), `hermes logs` command, log levels |
| `live-model-switching.md` | `/model` command, provider+model switching, aggregator resolution, platform pickers |
| `notify-on-complete.md` | `notify_on_complete` tool, process registry, background task notification flow |
| `review_plan_v2026.4.8.md` | Audit scope, sources, workstream plan |
| `release_evaluation_v2026.4.8.md` | Tag confirmation, delta sizing, exclusion list |
| `repo_review_v2026.4.8.md` | Source structure map, coverage analysis |
| `documentation_integrity_findings_v2026.4.8.md` | Integrity audit results |

---

## 5. Constraints

- **Release boundary:** `v2026.4.8` tag only. `origin/main` is ahead by commits; those are not documented.
- **Fact-checking:** Every claim sourced to released code or `RELEASE_v0.8.0.md`. Online docs used for confirmation only.
- **No speculation:** If a feature's behavior is not clear from released source, document what is confirmed and note the uncertainty.
- **Version framing:** Remove or update all `v0.6.0`/`v0.7.0`/older framing found during the pass.
- **Consistency:** File cross-references must be valid (no broken links to files that don't exist).
- **Commit:** Single commit to `hermes-agent-docs` on `main`, pushed to `https://github.com/mudrii/hermes-agent-docs`.

---

## 6. Deliverables Checklist

- [ ] `review_plan_v2026.4.8.md`
- [ ] `release_evaluation_v2026.4.8.md`
- [ ] `repo_review_v2026.4.8.md`
- [ ] `logging.md` (new)
- [ ] `live-model-switching.md` (new)
- [ ] `notify-on-complete.md` (new)
- [ ] All 84 existing docs files reviewed and updated
- [ ] `documentation_integrity_findings_v2026.4.8.md`
- [ ] `README.md` updated with new files
- [ ] `changelog.md` updated for v0.8.0
- [ ] Single git commit + push to GitHub
