# Review Plan: Hermes Agent v0.6.0 (v2026.3.30)

## Scope and release boundary

This review is restricted to released artifacts for the `v2026.3.30` release line:

- Release tag: `v2026.3.30`
- Release notes: `RELEASE_v0.6.0.md`
- Comparison window used for release delta sizing: `v2026.3.28...v2026.3.30`
- Unreleased `main` changes after the tag were explicitly excluded from normative documentation claims

## Source-truth baseline

Primary sources:

- Local git tag `v2026.3.30` in `~/src/claw/hermes-agent`
- Local release note file `~/src/claw/hermes-agent/RELEASE_v0.6.0.md`
- GitHub release metadata and release page for `v2026.3.30`

Secondary validation sources:

- Public docs site at `https://hermes-agent.nousresearch.com/docs/`
- Public GitHub compare and release URLs for `v2026.3.17`, `v2026.3.23`, and `v2026.3.30`

Documentation target:

- `~/src/claw/hermes-agent-docs`

## Execution status

- Completed fetch and fast-forward of local `main` to match `origin/main`
- Confirmed current `main` is ahead of `v2026.3.30`; unreleased deltas were separated from release review
- Completed release-tag validation and diff sizing
- Completed parallel subsystem audits across core/runtime, gateway/platforms, and tools/plugins/security
- Completed online fact checks against GitHub release metadata and public docs
- Completed documentation corrections and audit artifact creation in `hermes-agent-docs`

## Detailed workstreams

### 1) Release boundary control

- Confirm tag commit IDs for `v2026.3.17`, `v2026.3.23`, `v2026.3.28`, and `v2026.3.30`
- Confirm the release is published, non-draft, and non-prerelease
- Exclude post-tag `main` changes from release-facing claims

### 2) Core/runtime review

Review released behavior in:

- `README.md`
- `pyproject.toml`
- `run_agent.py`
- `agent/context_compressor.py`
- `hermes_cli/main.py`
- `hermes_cli/config.py`
- `mcp_serve.py`

Validate corresponding docs in:

- `README.md`
- `architecture.md`
- `configuration.md`
- `profiles.md`
- `mcp.md`
- `installation.md`
- `cli-reference.md`

### 3) Gateway/platform review

Review released behavior in:

- `gateway/run.py`
- `gateway/config.py`
- `gateway/builtin_hooks/boot_md.py`
- `gateway/platforms/telegram.py`
- `gateway/platforms/slack.py`
- `gateway/platforms/feishu.py`
- `gateway/platforms/wecom.py`
- `gateway/platforms/api_server.py`

Validate corresponding docs in:

- `gateway.md`
- `security.md`
- `hooks.md`
- `messaging/telegram.md`
- `messaging/slack.md`
- `messaging/feishu.md`
- `messaging/wecom.md`
- `api-server.md`

### 4) Tools, plugins, MCP, and security review

Review released behavior in:

- `tools/approval.py`
- `hermes_cli/plugins.py`
- `gateway/platforms/api_server.py`

Validate corresponding docs in:

- `security.md`
- `plugins.md`
- `mcp.md`
- `tools.md`
- `changelog.md`

### 5) Documentation integrity review

- Identify internal contradictions within `hermes-agent-docs`
- Separate fixed issues from remaining ambiguities
- Add a durable claim-to-source matrix so future review passes can re-check the same claims quickly

## Deliverables

This run produces or updates:

1. `review_plan_v2026.3.30.md`
2. `release_evaluation_v2026.3.30.md`
3. `documentation_integrity_findings_v2026.3.30.md`
4. `source_validation_matrix_v2026.3.30.md`
5. Release-corrected user-facing docs pages

## Exit criteria

- All major release claims in `hermes-agent-docs` are traced back to released code or release-note evidence
- Known drifts are either fixed or explicitly recorded as open
- Online validation links are preserved in the audit artifacts
- No unreleased `main` behavior is documented as released behavior
