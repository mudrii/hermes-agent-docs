# Repository Review: Hermes Agent source and docs sync for v2026.4.13

## 1) Objective

Perform a release-bound sync and review between:

- Source release: `v2026.4.13` (`Hermes Agent v0.9.0`)
- Documentation repo: `~/src/hermes/hermes-agent-docs`

The focus is on documenting stable-release behavior and excluding unreleased commits from `origin/main`.

## 2) Release baseline

- Latest published release: `v2026.4.13`
- Previous release: `v2026.4.8`
- Unreleased `origin/main` lead count: **46** commits
- Release window assessed: `v2026.4.8..v2026.4.13`

## 3) Source tree impact summary

From source docs only (`website/docs`), the release window touched 54 files with 3089 insertions and 195 deletions.

Top change areas by directory contribution:

- `developer-guide/`: new internals, context compression updates, platform integration guides
- `getting-started/`: onboarding, quickstart, Nix, Termux
- `guides/`: operational and migration guidance updates
- `integrations/`: provider/platform integration hub updates
- `reference/`: command/env/reference updates
- `user-guide/features/`: feature pages and new pages (`web-dashboard`, memory providers, sessions surface)
- `user-guide/messaging/`: significant adapter updates and new adapters (BlueBubbles, Weixin, WeCom callback)

## 4) Diff-based mapping: source to docs-repo paths

### Added in release

- `developer-guide/adding-platform-adapters.md` → new
- `developer-guide/context-engine-plugin.md` → new
- `getting-started/termux.md` → new
- `guides/cron-troubleshooting.md` → new
- `user-guide/features/web-dashboard.md` → `web-dashboard.md`
- `user-guide/messaging/bluebubbles.md` → `messaging/bluebubbles.md`
- `user-guide/messaging/wecom-callback.md` → `messaging/wecom-callback.md`
- `user-guide/messaging/weixin.md` → `messaging/weixin.md`

### Modified in release (tracked or synchronized)

- `developer-guide/agent-loop.md`
- `developer-guide/architecture.md`
- `developer-guide/context-compression-and-caching.md`
- `developer-guide/context-compression.md`
- `developer-guide/cron-internals.md`
- `developer-guide/gateway-internals.md`
- `developer-guide/memory-provider-plugin.md`
- `getting-started/installation.md`
- `getting-started/nix-setup.md`
- `getting-started/quickstart.md`
- `guides/automate-with-cron.md`
- `guides/build-a-hermes-plugin.md`
- `guides/delegation-patterns.md`
- `guides/local-llm-on-mac.md`
- `guides/migrate-from-openclaw.md`
- `guides/use-voice-mode-with-hermes.md`
- `integrations/index.md`
- `integrations/providers.md`
- `reference/cli-commands.md`
- `reference/environment-variables.md`
- `reference/faq.md`
- `reference/profile-commands.md`
- `reference/slash-commands.md`
- `reference/toolsets-reference.md`
- `user-guide/cli.md`
- `user-guide/configuration.md`
- `user-guide/features/api-server.md`
- `user-guide/features/batch-processing.md`
- `user-guide/features/code-execution.md`
- `user-guide/features/cron.md`
- `user-guide/features/fallback-providers.md`
- `user-guide/features/memory-providers.md`
- `user-guide/features/overview.md`
- `user-guide/features/plugins.md`
- `user-guide/features/skills.md`
- `user-guide/features/tts.md`
- `user-guide/messaging/discord.md`
- `user-guide/messaging/feishu.md`
- `user-guide/messaging/index.md`
- `user-guide/messaging/matrix.md`
- `user-guide/messaging/open-webui.md`
- `user-guide/messaging/sms.md`
- `user-guide/messaging/telegram.md`
- `user-guide/messaging/webhooks.md`
- `user-guide/messaging/whatsapp.md`
- `user-guide/sessions.md`

## 5) Coverage status in docs repo

### Newly added and synced files now present in docs repo

- `developer-guide/adding-platform-adapters.md`
- `developer-guide/context-compression-and-caching.md`
- `developer-guide/context-engine-plugin.md`
- `getting-started/installation.md`
- `getting-started/quickstart.md`
- `getting-started/termux.md`
- `guides/automate-with-cron.md`
- `guides/build-a-hermes-plugin.md`
- `guides/cron-troubleshooting.md`
- `guides/delegation-patterns.md`
- `guides/local-llm-on-mac.md`
- `guides/migrate-from-openclaw.md`
- `guides/use-voice-mode-with-hermes.md`
- `integrations/index.md`
- `integrations/providers.md`
- `index.md`
- `memory-providers.md`
- `messaging/bluebubbles.md`
- `messaging/index.md`
- `messaging/webhooks.md`
- `messaging/wecom-callback.md`
- `messaging/weixin.md`
- `overview.md`
- `sessions.md`
- `web-dashboard.md`

### Existing pages updated from release deltas

- `api-server.md`
- `batch-processing.md`
- `cli-reference.md`
- `code-execution.md`
- `configuration.md`
- `cron.md`
- `developer-guide/agent-loop.md`
- `developer-guide/architecture.md`
- `developer-guide/cron-internals.md`
- `developer-guide/gateway-internals.md`
- `developer-guide/memory-provider-plugin.md`
- `fallback-providers.md`
- `getting-started/nix-setup.md`
- `installation.md`
- `memory.md`
- `messaging/README.md`
- `messaging/discord.md`
- `messaging/feishu.md`
- `messaging/matrix.md`
- `messaging/open-webui.md`
- `messaging/sms.md`
- `messaging/telegram.md`
- `messaging/webhook.md`
- `messaging/whatsapp.md`
- `plugins.md`
- `providers.md`
- `reference/cli-commands.md`
- `reference/environment-variables.md`
- `reference/faq.md`
- `reference/profile-commands.md`
- `reference/slash-commands.md`
- `reference/toolsets-reference.md`
- `skills.md`
- `tts.md`

## 6) Risk and gap log

- Any path-mapped source links to `/docs/...` remain because they are inherited from the website source structure and were not used as a deterministic correctness signal during this boundary pass.
- Existing unreleased pages are still visible in repo history and references; only release-scoped claims were updated.

## 7) Next-step controls

Before the next release pass:

1. Re-run against `v2026.4.13..v2026.4.x`.
2. Extend this matrix with new feature pages and command/behavior deltas.
3. Fold any high-risk link integrity fixes into the next `repo_review` + `source_validation_matrix` pair.
