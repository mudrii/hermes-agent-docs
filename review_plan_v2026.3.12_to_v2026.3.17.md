# Review Plan: Hermes Agent v2026.3.12 -> v2026.3.17

## Scope and release boundary

This review covers **released** versions only:

- `v2026.3.12` (March 12, 2026)
- `v2026.3.17` (March 17, 2026)

No unreleased `main`-branch changes are included.

## Source truth baseline

- Repository tags in scope: `v2026.3.12`, `v2026.3.17`
- Release notes available locally in `RELEASE_v0.2.0.md`, `RELEASE_v0.3.0.md`
- Upstream release pages and diffs were used for fact checks:
  - https://github.com/NousResearch/hermes-agent/releases/tag/v2026.3.12
  - https://github.com/NousResearch/hermes-agent/releases/tag/v2026.3.17
- Documentation source in this pass: `hermes-agent-docs` only.

### Execution status (this run)

- Completed `git fetch` and confirmed local `main` is at origin.
- Completed full release diff context (`git diff --stat v2026.3.12 v2026.3.17`).
- Completed subsystem mapping for source and docs in the release window.
- Completed live validation for streaming and plugin command/model behaviors.
- Completed external fact-check pass against GitHub release pages.

## Review objective

Validate that the documentation in `hermes-agent-docs`:

1. Is factually aligned with released code behavior.
2. Covers major feature behavior introduced in `v2026.3.17`.
3. Is internally consistent with `v2026.3.12`/`v2026.3.17` release notes and in-code semantics.
4. Does not contain stale or conflicting operational guidance.

## Detailed workstream plan

### 1) Release boundary and delta control

- Compare release tags and `v2026.3.12...v2026.3.17` diff in Git.
- Confirm changed file count and directory distribution.
- Build a subsystem delta matrix (agent, provider/router, gateway, tools, MCP, plugins, docs/site).

### 2) Subsystem-level code review (in release diff)

- Core runtime:
  - `run_agent.py`, `agent/*`, `cli.py`, `hermes_cli/main.py`
- Provider routing and auth:
  - `hermes_cli/auth.py`, `agent/smart_model_routing.py`, `agent/auxiliary_client.py`, `agent/anthropic_adapter.py`
- Gateway + stream path:
  - `gateway/run.py`, `gateway/stream_consumer.py`, `gateway/platforms/*`
- Tools and tool registration:
  - `tools/*`, `toolsets.py`, `model_tools.py`
- MCP:
  - `tools/mcp_tool.py`
- Plugins and plugin-manager:
  - `hermes_cli/plugins.py`, `hermes_cli/plugins_cmd.py`, `model_tools.py`
- Browser/CVP flow:
  - `tools/browser_tool.py`, `tools/browser_providers/*`
- Docs/site for release claims:
  - `website/docs/*`, `docs/*`, and corresponding sidebar entries.

### 3) Documentation parity checks

For each major doc in scope (`README`, `changelog`, `architecture`, `streaming`, `plugins`, `provider*`, `mcp`, `gateway`, `tools`, `toolsets`, `configuration`):

- Link each normative claim to a source implementation path.
- Flag stale paths, wrong symbol names, deprecated config names, or missing behavior.
- Add severity (`critical`, `high`, `medium`, `low`) and fix priority.

### 4) Command behavior and command-surface audit

- Confirm command registrations in `hermes_cli/commands.py`.
- Cross-check CLI docs (`cli-reference.md`, `plugins.md`) for command syntax fidelity:
  - slash commands in interactive mode
  - top-level `hermes <command>` equivalents

### 5) Release-note completeness audit

- For each major `RELEASE_v0.3.0.md` claim, verify implementation presence in code.
- Mark: fully-covered, partially-covered, or missing.

Suggested triage levels in findings:

- **Critical:** runtime-breaking mismatch between docs and released behavior.
- **High:** incorrect operator expectation with likely production confusion.
- **Medium:** meaningful behavioral mismatch needing docs correction.
- **Low:** command-surface/wording clarity.

## Deliverable structure in `hermes-agent-docs`

1. `review_plan_v2026.3.12_to_v2026.3.17.md`
2. `release_evaluation_v2026.3.12_and_v2026.3.17.md`
3. `documentation_integrity_findings_v2026.3.17.md`

## Exit criteria for this review

- All docs are reconciled against code for the release range.
- Known drifts are explicit and reproducible.
- Remediation recommendations include file-level owners and precise patch targets.
