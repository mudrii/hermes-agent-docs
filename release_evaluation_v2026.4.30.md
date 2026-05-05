# Release Evaluation: Hermes Agent v0.12.0 (v2026.4.30)

## Release Boundary

- Source repo: `~/src/hermes/hermes-agent`
- Docs repo: `~/src/hermes/hermes-agent-docs`
- Release tag: `v2026.4.30`
- Release commit: `0c35092accdc4e306e982c7b1913bf97b9bb3d3d`
- Current pulled source main: `601e5f1d57cfd4ceefee50a6df05a860a1a602e8`
- Release note: `RELEASE_v0.12.0.md`
- Compare: `v2026.4.23...v2026.4.30`

## Evaluation

The docs repo was previously aligned to `v0.11.0` / `v2026.4.23`. The v0.12.0 release adds new release-critical surfaces across Curator, providers, skills, plugins, messaging, TUI/dashboard, TTS, model catalogs, and CLI/update commands.

Updated in this pass:

- `changelog.md`, `README.md`, and `index.md` release framing.
- Feature pages for Curator, Spotify, built-in plugins, TTS/Piper, web dashboard, tools/toolsets, skills, and provider/fallback surfaces.
- Messaging pages for Yuanbao and Microsoft Teams plus messaging index updates.
- Provider and reference pages for GMI Cloud, Azure AI Foundry, LM Studio, MiniMax OAuth, Tencent Tokenhub, model catalog, CLI/slash commands, environment variables, tools, toolsets, and generated skills.
- Developer docs for browser supervisor and provider runtime corrections.
- ACP, vision, security, architecture, and context-compression factual patches.
- Follow-up corrections after pulling latest `main`: release-bound pages were checked against the `v2026.4.30` tag, and unreleased post-tag claims were removed from stable docs.

## Residual Risk

The standalone docs repo is not the upstream Docusaurus tree, so imported pages may retain some `/docs/...` absolute links and upstream frontmatter conventions. The content is source-aligned to `v2026.4.30`; post-tag source-main features remain out of scope until they are part of a release note or changelog.
