# Review Plan: Hermes Agent v0.12.0 release-bound audit (2026-05-04)

## Scope

- Target release boundary: `v2026.4.30` / `Hermes Agent v0.12.0`
- Previous released baseline: `v2026.4.23` / `Hermes Agent v0.11.0`
- Primary source artifact: `~/src/hermes/hermes-agent/RELEASE_v0.12.0.md`
- Compare window: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.23...v2026.4.30`
- Local source tag: `0c35092accdc4e306e982c7b1913bf97b9bb3d3d`
- Local source main after pull: `601e5f1d57cfd4ceefee50a6df05a860a1a602e8`

Post-tag commits on `main` after `v2026.4.30` are reviewed only when they correct documentation for behavior already released in `v0.12.0`. Release claims remain bounded to the tag and release note; unreleased post-tag features are excluded from the stable docs.

## Plan

1. Sync `hermes-agent` and `hermes-agent-docs` from `origin/main`.
2. Confirm latest released version from local tags, release notes, and public GitHub release search.
3. Review `RELEASE_v0.12.0.md` for release-critical documentation deltas.
4. Compare upstream first-party `website/docs/**` pages against standalone docs repo pages.
5. Split source review by subsystem with parallel read-only agents:
   - CLI, update, TUI, dashboard
   - providers, model runtime, auxiliary models, image routing
   - gateway, platforms, Teams, Yuanbao, media routing
   - skills, curator, plugins, Spotify, Google Meet
   - release surfaces, changelog, index, audit artifacts
6. Import first-party upstream v0.12 pages where paths map cleanly.
7. Patch standalone-only pages for release markers and stale factual claims.
8. Remove or roll back unreleased post-tag claims that were accidentally imported into stable docs.
9. Run text/link sanity checks and inspect the resulting diff.
10. Commit and push docs-only changes.

## Success Criteria

- `README.md`, `index.md`, and `changelog.md` identify v0.12.0 / v2026.4.30 as current stable.
- New v0.12 docs exist for Curator, Spotify, Teams, Yuanbao, model catalog/configuration, Azure Foundry, MiniMax OAuth, and browser supervisor.
- Reference tables include v0.12 tools, toolsets, CLI/slash commands, providers, environment variables, and generated skill catalogs.
- Audit artifacts mirror prior release-audit structure for `v2026.4.30`.
- Documentation changes are sourced to release notes, source files, or upstream first-party docs.
