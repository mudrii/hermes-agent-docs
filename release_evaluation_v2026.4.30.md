# Release Evaluation: Hermes Agent v0.12.0 (v2026.4.30)

## Release Boundary

- Source repo: `~/src/hermes/hermes-agent`
- Docs repo: `~/src/hermes/hermes-agent-docs`
- Release tag: `v2026.4.30`
- Release commit: `73bf3ab1b22314ed9dfecbb59242c03742fe72af`
- Current pulled source main: `95f395027f72c69f06bddcecb08da53cfd10c440`
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

## Residual Risk

The standalone docs repo is not the upstream Docusaurus tree, so imported pages may retain some `/docs/...` absolute links and upstream frontmatter conventions. The content is source-aligned, but a separate site-build/link-check pass remains useful if this repo has its own publishing pipeline.
