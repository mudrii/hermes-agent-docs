# Release Evaluation: Hermes Agent stable release baseline as of 2026-04-14

## 1) Stable release confirmation

- GitHub latest-release API: [`v2026.4.13`](https://api.github.com/repos/NousResearch/hermes-agent/releases/latest)
- Release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.13`
- Release name: `Hermes Agent v0.9.0 (v2026.4.13)`
- Published timestamp: `2026-04-13T18:52:41Z`
- Tag commit: `1af2e18d408a9dcc2c61d6fc1eef5c6667f8e254`
- Latest prior release: `v2026.4.8` (`86960cdbb0148145890e2ee90b4e157fa899f6e1`)

## 2) Release-window size and diff boundary

- Commit window: `v2026.4.8..v2026.4.13`
- Commit count: **554**
- Source docs window (`website/docs`): **54 files changed**, **3089 insertions**, **195 deletions**
- Compare URL: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.8...v2026.4.13`

## 3) Main-after-release exclusion

- `git rev-list --count v2026.4.13..origin/main` currently resolves to **46**.
- Those commits are excluded because they are unreleased relative to this boundary.
- Only released material included in this docs pass is `<= v2026.4.13`.

## 4) External validation summary

Primary links used for provenance:

- Latest release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- Release notes: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.13`
- Source release metadata: `~/src/hermes/hermes-agent/RELEASE_v0.9.0.md`
- Release compare: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.8...v2026.4.13`

Secondary sources (applied only after local release evidence is confirmed):

- `https://hermes-agent.nousresearch.com/docs/` (public docs-site content used to confirm page intent when local source doc structure was ambiguous)

## 5) High-impact change areas in source for docs verification

Across the release window, docs updates are concentrated in:

- Developer internals (`developer-guide/*`, especially model routing, gateway internals, context compression, cron architecture)
- Getting started paths (`installation`, `quickstart`, `nix`, Termux)
- Messaging (`bluebubbles`, `weixin`, `wecom-callback`, webhook flow and index refresh)
- CLI and command surfaces (`cli-reference`, `cli-commands`, profile/faq/help references)
- Integrations (`providers`, provider catalog organization, integration docs hub)
- Core features (`web-dashboard`, `api-server`, cron, sessions, code execution, memory providers, tts)
- New user-facing pages for feature discovery and operational workflows

## 6) Notable release-backed docs changes implemented from this boundary

1. Added/updated onboarding and platform-specific pages to cover Termux, BlueBubbles, WeCom callback, and Weixin setup.
2. Added dedicated pages for integrations and sessions/web-dashboard/overview/memory provider surfacing in line with release features.
3. Updated CLI, configuration, model routing, and reference command pages for `/model`, `/fast`, provider flows, and diagnostics updates.
4. Updated gateway and messaging coverage for 16th platform surface and platform reliability behavior.
5. Refreshed security and reliability related pages (`reference`, `cron`, environment and tools) for changes explicitly present in source release artifacts.

## 7) Current state after this release-bound pass

- Repository metadata and docs index were updated to v0.9.0 (with explicit release boundary).
- Review artifacts for this cycle were added for traceability.
- Release-scoped pages now map to source changes from `v2026.4.8..v2026.4.13`.
