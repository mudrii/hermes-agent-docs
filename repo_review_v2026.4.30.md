# Repo Review: Hermes Agent Docs v2026.4.30 Sync

## Reviewed Inputs

- Source release note: `~/src/hermes/hermes-agent/RELEASE_v0.12.0.md`
- Source tag: `v2026.4.30`
- Upstream docs: `~/src/hermes/hermes-agent/website/docs/**`
- Standalone docs repo: `~/src/hermes/hermes-agent-docs`
- Parallel review findings from five subsystem agents

## Result

The docs repo has been advanced from a v0.11.0 baseline to a v0.12.0 baseline. The highest-impact drift was in release framing, Curator, providers, messaging platforms, generated skills catalogs, plugins, TTS, CLI/TUI/dashboard references, and security/image-routing details.

## Follow-Up Candidates

- Run the standalone publishing pipeline, if one exists, to catch absolute-link or frontmatter issues introduced by importing first-party Docusaurus docs.
- Decide whether old release-audit artifacts should remain root-level or move under `docs/superpowers/` in a future cleanup.
