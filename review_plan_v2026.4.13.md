# Review Plan: Hermes Agent v0.9.0 release-bound audit (2026-04-14)

## Scope and release boundary

This review is restricted to the latest released Hermes Agent state as of **v2026.4.13**.

- Latest published release: `v2026.4.13` / `Hermes Agent v0.9.0`
- Previous release baseline: `v2026.4.8` / `Hermes Agent v0.8.0`
- `origin/main` currently leads the latest release tag by **46 commits** and is excluded from normative documentation updates
- Only behavior and documentation in `main` at `v2026.4.13` is treated as authoritative for this pass

## Primary sources

- Local source tags in `~/src/hermes/hermes-agent`
  - `v2026.4.8` (previous release)
  - `v2026.4.13` (target release)
- Release note: `~/src/hermes/hermes-agent/RELEASE_v0.9.0.md`
- GitHub release metadata:
  - `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
  - `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.13`
- Release compare: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.8...v2026.4.13`
- Source docs diff: `git diff --name-only v2026.4.8..v2026.4.13 -- website/docs`
- Docs repo: `~/src/hermes/hermes-agent-docs`

## Constraints

- No unreleased (`origin/main` beyond `v2026.4.13`) claims
- No code assumptions from non-release source changes
- Preserve existing non-doc style conventions in the documentation repo unless directly updated for correctness
- Any `/docs/`-scoped links in source-migrated pages are treated as accepted source-site links unless they refer to missing local files

## Workstreams

### 1) Release boundary confirmation

1. Pull the latest source origin and verify release tag metadata
2. Count and list commits between `v2026.4.8` and `v2026.4.13`
3. Confirm unreleased lead (`v2026.4.13..origin/main`)
4. Record the official release summary links and artifact hashes

### 2) Source delta extraction and mapping

1. Enumerate exact files changed under `website/docs` in `v2026.4.8..v2026.4.13`.
2. Map each source path to its docs-repo path and classify as:
   - Added page
   - Updated page
   - Renamed/re-scoped page
3. Keep a claim matrix of source-to-docs coverage.

### 3) Full docs repo audit (repository-wide)

1. Validate all docs files touched by the delta for factual alignment:
   - 54 source-diff files
   - New pages and updated pages listed in section 2
2. Update all impacted files in docs repo with the release-bound surface only.
3. Update landing artifacts:
   - `README.md` current version line
   - `changelog.md` current stable section and insertion
4. Add or refresh the release artifacts required for audit traceability.

### 4) Release artifact generation

1. Create `review_plan_v2026.4.13.md`
2. Create `release_evaluation_v2026.4.13.md`
3. Create `repo_review_v2026.4.13.md`
4. Create `source_validation_matrix_v2026.4.13.md`

### 5) Integrity + finalization

1. Run a manual index sweep on modified/new docs paths to detect missing target files
2. Verify there are no unreleased-in-main claims
3. Stage all intended files, preserving prior release artifacts not related to this boundary
4. Commit and push to `origin/main`

## Detailed file coverage

### New docs from this release window

- `developer-guide/adding-platform-adapters.md`
- `developer-guide/context-compression-and-caching.md`
- `developer-guide/context-engine-plugin.md`
- `getting-started/installation.md`
- `getting-started/quickstart.md`
- `getting-started/termux.md`
- `guides/`
- `integrations/index.md`
- `integrations/providers.md`
- `memory-providers.md`
- `index.md`
- `messaging/bluebubbles.md`
- `messaging/index.md`
- `messaging/webhooks.md`
- `messaging/wecom-callback.md`
- `messaging/weixin.md`
- `overview.md`
- `sessions.md`
- `web-dashboard.md`

### Existing docs requiring revision from this release

- `api-server.md`
- `batch-processing.md`
- `cli-reference.md`
- `code-execution.md`
- `configuration.md`
- `cron.md`
- `developer-guide/agent-loop.md`
- `developer-guide/architecture.md`
- `developer-guide/context-compression.md`
- `developer-guide/cron-internals.md`
- `developer-guide/gateway-internals.md`
- `developer-guide/memory-provider-plugin.md`
- `fallback-providers.md`
- `messaging/README.md`
- `messaging/discord.md`
- `messaging/feishu.md`
- `messaging/matrix.md`
- `messaging/open-webui.md`
- `messaging/sms.md`
- `messaging/telegram.md`
- `messaging/webhook.md`
- `messaging/whatsapp.md`
- `plugins.md`
- `providers.md`
- `reference/cli-commands.md`
- `reference/environment-variables.md`
- `reference/faq.md`
- `reference/profile-commands.md`
- `reference/slash-commands.md`
- `reference/toolsets-reference.md`
- `skills.md`
- `tts.md`
- `getting-started/nix-setup.md`
- `installation.md`
- `memory.md`

## Exit criteria

- Scope anchored to `v2026.4.13` with no unreleased claims
- Every source doc delta item from `v2026.4.8..v2026.4.13` has corresponding updated or added docs
- Top-level repository metadata reflects v0.9.0 and has a new stable changelog entry
- New release artifacts are present and consistent with this plan
