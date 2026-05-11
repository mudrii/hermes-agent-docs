# Release Evaluation: Hermes Agent v0.13.0 / v2026.5.7

Date reviewed: 2026-05-11

## Release Boundary

- Current stable release: `v0.13.0`
- Release tag: `v2026.5.7`
- Release date: May 7, 2026
- Previous stable baseline: `v0.12.0` / `v2026.4.30`
- Local tag commit: `498bfc7bc12a937621b4215312049b1000726df3`
- Local `main` after rerun pull: `271883447e7b8a5b9bd95879aca71afadc87616f` (`v2026.5.7-446-g271883447`)
- Compare range: `v2026.4.30...v2026.5.7`

## Validation Sources

- Local source release note: `~/src/hermes/hermes-agent/RELEASE_v0.13.0.md`
- Local source tag: `git -C ~/src/hermes/hermes-agent rev-list -n 1 v2026.5.7`
- First-party website docs: `~/src/hermes/hermes-agent/website/docs/**`
- Source tests and implementation files for Kanban, plugin LLM access, browser supervisor, gateway/messaging, and i18n
- Online confirmation: GitHub Releases lists `Hermes Agent v0.13.0 (2026.5.7)` as latest at `https://github.com/NousResearch/hermes-agent/releases`

## Release Summary

The v0.13.0 release is the Tenacity release. The release note reports 864 commits, 588 merged PRs, 829 files changed, 128,366 insertions, and 282 closed issues since v0.12.0.

The stable documentation update is release-bounded. Post-tag commits on `main` are documented as current-main changes only when the standalone docs intentionally track current source behavior; they are not counted as v0.13.0 release claims.

## Documentation Impact

- Promoted `changelog.md`, `README.md`, and `index.md` to v0.13.0 / v2026.5.7.
- Added or refreshed first-party docs for durable Kanban, Kanban tutorial, `/goal`, web search/SearXNG, plugin LLM access, model-provider plugins, image-generation provider plugins, Google Chat, and profile distributions.
- Refreshed reference docs for environment variables, tools, toolsets, CLI commands, and slash commands from first-party source docs.
- Corrected source-doc omissions discovered during review: `HERMES_LANGUAGE`, Browserbase advanced environment variables, dashboard plugin API authentication, and CDP/Browserbase wording.
- Separately synced post-release current-main docs for LINE setup, `/handoff`, slash-command access control, Telegram native draft streaming, Kanban worker lanes, and computer-use pages without representing them as v0.13.0 release contents.

## Residual Risk

- The release window is large, so this pass prioritizes release-critical and first-party documented changes rather than proving every generated skill catalog entry by hand.
- Some first-party website docs were already stale on small details; those were corrected in this standalone docs repo when source/tests contradicted the copied page.
