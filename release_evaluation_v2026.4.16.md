# Release Evaluation: Hermes Agent release-bound docs baseline as of 2026-04-17

## 1) Local release confirmation

- local release tag present in source repo: `v2026.4.16`
- release note present in source repo: `RELEASE_v0.10.0.md`
- target commit on local `main`: `1dd6b5d5fb94cac59e93388f9aeee6bc365b8f42`
- prior release tag: `v2026.4.13`

## 2) Public-release-surface status

Initially during the v2026.4.16 audit the public GitHub releases surface lagged behind the local tag. That lag has since resolved: v2026.4.16 / v0.10.0 is now the latest published release at https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.16 (verified 2026-04-17).

The local tagged source tree plus `RELEASE_v0.10.0.md` remain the authoritative release boundary used for docs synchronization, and the public surface is consistent with that boundary.

## 3) Release-window scope

Window audited:

- `v2026.4.13..v2026.4.16`

Highest-impact released areas in this window:

- Nous Tool Gateway for paid Nous Portal subscribers
- native AWS Bedrock provider support
- QQ Bot messaging adapter
- dashboard plugin extension surface
- web dashboard theme and plugin documentation
- a broad reliability and hardening pass across gateway, browser, CLI, and provider routing

## 4) Source/docs delta mapped in this pass

Added or newly represented in `hermes-agent-docs`:

- `tool-gateway.md`
- `guides/aws-bedrock.md`
- `dashboard-plugins.md`
- `messaging/qqbot.md`

Updated in this pass:

- `README.md`
- `changelog.md`
- `providers.md`
- `tools.md`
- `image-generation.md`
- `tts.md`
- `browser.md`
- `messaging/README.md`
- `reference/toolsets-reference.md`

## 5) External links used for validation

- GitHub releases page:
  - `https://github.com/NousResearch/hermes-agent/releases`
- compare view:
  - `https://github.com/NousResearch/hermes-agent/compare/v2026.4.13...v2026.4.16`

## 6) Outcome

After this pass, `hermes-agent-docs` is updated from a `v0.9.0` baseline to a `v0.10.0` release-bound baseline, with the main released documentation gaps from the `v2026.4.13..v2026.4.16` window filled in.

