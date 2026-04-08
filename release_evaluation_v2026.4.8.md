# Release Evaluation: Hermes Agent stable release baseline as of 2026-04-09

## 1) Stable release confirmation

- GitHub latest-release API reports `v2026.4.8` as the latest published release
- Release name: `Hermes Agent v0.8.0 (v2026.4.8)`
- GitHub metadata confirms `draft: false` and `prerelease: false`
- Published timestamp from GitHub API: `2026-04-08T11:56:44Z`
- Local tag `v2026.4.8` resolves to commit `86960cdbb0148145890e2ee90b4e157fa899f6e1`
- Previous release tag `v2026.4.3` resolves to commit `abf1e98f6253f6984479fe03d1098173a9b065a7`

## 2) Release-window size

`v2026.4.3..v2026.4.8` touched:

- 337 commits
- 582 files changed
- 76,163 insertions
- 6,750 deletions

Highest-impact released areas (from RELEASE_v0.8.0.md):

- centralized logging and config validation at startup
- live model switching across all gateway platforms
- background task auto-notifications via `notify_on_complete` and process registry
- Google AI Studio (Gemini) native provider with models.dev integration
- self-optimized GPT/Codex tool-use guidance through automated behavioral benchmarking
- inactivity-based agent timeouts (active tasks never killed)
- approval buttons on Slack and Telegram (native platform buttons)
- MCP OAuth 2.1 PKCE + OSV malware scanning for MCP packages
- Matrix Tier 1 feature parity (reactions, read receipts, E2EE, room management)
- plugin system expansion (CLI subcommands, request-scoped API hooks, session lifecycle hooks)
- Supermemory external memory provider
- security hardening pass (SSRF, timing attacks, tar traversal, cross-session isolation)

## 3) Main-after-release exclusion

As of this audit run, local `origin/main` is ahead of the latest published release:

- 1 commit ahead of `v2026.4.8`
- 1 file changed
- 6 insertions
- 2 deletions

Notable unreleased commit explicitly excluded from release-facing docs:

- `ff6a86cb` — docs: update v0.8.0 highlights — notify_on_complete, MiMo v2 Pro, reorder

This commit touches only `RELEASE_v0.8.0.md` itself (a minor re-ordering and clarification in the release notes file). It does not introduce new source behavior and is excluded from normative documentation per the v2026.4.8 release boundary.

## 4) Online validation summary

Primary links:

- Latest release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- Release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.8`
- Compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.3...v2026.4.8`

Supporting public-docs links used only after source/release-note confirmation:

- `https://hermes-agent.nousresearch.com/docs/`

Important caveat: the public docs site is current-branch documentation, not release-pinned. It can confirm that a fact exists publicly, but it cannot by itself prove that the fact belonged to `v2026.4.8`.

## 5) Key new subsystems introduced in v0.8.0

The following subsystems are new in v0.8.0 and require new documentation pages:

| Subsystem | Source location | New doc page |
|---|---|---|
| Centralized logging | `hermes_logging.py`, `~/.hermes/logs/` | `logging.md` |
| Live model switching | `hermes_cli/`, gateway platform adapters | `live-model-switching.md` |
| Background task notifications | `tools/process_registry.py` | `notify-on-complete.md` |

The following subsystems received substantial updates in v0.8.0 and require significant doc updates on existing pages:

- Google AI Studio provider (`providers.md`)
- Supermemory memory provider (`memory.md`)
- MCP OAuth 2.1 PKCE (`mcp.md`)
- Matrix Tier 1 (`messaging/matrix.md`)
- Plugin session lifecycle hooks (`plugins.md`, `hooks.md`)
- Config structure validation at startup (`configuration.md`)
- Inactivity-based timeout (`gateway.md`, `cron.md`)
- Approval buttons (`messaging/telegram.md`, `messaging/slack.md`)

## 6) Documentation corrections applied in this pass

- `architecture.md` and `developer-guide/architecture.md` refreshed to v0.8.0 surface
- `configuration.md` updated for config structure validation and new env vars
- `providers.md` updated for Google AI Studio, MiMo v2 Pro, MiniMax, xAI Grok, Z.AI, Ollama Cloud
- `memory.md` updated for Supermemory, Hindsight overhaul, mem0 v2, RetainDB fixes
- `gateway.md` updated for inactivity-based timeout and approval buttons
- All messaging platform pages updated; `messaging/matrix.md` substantially expanded for Tier 1
- `cli-reference.md` and `reference/cli-commands.md` updated for `hermes logs`, `hermes auth`
- `mcp.md` updated for OAuth 2.1 PKCE and OSV malware scanning
- `plugins.md` and `hooks.md` updated for new plugin capabilities
- `security.md` updated for the v0.8.0 security hardening pass
- Three new pages created: `logging.md`, `live-model-switching.md`, `notify-on-complete.md`
- `README.md` updated index; `changelog.md` updated with v0.8.0 entry

## 7) Remaining constraints

- The single unreleased commit (`ff6a86cb`) ahead of `v2026.4.8` is editorial-only and does not affect source behavior documentation
- Public docs-site pages should continue to be treated as supplemental validation rather than the release boundary itself
- Any features found only in `origin/main` beyond `v2026.4.8` remain excluded until the next release
