# Review Plan: Hermes Agent v0.8.0 release-bound audit (2026-04-09)

## Scope and release boundary

This review is restricted to released Hermes Agent artifacts as of April 9, 2026.

- Latest published release: `v2026.4.8` / Hermes Agent `v0.8.0`
- Previous release used for delta sizing: `v2026.4.3` / `v0.7.0`
- Local `origin/main` is ahead of `v2026.4.8` and is treated as unreleased for documentation purposes

## Primary sources

- Local git tag `v2026.4.8` in `~/src/claw/hermes-agent`
- Local release note file `~/src/claw/hermes-agent/RELEASE_v0.8.0.md`
- GitHub latest-release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- GitHub release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.8`
- GitHub compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.3...v2026.4.8`

## Secondary validation sources

These are used only after a claim is already confirmed in released source or release notes:

- `https://hermes-agent.nousresearch.com/docs/`

The public docs site is current-branch documentation, not release-pinned documentation. It can validate that a fact exists publicly, but it cannot by itself prove the fact belonged to `v2026.4.8`.

## Workstreams

### 1) Release control

- Fetch source and docs repos from `origin/main`
- Confirm the latest published release tag, metadata, and commit IDs
- Measure `v2026.4.3..v2026.4.8` delta to scope the work
- Identify unreleased `origin/main` commits to exclude

### 2) Source audit

- Review released source structure across `agent/`, `hermes_cli/`, `gateway/`, `cron/`, `acp_adapter/`, `plugins/memory/`, `tools/`, `skills/`, `optional-skills/`, `environments/`, and `tests/`
- Map major subsystems to existing docs coverage
- Identify new subsystems introduced in v0.8.0 with no existing doc page

### 3) Full docs audit

- Review all 84 existing documentation files for stale version framing, factual errors, and broken cross-references
- Update all files to reflect v0.8.0 released behavior
- Add new pages for three subsystems introduced in v0.8.0: centralized logging, live model switching, and background task notifications

### 4) External validation

- Validate the stable release against GitHub release API metadata
- Cross-check public docs-site pages only for facts that also exist in released source or release notes
- Record where the public site is ahead of the release boundary

### 5) Integrity pass

- Aggregate findings from all domain agents
- Produce `documentation_integrity_findings_v2026.4.8.md`
- Update `README.md` index and `changelog.md`

## Deliverables

New files created in this pass:

1. `logging.md` — centralized logging system (`~/.hermes/logs/`), `hermes logs` command, log levels
2. `live-model-switching.md` — `/model` command, provider+model switching, aggregator-aware resolution, platform pickers
3. `notify-on-complete.md` — `notify_on_complete` tool, process registry, background task auto-notifications
4. `review_plan_v2026.4.8.md`
5. `release_evaluation_v2026.4.8.md`
6. `repo_review_v2026.4.8.md`
7. `documentation_integrity_findings_v2026.4.8.md`

Updated files: all 84 existing documentation files reviewed; those with stale content or v0.7.0/v0.6.0 framing updated to v0.8.0.

Updated indexes and changelog:

- `README.md` — rebuilt index including new files
- `changelog.md` — v0.8.0 entry added at top

Note: `source_validation_matrix_v2026.4.8.md` is not produced in this pass. The subsystem-to-docs coverage map in `repo_review_v2026.4.8.md` section 7 serves the same role for this release cycle.

## Exit criteria

- Release-facing claims are backed by released code at `v2026.4.8`, `RELEASE_v0.8.0.md`, or both
- Unreleased `origin/main` work is explicitly excluded from normative docs
- New subsystems (centralized logging, live model switching, notify_on_complete/process registry) each have a dedicated page
- All 84 existing files reviewed; stale version framing corrected
- Validation links are preserved so the audit can be rerun later
