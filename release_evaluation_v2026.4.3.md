# Release Evaluation: Hermes Agent v0.7.0 (v2026.4.3)

## 1) Scope

- Review scope is restricted to released artifacts: `v2026.3.30` -> `v2026.4.3`
- No unreleased `main` deltas are included as released behavior

## 2) Release validation

- Git tag `v2026.4.3` confirmed in the repository at commit `1f6efd6eb0ba6b573c73c96d14dff1be4454823d`
- Previous stable tag `v2026.3.30` confirmed at commit `8c4ca749ee70de2b750a25e368c47db69a280d3d`
- Release date: April 3, 2026 (confirmed via `RELEASE_v0.7.0.md` and GitHub releases page)
- `RELEASE_v0.7.0.md` present at `~/src/claw/hermes-agent/RELEASE_v0.7.0.md`
- Online confirmation: https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.3

## 3) Diff scope and complexity

`v2026.3.30..v2026.4.3` touched **452 files** with **45,954 insertions** and **7,926 deletions** (confirmed via `git diff --shortstat v2026.3.30 v2026.4.3` in `~/src/claw/hermes-agent`).

Highest-impact release areas from source and release notes:

- memory provider architecture
- same-provider credential pooling
- browser/Camofox runtime
- gateway hardening and approvals
- API server continuity and streaming
- security hardening

## 4) Main-after-release exclusion

Local `main` is ahead of the released tag. The following user-facing commits were present on `main` during this audit and were explicitly excluded from release-facing docs:

- `0c54da8a` -- live-stream `/update` output plus interactive prompt buttons
- `cb63b5f3` -- `popular-web-designs` skill
- `4976a8b0` -- `/model` overhaul using `models.dev` as primary database
- `d932980c` -- `gitnexus-explorer` optional skill

These commits are relevant for future docs work, but not for `v0.7.0` release documentation.

## 5) Documentation coverage matrix

| Feature | Release evidence | Docs touched in this run |
|---------|------------------|--------------------------|
| `v2026.4.3` is the current stable release | `RELEASE_v0.7.0.md`, git tag, GitHub release page | `README.md`, `changelog.md` |
| Memory providers are released, not planned | `agent/memory_provider.py`, `agent/memory_manager.py`, `plugins/memory/*` | `README.md`, `developer-guide/memory-provider-plugin.md` |
| Credential pools include `least_used`, preserved fallback state, and primary-runtime restoration | `agent/credential_pool.py`, `run_agent.py`, release notes | `README.md`, `credential-pools.md`, `changelog.md` |
| Camofox is a released browser backend | `tools/browser_tool.py`, `tools/browser_camofox.py`, release notes | `README.md`, `browser.md`, `changelog.md` |
| `/v1/chat/completions` supports `X-Hermes-Session-Id` | `gateway/platforms/api_server.py` | `README.md`, `api-server.md`, `messaging/open-webui.md`, `gateway.md` |
| Streaming API responses include tool progress | `gateway/platforms/api_server.py`, release notes | `api-server.md`, `messaging/open-webui.md`, `changelog.md` |
| ACP can accept client-provided MCP servers | `acp_adapter/server.py` and release notes, plus source/release evidence | `acp.md`, `changelog.md` |
| Discord uses button-based approvals and configurable reactions | `gateway/platforms/discord.py` | `messaging/discord.md`, `changelog.md` |
| WhatsApp supports `require_mention` in group chats | `gateway/platforms/whatsapp.py` | `messaging/whatsapp.md`, `changelog.md` |
| Webhook adapters suppress tool-progress chatter | release notes and gateway/platform behavior | `messaging/webhook.md`, `gateway.md`, `changelog.md` |
| Security now covers browser secret exfiltration and more protected dirs | `tools/browser_tool.py`, `tools/file_operations.py`, `agent/context_references.py`, `tools/code_execution_tool.py` | `security.md`, `changelog.md` |

## 6) Key source-verified corrections vs. prior docs

- `README.md` and `changelog.md` still presented `v0.6.0` as current stable; both were updated to `v0.7.0`
- `developer-guide/memory-provider-plugin.md` incorrectly described released provider plugins as future work; it was rewritten to reflect the shipped `MemoryProvider` architecture
- `api-server.md` and `messaging/open-webui.md` lacked `X-Hermes-Session-Id` continuity and streaming tool-progress coverage
- `browser.md` lacked the released Camofox backend and associated config/runtime behavior
- `messaging/discord.md` and `messaging/whatsapp.md` were missing released v0.7.0 platform behavior
- `security.md` did not summarize the v0.7.0 exfiltration/protected-directory changes

## 7) Remaining gaps / open questions

The following items remain intentionally out of scope for this release pass because they are post-release `main` changes, not part of `v0.7.0`:

- `/model` command overhaul and `models.dev` primary database
- live-stream `/update` output and interactive prompt buttons
- Matrix `require_mention` / `auto_thread`
- newly added optional skills after the release tag

These should be handled in the next release-bounded audit, not backported into `v0.7.0` docs.

## 8) Corrections applied in this run

The following user-facing documentation drifts were corrected directly in `hermes-agent-docs` during this audit:

- `README.md`
- `changelog.md`
- `gateway.md`
- `browser.md`
- `developer-guide/memory-provider-plugin.md`
- `api-server.md`
- `messaging/open-webui.md`
- `messaging/discord.md`
- `messaging/matrix.md`
- `messaging/whatsapp.md`
- `messaging/webhook.md`
- `acp.md`
- `configuration.md`
- `reference/environment-variables.md`
- `security.md`
- `credential-pools.md`
