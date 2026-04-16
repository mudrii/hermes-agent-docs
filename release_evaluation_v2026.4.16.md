# Release Evaluation: Hermes Agent release-bound docs baseline as of 2026-04-17

## 1) Local release confirmation

- local release tag present in source repo: `v2026.4.16`
- release note present in source repo: `RELEASE_v0.10.0.md`
- target commit on local `main`: `1dd6b5d5fb94cac59e93388f9aeee6bc365b8f42`
- prior release tag: `v2026.4.13`

## 2) Public-release-surface discrepancy

During this audit, the public GitHub releases surface still reported `v2026.4.13` / `v0.9.0` as the latest published release, while the local source repository contained a newer release tag and release artifact for `v2026.4.16` / `v0.10.0`.

This docs pass treats the local tagged source tree plus `RELEASE_v0.10.0.md` as the authoritative release boundary because that is the code and documentation source the docs repo is explicitly syncing against.

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

