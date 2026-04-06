# Documentation Integrity Findings: Hermes Agent audit rerun on 2026-04-07

**Source repo:** `~/src/claw/hermes-agent/`  
**Docs repo:** `~/src/claw/hermes-agent-docs/`  
**Released version audited:** `v0.7.0` / `v2026.4.3`  
**Current `origin/main` status:** ahead of the latest release and excluded from release-facing claims

## Summary

This rerun found that the docs repo already had broad coverage, but its integrity was weakened by three recurring problems:

1. Broken or imaginary navigation in `README.md`
2. Stale release framing left over from `v0.6.0`
3. Contradictions between release-correct pages and older unchanged pages

## Highest-signal findings

### 1) README navigation was not trustworthy

The main index linked many files that do not exist in this repo:

- `features/*`
- `getting-started/quickstart.md`
- `getting-started/first-config.md`
- `getting-started/gateway-setup.md`
- `developer-guide/adding-a-tool.md`
- `developer-guide/adding-a-provider.md`
- several other non-existent developer/reference pages

This was corrected by rebuilding the index around files that actually exist.

### 2) Released memory-provider plugins were still described as future work

`plugins.md` still described memory-provider plugins as unreleased/planned. That directly contradicted:

- `RELEASE_v0.7.0.md`
- the stable GitHub release
- the docs repo’s own `changelog.md`
- the existing `developer-guide/memory-provider-plugin.md`

This was corrected to reflect the released `v0.7.0` provider surface.

### 3) Multiple high-traffic pages still framed themselves as `v0.6.0`

Pages corrected in this pass:

- `architecture.md`
- `developer-guide/architecture.md`
- `messaging/README.md`
- `messaging/telegram.md`
- `messaging/slack.md`
- `messaging/email.md`
- `messaging/signal.md`
- `messaging/mattermost.md`
- `reference/tools-reference.md`

### 4) Fixed totals had gone stale

The docs repo still contained brittle fixed totals such as:

- “49 built-in tools”
- “52 registered tools across 20 toolsets”

Those counts drifted as the product evolved. They were replaced with wording that keeps the per-page reference authoritative without promising a frozen aggregate total.

## Corrections applied in this pass

- Rebuilt the `README.md` documentation index
- Added an explicit April 7, 2026 audit set
- Corrected plugin-memory release status
- Refreshed stale version framing on architecture and messaging overview pages
- Removed stale fixed-count language from the tool reference intro

## Remaining caveats

- Historical audit artifacts remain in the repo; they should be treated as point-in-time records, not current truth
- The public docs site is not release-pinned, so release-only claims must continue to anchor on tags, release notes, and released source
- Unreleased post-`v2026.4.3` work such as Discord native slash-command registration, RetainDB follow-ups, centralized logging, and `hermes logs` remains intentionally excluded from release docs

## Validation links

- Latest release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- Release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.3`
- Compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.3.30...v2026.4.3`
- Public Memory Providers page: `https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers/`
- Public Browser page: `https://hermes-agent.nousresearch.com/docs/user-guide/features/browser/`
- Public API Server page: `https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server/`
- Public Open WebUI page: `https://hermes-agent.nousresearch.com/docs/user-guide/messaging/open-webui/`
- Public ACP page: `https://hermes-agent.nousresearch.com/docs/user-guide/features/acp/`
