# Release Evaluation: Hermes Agent v0.6.0 (v2026.3.30)

## 1) Scope

- Review scope is restricted to released artifacts: `v2026.3.28` → `v2026.3.30`
- No unreleased `main` deltas are included

## 2) Release validation

- Git tag `v2026.3.30` confirmed in the repository
- Release date: March 30, 2026 (confirmed via RELEASE_v0.6.0.md and GitHub releases page)
- RELEASE_v0.6.0.md present at ~/src/claw/hermes-agent/RELEASE_v0.6.0.md
- Online confirmation: https://github.com/NousResearch/hermes-agent/releases/tag/v2026.3.30

GitHub API validation confirmed:
- Release name: "Hermes Agent v0.6.0 (v2026.3.30)"
- Published: 2026-03-30T15:30:14Z
- Author: @teknium1
- Not a draft, not a pre-release
- PR spot-check: #3681, #3795, #3813, #3799, #3847, #3903, #3880, #3648 all present in release body

Compare URL used in changelog.md — `https://github.com/NousResearch/hermes-agent/compare/v2026.3.28...v2026.3.30` — matches the "Full Changelog" link at the bottom of the GitHub release page exactly.

## 3) Diff scope and complexity

`v2026.3.28..v2026.3.30` touched **230 files** with **27,081 insertions** and **1,310 deletions** (confirmed via `git diff --stat v2026.3.28 v2026.3.30` in ~/src/claw/hermes-agent).

Directory breakdown (files changed):

| Directory | Files |
|-----------|-------|
| tests | 70 |
| website | 30 |
| tools | 26 |
| hermes_cli | 24 |
| gateway | 20 |
| skills | 11 |
| agent | 10 |
| optional-skills | 4 |
| scripts | 3 |
| honcho_integration | 2 |
| docker | 2 |
| acp_adapter | 2 |
| root-level files (run_agent.py, mcp_serve.py, cli.py, etc.) | ~16 |

Highest-impact additions: `mcp_serve.py` (+868 lines, new file), `run_agent.py` (+214 lines), `cli.py` (+286 lines), `RELEASE_v0.6.0.md` (+249 lines). Tests dominate file count (70/230) consistent with the 10+ CI fix PRs noted in the release.

## 4) Documentation coverage matrix

| Feature | PRs | Doc file |
|---------|-----|----------|
| Profiles | #3681 | profiles.md |
| MCP Server Mode | #3795 | mcp.md |
| Docker Container | #3668 | installation.md |
| Fallback Provider Chain | #3813 | configuration.md, providers.md |
| Feishu/Lark | #3799, #3817 | messaging/feishu.md |
| WeCom | #3847 | messaging/wecom.md |
| Slack Multi-Workspace | #3903 | messaging/slack.md |
| Telegram Webhook Mode | #3880 | messaging/telegram.md |
| Telegram Group Gating | #3870 | messaging/telegram.md |
| Exa Search Backend | #3648 | tools.md |
| Skills on Remote Backends | #3890 | tools.md |
| Credentials on Remote Backends | #3671 | tools.md |
| Plugin enable/disable | #3747 | plugins.md |
| Plugin inject_message | #3778 | plugins.md |
| External Skill Directories | #3678 | skills.md |
| New Skills (5) | #3827, #3834, #3742, #3797 | skills.md |
| Hardened dangerous command detection | #3872 | security.md |
| Secret redaction expansion | #3920 | security.md |
| Atomic config.yaml writes | #3800 | gateway.md |
| Cron delivery labels | #3860 | gateway.md |
| BOOT.md hook | #3733 | gateway.md |

## 5) Key source-verified corrections vs. initial plan

The following corrections were found and applied during execution (vs. the initial documentation plan):

- Profile subcommand is `hermes profile use` not `hermes profile switch` — the release notes say "switch with `hermes -p <name>`"; the CLI subcommand for switching named profiles is `hermes profile use`
- `hermes mcp serve` is stdio only — no --transport/--port flags; "Streamable HTTP" in the release notes refers to client-side transport (the `streamable_http_client` used when Hermes connects to external MCP servers), not a server-side flag
- WeCom uses `WECOM_BOT_ID` + `WECOM_SECRET` (not WECOM_CORP_ID etc.)
- Telegram group gating uses `TELEGRAM_REQUIRE_MENTION` boolean + `TELEGRAM_MENTION_PATTERNS` JSON array (not a single enum value)
- Approval timeout config key is `approvals.timeout` (not `security.approval_timeout_seconds`)
- `inject_message()` is synchronous, not async
- Docker port is 8642 (gateway API server default), not 8080; no EXPOSE directive in Dockerfile
- Profile directory layout: `~/.hermes/profiles/<name>/` (not `~/.hermes-profiles/<name>/`)

## 6) Outstanding gaps / open questions

The following v0.6.0 features from the release are not explicitly documented in this doc update (all are minor or internal):

- Discord enhancements (#3871, #3640, #3674) — messaging/discord.md was updated for general adapter pattern but no dedicated Discord-specific section for these fixes
- WhatsApp reliability fixes (#3818, #3830, #3931) — documented at the adapter level in messaging but no explicit callout of LID↔phone alias behavior
- Matrix MSC3245 native voice messages (#3877) — not covered; Matrix docs remain at connection/setup level
- Mattermost configurable mention behavior (#3664) — not covered in this pass
- ACP editor integration session management (#3675) — out of scope for this doc update; covered by separate acp.md history
- MCP dynamic tool discovery / `notifications/tools/list_changed` (#3812) — covered at the feature level in mcp.md but the notification mechanism is not described in detail
- Configurable approval timeouts (#3886) — correction of config key is noted above; the feature itself is documented in security.md
- Audio .aac format and retry logic (#3865, #3401), vision file rejection (#3845) — tool-level bug fixes; not individually called out in tools.md
- OpenClaw migration guide (#3864, #3900) — out of scope for hermes-agent-docs; separate migration doc in upstream repo
- All testing and CI fixes (#3848, #3721, #3936) — not user-facing; no documentation required
