# Repo Review: Hermes Agent docs sync for v0.10.0

## Summary

The docs repo was already fast-forwarded to a large `v0.9.0` sync pass. The remaining gaps against the current released source window were concentrated in four areas:

1. global version markers still pointed to `v0.9.0`
2. the changelog did not include `v0.10.0`
3. released pages for Tool Gateway, AWS Bedrock, dashboard plugins, and QQ Bot were missing
4. several existing docs still described pre-Tool-Gateway behavior for image generation, browser automation, TTS, and tool/provider routing

## Corrected gaps

- bumped the docs repo release markers from `v0.9.0` to `v0.10.0`
- added a `v0.10.0` changelog entry
- added new released docs pages:
  - `tool-gateway.md`
  - `guides/aws-bedrock.md`
  - `dashboard-plugins.md`
  - `messaging/qqbot.md`
- updated existing docs to reflect:
  - paid Nous Portal Tool Gateway routing
  - `use_gateway` behavior
  - Bedrock provider availability
  - QQ Bot messaging coverage

## Residual note

The public GitHub releases page still appeared to lag the local source tag state during the audit. That mismatch is documented in `release_evaluation_v2026.4.16.md` rather than merged into normative product documentation.

