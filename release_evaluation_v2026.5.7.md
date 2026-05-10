# Release Evaluation: Hermes Agent v0.13.0 / v2026.5.7

Date reviewed: 2026-05-10

## Release Boundary

- Current stable release: `v0.13.0`
- Release tag: `v2026.5.7`
- Release date: May 7, 2026
- Previous stable baseline: `v0.12.0` / `v2026.4.30`
- Local tag commit: `498bfc7bc12a937621b4215312049b1000726df3`
- Local `main` after pull: `d4b26df8974bca7114fa4fbff83e4600c31230f9`
- Compare range: `v2026.4.30...v2026.5.7`

## Validation Sources

- Local source release note: `~/src/hermes/hermes-agent/RELEASE_v0.13.0.md`
- Local source tag: `git -C ~/src/hermes/hermes-agent rev-list -n 1 v2026.5.7`
- First-party website docs: `~/src/hermes/hermes-agent/website/docs/**`
- Source tests and implementation files for LINE, Kanban, plugin LLM access, browser supervisor, and i18n
- Online confirmation: GitHub Releases lists `Hermes Agent v0.13.0 (2026.5.7)` as latest at `https://github.com/NousResearch/hermes-agent/releases`

## Release Summary

The v0.13.0 release is the Tenacity release. The release note reports 864 commits, 588 merged PRs, 829 files changed, 128,366 insertions, and 282 closed issues since v0.12.0.

The stable documentation update is release-bounded. Post-tag commits on `main` were used only when they clarified or documented behavior already present in the released code.

## Documentation Impact

- Promoted `changelog.md`, `README.md`, and `index.md` to v0.13.0 / v2026.5.7.
- Added first-party docs for durable Kanban, Kanban tutorial, `/goal`, web search/SearXNG, plugin LLM access, model-provider plugins, image-generation provider plugins, LINE, Google Chat, Teams meeting pipeline, Microsoft Graph webhook setup, profile distributions, and computer use.
- Refreshed reference docs for environment variables, tools, toolsets, CLI commands, and slash commands from first-party source docs.
- Corrected source-doc omissions discovered during review: `HERMES_LANGUAGE`, Browserbase advanced environment variables, 16-language i18n config, dashboard plugin API authentication, and CDP/Browserbase wording.

## Residual Risk

- The release window is large, so this pass prioritizes release-critical and first-party documented changes rather than proving every generated skill catalog entry by hand.
- Some first-party website docs were already stale on small details; those were corrected in this standalone docs repo when source/tests contradicted the copied page.
