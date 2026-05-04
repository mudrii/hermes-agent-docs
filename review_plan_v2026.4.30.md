# Review Plan: Hermes Agent v0.12.0 release-bound audit (2026-05-04)

## Scope

- Target release boundary: `v2026.4.30` / `Hermes Agent v0.12.0`
- Previous released baseline: `v2026.4.23` / `Hermes Agent v0.11.0`
- Primary source artifact: `~/src/hermes/hermes-agent/RELEASE_v0.12.0.md`
- Compare window: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.23...v2026.4.30`
- Local source tag: `73bf3ab1b22314ed9dfecbb59242c03742fe72af`
- Local source main after pull: `95f395027f72c69f06bddcecb08da53cfd10c440`

Post-tag commits on `main` after `v2026.4.30` are reviewed only when they affect current docs integrity; release claims are bounded to the tag and release note.

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
8. Add v2026.4.30 audit artifacts.
9. Run text/link sanity checks and inspect the resulting diff.
10. Commit and push docs-only changes.

## Success Criteria

- `README.md`, `index.md`, and `changelog.md` identify v0.12.0 / v2026.4.30 as current stable.
- New v0.12 docs exist for Curator, Spotify, Teams, Yuanbao, model catalog/configuration, Azure Foundry, MiniMax OAuth, and browser supervisor.
- Reference tables include v0.12 tools, toolsets, CLI/slash commands, providers, environment variables, and generated skill catalogs.
- Audit artifacts mirror prior release-audit structure for `v2026.4.30`.
- Documentation changes are sourced to release notes, source files, or upstream first-party docs.
