# Review Plan: Hermes Agent v0.10.0 release-bound audit (2026-04-17)

## Scope and release boundary

This pass is restricted to the latest locally released Hermes Agent state present in `~/src/hermes/hermes-agent`:

- target release boundary: `v2026.4.16` / `Hermes Agent v0.10.0`
- previous release baseline: `v2026.4.13` / `Hermes Agent v0.9.0`
- only material in the `v2026.4.13..v2026.4.16` window is treated as normative for this pass

## Primary sources

- local source tags in `~/src/hermes/hermes-agent`
  - `v2026.4.13`
  - `v2026.4.16`
- release note: `~/src/hermes/hermes-agent/RELEASE_v0.10.0.md`
- release compare window:
  - `https://github.com/NousResearch/hermes-agent/compare/v2026.4.13...v2026.4.16`
- source docs delta from `website/docs`
- docs repo target: `~/src/hermes/hermes-agent-docs`

## Online validation

External validation is used only as a secondary surface:

- GitHub releases UI
- GitHub latest release API / release pages when available

If public GitHub release metadata differs from the local release artifacts, the discrepancy is recorded rather than silently normalized.

## Workstreams

1. Confirm the release boundary and tag metadata.
2. Enumerate source docs added or changed in `v2026.4.13..v2026.4.16`.
3. Map those source changes onto `hermes-agent-docs`.
4. Update docs repo version markers and changelog.
5. Add missing released docs pages for:
   - Tool Gateway
   - AWS Bedrock
   - QQ Bot
   - Dashboard plugins
6. Patch stale pages whose released behavior changed:
   - providers
   - tools
   - image generation
   - TTS
   - browser automation
   - messaging indexes
   - toolsets reference
7. Produce release-bound audit artifacts and commit/push the docs repo.

## Exit criteria

- docs repo version markers reflect `v0.10.0 (v2026.4.16)`
- changelog includes a `v0.10.0` section
- released source docs added in the `v2026.4.13..v2026.4.16` window exist or are represented in the docs repo
- the Tool Gateway and Bedrock surfaces are documented factually
- any public-release-surface mismatch is preserved in audit notes

