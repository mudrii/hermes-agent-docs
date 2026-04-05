# Review Plan: Hermes Agent v0.7.0 (v2026.4.3)

## Scope and release boundary

This review is restricted to released artifacts for the `v2026.4.3` release line:

- Release tag: `v2026.4.3`
- Release notes: `RELEASE_v0.7.0.md`
- Comparison window used for release delta sizing: `v2026.3.30...v2026.4.3`
- Unreleased `main` changes after the tag were explicitly excluded from normative documentation claims

## Source-truth baseline

Primary sources:

- Local git tag `v2026.4.3` in `~/src/claw/hermes-agent`
- Local release note file `~/src/claw/hermes-agent/RELEASE_v0.7.0.md`
- GitHub release metadata and release page for `v2026.4.3`

Secondary validation sources:

- Public docs site at `https://hermes-agent.nousresearch.com/docs/`
- Public GitHub release page and compare URL for `v2026.3.30...v2026.4.3`

Documentation target:

- `~/src/claw/hermes-agent-docs`

## Execution status

- Completed fetch and fast-forward of local `main` in both source and docs repositories
- Confirmed `v2026.4.3` tag commit and separated unreleased `main` commits from release-facing claims
- Completed source review across memory, credential pooling, browser, gateway/platforms, API server, ACP, and security surfaces
- Completed online fact checks against GitHub releases and the public Hermes docs site
- Completed release-bounded documentation corrections and new audit artifacts in `hermes-agent-docs`

## Detailed workstreams

### 1) Release boundary control

- Confirm tag commit IDs for `v2026.3.30` and `v2026.4.3`
- Confirm the release is published, non-draft, and non-prerelease
- Exclude post-tag `main` changes from release-facing claims

### 2) Core/runtime and memory review

Review released behavior in:

- `RELEASE_v0.7.0.md`
- `run_agent.py`
- `agent/memory_provider.py`
- `agent/memory_manager.py`
- `agent/credential_pool.py`

Validate corresponding docs in:

- `README.md`
- `credential-pools.md`
- `memory.md`
- `honcho.md`
- `developer-guide/memory-provider-plugin.md`

### 3) Gateway/platform and API review

Review released behavior in:

- `gateway/run.py`
- `gateway/platforms/api_server.py`
- `gateway/platforms/discord.py`
- `gateway/platforms/whatsapp.py`
- `gateway/platforms/slack.py`
- `gateway/platforms/webhook.py` behavior as documented by release notes and adapter code paths

Validate corresponding docs in:

- `gateway.md`
- `api-server.md`
- `messaging/open-webui.md`
- `messaging/discord.md`
- `messaging/whatsapp.md`
- `messaging/webhook.md`

### 4) Browser, ACP, and security review

Review released behavior in:

- `tools/browser_tool.py`
- `tools/browser_camofox.py`
- `acp_adapter/session.py`
- `tools/code_execution_tool.py`
- `tools/file_operations.py`
- `agent/context_references.py`

Validate corresponding docs in:

- `browser.md`
- `acp.md`
- `security.md`

### 5) Documentation integrity review

- Identify internal contradictions within `hermes-agent-docs`
- Cross-check local docs against public docs to distinguish already-covered release features from gaps specific to this docs repo
- Record open gaps and explicitly mark unreleased `main` behavior as excluded from release docs

## Deliverables

This run produces or updates:

1. `review_plan_v2026.4.3.md`
2. `release_evaluation_v2026.4.3.md`
3. `documentation_integrity_findings_v2026.4.3.md`
4. `source_validation_matrix_v2026.4.3.md`
5. Release-corrected user-facing docs pages

## Exit criteria

- All major `v0.7.0` release claims documented here are traced back to released code or release-note evidence
- Known drifts are either fixed or explicitly recorded as open
- Online validation links are preserved in the audit artifacts
- No unreleased `main` behavior is documented as released behavior
