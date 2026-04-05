# Documentation Integrity Findings -- v2026.4.3

**Audit date:** April 5, 2026  
**Auditor:** Automated source review plus online fact-checking  
**Source repo:** `~/src/claw/hermes-agent/`  
**Docs repo:** `~/src/claw/hermes-agent-docs/`  
**Hermes version audited:** v0.7.0 (v2026.4.3)  
**Unreleased `main` status:** ahead of tag, explicitly excluded from release-facing claims

---

## Review Scope

- Full released-surface review of Hermes Agent at `v2026.4.3`
- Cross-reference of local docs against released source and release notes
- Online fact-checking against GitHub releases and the public Hermes docs site
- Explicit exclusion of post-release `main` commits from release documentation

## Summary

The docs repo had two kinds of drift at the start of this pass:

- **hard stale versioning** -- `README.md`, `changelog.md`, and `gateway.md` still framed `v0.6.0` as current
- **feature-specific drift** -- several important `v0.7.0` features were either missing or still described as future work

This pass corrected the major release-critical gaps and created a new source-validation trail for `v2026.4.3`.

## Critical Corrections

1. **Version surfaces corrected** -- `README.md`, `changelog.md`, and `gateway.md` now reflect the released `v0.7.0` line instead of `v0.6.0`.
2. **Memory provider docs corrected** -- `developer-guide/memory-provider-plugin.md` no longer describes released provider plugins as future work.
3. **API continuity documented** -- `api-server.md` and `messaging/open-webui.md` now cover `X-Hermes-Session-Id`, shared response/session continuity, and streaming tool-progress behavior.
4. **Browser docs corrected** -- `browser.md` now documents the released Camofox backend and the local-backend SSRF distinction.
5. **Platform docs corrected** -- Discord button approvals/reactions, WhatsApp group `require_mention`, and webhook progress suppression are now reflected locally.
6. **Security docs corrected** -- `security.md` now summarizes the v0.7.0 secret-exfiltration and protected-directory hardening.

## Public Docs Cross-Check

During this audit, the public Hermes docs site already covered some `v0.7.0` surfaces well:

- memory providers
- credential pools
- browser/Camofox
- API/Open WebUI tool-progress streaming

But several details were still absent or not clearly surfaced on the public docs pages reviewed:

- `X-Hermes-Session-Id`
- ACP client-provided MCP servers
- Discord button-based approvals / reactions config
- WhatsApp group `require_mention`
- webhook tool-progress suppression

Those gaps were treated as validation targets for this docs repo rather than reasons to downgrade source-backed release claims.

## Known Remaining Gaps

The following user-facing changes exist on `main` after `v2026.4.3` and were intentionally excluded from this release pass:

- `/model` command overhaul using `models.dev`
- live `/update` stream output and interactive prompt buttons
- Matrix `require_mention` and `auto_thread`
- post-release skills added after the tag

Those should be documented in the next release-bounded audit, not retroactively inserted into `v0.7.0`.

## Files Updated In This Pass

- `README.md`
- `changelog.md`
- `gateway.md`
- `browser.md`
- `developer-guide/memory-provider-plugin.md`
- `api-server.md`
- `messaging/open-webui.md`
- `messaging/discord.md`
- `messaging/whatsapp.md`
- `messaging/webhook.md`
- `acp.md`
- `security.md`
- `credential-pools.md`

## Validation Links

- GitHub release: https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.3
- Public browser docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/browser/
- Public memory providers docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers/
- Public credential pools docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/credential-pools/
- Public API server docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server/
- Public Open WebUI docs: https://hermes-agent.nousresearch.com/docs/user-guide/messaging/open-webui/
- Public ACP docs: https://hermes-agent.nousresearch.com/docs/user-guide/features/acp/
