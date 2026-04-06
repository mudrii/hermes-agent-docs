# Review Plan: Hermes Agent release-bound audit rerun (2026-04-07)

## Scope and release boundary

This review is restricted to released Hermes Agent artifacts as of April 7, 2026.

- Latest published release: `v2026.4.3` / Hermes Agent `v0.7.0`
- Previous release used for delta sizing: `v2026.3.30`
- Local `origin/main` is ahead of `v2026.4.3` and is treated as unreleased for documentation purposes

## Primary sources

- Local git tag `v2026.4.3` in `~/src/claw/hermes-agent`
- Local release note file `~/src/claw/hermes-agent/RELEASE_v0.7.0.md`
- GitHub latest-release API: `https://api.github.com/repos/NousResearch/hermes-agent/releases/latest`
- GitHub release page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.3`
- GitHub compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.3.30...v2026.4.3`

## Secondary validation sources

These were used only as supporting validation because the public docs site is current-branch, not release-pinned:

- `https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers/`
- `https://hermes-agent.nousresearch.com/docs/user-guide/features/browser/`
- `https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server/`
- `https://hermes-agent.nousresearch.com/docs/user-guide/messaging/open-webui/`
- `https://hermes-agent.nousresearch.com/docs/user-guide/features/acp/`

## Workstreams

### 1) Release control

- Fetch source and docs repos from `origin/main`
- Confirm the latest published release and tag commit IDs
- Measure `v2026.4.3..origin/main` so unreleased work is explicitly excluded

### 2) Source audit

- Review released source structure across `agent/`, `hermes_cli/`, `gateway/`, `cron/`, `acp_adapter/`, `plugins/memory/`, `tools/`, `skills/`, `optional-skills/`, `environments/`, and `tests/`
- Map major subsystems to existing docs coverage
- Identify code-level facts that are safe to document as released behavior

### 3) Docs audit

- Review `~/src/claw/hermes-agent-docs` for stale version framing, broken navigation, and contradictions
- Separate historical release notes from current release-scoped guidance
- Prefer corrections that improve integrity without inventing unreleased functionality

### 4) External validation

- Validate the stable release against GitHub release metadata
- Cross-check current docs-site pages only for facts that also exist in released source or release notes
- Record where the public site is ahead of the release boundary

### 5) Documentation corrections

- Rebuild `README.md` documentation index around files that actually exist
- Correct released-vs-planned status for memory-provider plugins
- Refresh stale `v0.6.0` framing on architecture, messaging overview, selected messaging pages, and tool reference
- Add a fresh April 7 audit set for reproducible review history

## Deliverables

This pass updates or adds:

1. `README.md`
2. `plugins.md`
3. `architecture.md`
4. `developer-guide/architecture.md`
5. `messaging/README.md`
6. `messaging/telegram.md`
7. `messaging/slack.md`
8. `messaging/email.md`
9. `messaging/signal.md`
10. `messaging/mattermost.md`
11. `reference/tools-reference.md`
12. `review_plan_v2026.4.7.md`
13. `repo_review_v2026.4.7.md`
14. `release_evaluation_v2026.4.7.md`
15. `documentation_integrity_findings_v2026.4.7.md`
16. `source_validation_matrix_v2026.4.7.md`

## Exit criteria

- Release-facing claims are backed by released code, release notes, or both
- Unreleased `origin/main` work is explicitly excluded from normative docs
- Broken README navigation is removed
- Stale version framing is corrected on the highest-traffic pages touched in this pass
- Validation links are preserved so the audit can be rerun later
