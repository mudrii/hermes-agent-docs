# Release Evaluation: Hermes Agent stable release baseline as of 2026-04-07

## 1) Stable release confirmation

- GitHub latest-release API reports `v2026.4.3` as the latest published release
- Release name: `Hermes Agent v0.7.0 (v2026.4.3)`
- GitHub metadata confirms `draft: false` and `prerelease: false`
- Published timestamp from GitHub API: `2026-04-03T18:15:29Z`
- Local tag `v2026.4.3` resolves to commit `1f6efd6eb0ba6b573c73c96d14dff1be4454823d`
- Previous release tag `v2026.3.30` resolves to commit `8c4ca749ee70de2b750a25e368c47db69a280d3d`

## 2) Release-window size

`v2026.3.30..v2026.4.3` touched:

- 452 files changed
- 45,954 insertions
- 7,926 deletions

Highest-impact released areas:

- external memory-provider architecture
- same-provider credential pools
- Camofox browser backend
- API server continuity and tool-progress streaming
- ACP client-provided MCP servers
- gateway hardening and security tightening

## 3) Main-after-release exclusion

As of this audit rerun, local `origin/main` is ahead of the latest published release:

- 165 commits ahead of `v2026.4.3`
- 305 files changed
- 46,903 insertions
- 3,946 deletions

Notable unreleased commits explicitly excluded from release-facing docs:

- `6f1cb46d` — register `/queue`, `/background`, `/btw` as native Discord slash commands
- `ea8ec270` — RetainDB project defaulting and optional project handling
- `6df48602` — RetainDB API routes, write queue, dialectic, agent model, file tools
- `6c12999b` — Copilot ACP tool-call bridge
- `9c96f669` — centralized logging, instrumentation, `hermes logs`
- `38d84460` — MCP OAuth 2.1 PKCE client support
- `dce5f51c` — config structure validation at startup
- `9ca954a2` — Mem0 API v2 compatibility and related redaction work

These belong in the next release-bounded docs pass, not in `v0.7.0` release docs.

## 4) Online validation summary

Primary links:

- Latest release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- Release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.3`
- Compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.3.30...v2026.4.3`

Supporting public-docs links used only after source/release-note confirmation:

- Memory Providers: `https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers/`
- Browser Automation: `https://hermes-agent.nousresearch.com/docs/user-guide/features/browser/`
- API Server: `https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server/`
- Open WebUI: `https://hermes-agent.nousresearch.com/docs/user-guide/messaging/open-webui/`
- ACP: `https://hermes-agent.nousresearch.com/docs/user-guide/features/acp/`

Important caveat:

- The public docs site is current-branch documentation, not release-pinned documentation. It can validate that a fact exists publicly, but it cannot by itself prove that the fact belonged to `v2026.4.3`.

## 5) Documentation corrections applied in this pass

- `README.md` rebuilt around real files and current audit artifacts
- `plugins.md` corrected from “planned” to released memory-provider plugins
- `architecture.md` refreshed to the released v0.7.0 surface
- `developer-guide/architecture.md` refreshed to the released v0.7.0 surface
- `messaging/README.md` and selected platform pages updated to v0.7.0 framing
- `reference/tools-reference.md` stripped of stale fixed tool totals

## 6) Remaining constraints

- Some docs pages still include body text focused on earlier releases where no later release-specific changes were applied; that is acceptable if the page header now clearly frames the released surface
- Public docs-site pages should continue to be treated as supplemental validation rather than the release boundary itself
