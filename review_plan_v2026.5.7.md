# Review Plan: Hermes Agent v0.13.0 release-bound docs sync (2026-05-10)

## Scope

- Target release boundary: `v2026.5.7` / `Hermes Agent v0.13.0`
- Previous released baseline: `v2026.4.30` / `Hermes Agent v0.12.0`
- Primary source artifact: `~/src/hermes/hermes-agent/RELEASE_v0.13.0.md`
- Compare window: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.30...v2026.5.7`
- Local source tag commit: `498bfc7bc12a937621b4215312049b1000726df3`
- Local source `main` after pull: `d4b26df8974bca7114fa4fbff83e4600c31230f9`
- Documentation target: `~/src/hermes/hermes-agent-docs`

Post-tag commits on `main` after `v2026.5.7` are reviewed only when they correct documentation for behavior already released in `v0.13.0`. Release claims remain bounded to the tag, release note, source files, tests, and first-party website docs.

## Assumptions

- `v2026.5.7` is the latest released tag present in the local source repository after `origin/main` sync.
- First-party docs under `~/src/hermes/hermes-agent/website/docs/**` are authoritative when they describe released behavior and match source/tests.
- The standalone docs repository intentionally preserves prior release-audit artifacts, so sync work should add/update documentation without deleting historical review files.

## Review Plan

1. Sync both repositories from `origin/main`.
   - Verify: `git status --short --branch`, `git fetch origin`, `git pull --ff-only origin main`.
2. Confirm the release boundary.
   - Verify: latest tag is `v2026.5.7`, `RELEASE_v0.13.0.md` exists, and `changelog.md` should promote v0.13.0 above v0.12.0.
3. Review release and changelog surfaces.
   - Source files: `RELEASE_v0.13.0.md`, `RELEASE_v0.12.0.md`, `README.md`, git tag/compare output.
   - Docs files: `changelog.md`, `README.md`, `index.md`, prior `release_evaluation_*`, `source_validation_matrix_*`, and `documentation_integrity_findings_*`.
   - Verify: stable version, release date, compare range, and headline claims agree.
4. Review messaging/platform documentation.
   - Source files: `plugins/platforms/line/**`, `gateway/platforms/**`, `gateway/platform_registry.py`, `gateway/config.py`, `tests/gateway/test_line_plugin.py`, first-party `website/docs/user-guide/messaging/**`.
   - Docs files: `messaging/**`, `user-guide/messaging/**`, platform references.
   - Verify: new LINE docs are present, platform-plugin behavior is described accurately, and allowlist/cron/media environment variables are documented.
5. Review plugin, dashboard, and kanban documentation.
   - Source files: `agent/plugin_llm.py`, `hermes_cli/plugins.py`, `plugins/kanban/**`, `tools/kanban_tools.py`, `hermes_cli/kanban_db.py`, dashboard plugin files, related tests.
   - Docs files: `plugins.md`, `built-in-plugins.md`, `extending-the-dashboard.md`, `developer-guide/plugin-llm-access.md`, `kanban.md`, `kanban-tutorial.md`.
   - Verify: `ctx.llm`, durable kanban, dashboard plugin/auth behavior, removed demo plugins, and kanban environment variables match source.
6. Review CLI, tools, runtime, browser, and i18n documentation.
   - Source files: `tools/browser_supervisor.py`, `tools/browser_tool.py`, `scripts/benchmark_browser_eval.py`, `agent/i18n.py`, `locales/**`, `web/src/i18n/**`, `hermes_cli/**`.
   - Docs files: `browser.md`, `web-search.md`, `reference/environment-variables.md`, CLI/slash/tool references, getting-started pages.
   - Verify: browser supervisor path, SearXNG/web-tool split, i18n support, and environment variables are documented.
7. Review provider and integration docs added or changed for v0.13.0.
   - Source files: provider plugin interfaces and related first-party docs.
   - Docs files: `developer-guide/model-provider-plugin.md`, `developer-guide/image-gen-provider-plugin.md`, provider/integration references.
   - Verify: plugin-provider claims are backed by released source and do not overstate post-tag work.
8. Create/update release audit artifacts for `v2026.5.7`.
   - Docs files: `release_evaluation_v2026.5.7.md`, `repo_review_v2026.5.7.md`, `source_validation_matrix_v2026.5.7.md`, `documentation_integrity_findings_v2026.5.7.md`.
   - Verify: every material doc change cites source evidence or first-party docs.
9. Run local integrity checks.
   - Verify: no `v0.12.0` current-stable claims remain; key new files exist; markdown links to local docs are not obviously broken; `git diff --check` passes.
10. Commit and push docs-only changes.
   - Verify: commit contains only `hermes-agent-docs` changes and push succeeds to `origin/main`.

## Review of Plan

- The plan is release-bounded: it uses `v2026.5.7` as the stable source of truth and treats newer `main` commits only as corrective evidence.
- The plan is file-oriented: each review area names both source and docs files so findings can be traced.
- The plan is intentionally additive for historical artifacts: it does not delete previous audit records from the docs repo.
- The plan uses parallel read-only agents for independent review slices, then applies a single integrated documentation patch to avoid conflicting edits.

## Success Criteria

- `changelog.md`, `README.md`, and `index.md` identify v0.13.0 / v2026.5.7 as the current stable release.
- v0.13.0 documentation exists for durable kanban, `/goal`, Checkpoints v2, Google Chat, LINE, SearXNG/web-tool split, plugin LLM access, provider plugins, i18n, and security changes where first-party docs/source support them.
- `reference/environment-variables.md` includes current LINE, kanban, OpenRouter cache, SearXNG, and relevant v0.13 variables.
- New v2026.5.7 audit artifacts exist and distinguish source-backed facts from gaps that require future documentation.
- `git diff --check` passes before commit.
