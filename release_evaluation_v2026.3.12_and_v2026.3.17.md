# Release Evaluation: Hermes Agent (v2026.3.12 -> v2026.3.17)

## 1) Scope

- Review scope is restricted to released artifacts:
  - `v2026.3.12`
  - `v2026.3.17`

No unreleased `main` deltas are included.

## 2) Release validation

- Git tags confirm both boundaries in the repository.
- Upstream release metadata corroborates:
  - `v2026.3.12` date/notes from local `RELEASE_v0.2.0.md`
  - `v2026.3.17` date/notes from local `RELEASE_v0.3.0.md`
- External release tags and highlights were validated against the GitHub release page:
  - https://github.com/NousResearch/hermes-agent/releases/tag/v2026.3.17
  - https://github.com/NousResearch/hermes-agent/releases/tag/v2026.3.12

## 3) Diff scope and complexity

`v2026.3.12..v2026.3.17` touched **467 files** (confirmed via `git diff --stat v2026.3.12 v2026.3.17`), with high concentration in:

- `tests`: 186
- `website`: 76
- `tools`: 41
- `skills`: 40
- `hermes_cli`: 23
- `optional-skills`: 21
- `gateway`: 19
- `agent`: 12
- `acp_adapter`: 9

This is a large release surface; review is therefore evidence-backed by subsystem matrix plus high-sensitivity doc/code checks.

## 4) Runtime findings and behavior verification

### A. Provider/model routing and auth

Verified in:
- `hermes_cli/auth.py`
- `hermes_cli/models.py`
- `hermes_cli/main.py`
- `agent/smart_model_routing.py`
- `agent/auxiliary_client.py`
- `agent/anthropic_adapter.py`

Observed behaviors:
- provider/provider fallback logic is active and used by runtime invocation in gateway and CLI contexts.
- model setup and model selection workflows include direct `simple_term_menu` fallback path in multiple places.

### B. Streaming architecture

Verified in:
- `run_agent.py`
- `gateway/run.py`
- `gateway/stream_consumer.py`
- `gateway/platforms/api_server.py`

Observed behaviors:
- `run_agent.py` routes to `_interruptible_streaming_api_call()` when stream consumers are registered.
- `gateway/stream_consumer.py` provides async edit-based delivery for platforms.
- `gateway/run.py` stores `already_sent` outcome for stream finalization control.
- `/v1/responses` API streaming is not currently surfaced in the released implementation; SSE streaming is implemented on `/v1/chat/completions` with `stream: true`.

### C. Plugins and extension lifecycle

Verified in:
- `hermes_cli/plugins.py`
- `hermes_cli/plugins_cmd.py`
- `hermes_cli/main.py`
- `model_tools.py`

Observed behaviors:
- plugin discovery (user/project/pip)
- `hermes plugins <install|update|remove|list>` command surface is present
- runtime slash command registry exposes `/plugins` as Tools & Skills command
- `requires_env` in `plugin.yaml` is loaded as metadata but not currently used as a discovery-time gate for plugin `register()` calls.

### D. Browser and MCP

Verified in:
- `tools/browser_tool.py`
- `tools/browser_providers/*`
- `tools/mcp_tool.py`

Observed behaviors:
- browser CDP connect and session lifecycle path exists
- MCP discovery and tool registration logic remain present and covered by runtime docs path

## 5) Documentation integrity findings

### Severity: Medium

1) `streaming.md` has stale implementation symbol for streaming dedupe
- Claim in doc references `_streamed_msg_id`.
- Runtime now uses `already_sent` in stream-finalization flow (`gateway/run.py` and `gateway/stream_consumer.py`).
- Impact: moderate confusion for troubleshooting and contributor alignment.
 
### Severity: High

2) `plugins.md` overstates `requires_env` behavior
- Claim in doc describes plugin-level load skipping when required env vars are missing.
- Runtime currently parses `requires_env` but does not enforce it before `register()`; plugins are still imported unless their own tool registration/paths gate themselves.
- Impact: operators may assume plugin safety/gating behavior that is not enforced.

### Severity: Medium

3) `AGENTS.md` policy/implementation drift on interactive menu path
- AGENTS states “Do not use `simple_term_menu`” for interactive menus.
- Code uses `simple_term_menu` in multiple runtime paths in `hermes_cli/main.py` and `hermes_cli/auth.py` with fallback.
- This is a governance/doc instruction drift, not a runtime bug.

### Severity: Low

4) Plugin management command semantics are partially underspecified in docs
- `plugins.md` currently emphasizes `/plugins` without explicit split between slash command and top-level `hermes plugins ...` forms.
- Runtime provides both command surfaces; this can be clarified for operator precision.

## 6) No-fault findings

- `gateway/platforms/api_server.py` defines SSE handling for chat completions and a full non-streaming `Responses` API response path.
- `AGENTS.md` guidance drift is documentation-only; no runtime policy or security bug identified there.

## 7) Recommended remediation (priority order)

1. Update streaming docs: replace stale `_streamed_msg_id` references with `already_sent` and align dedupe wording.
2. Correct `plugins.md` for 2026.3.17 behavior:
   - Clarify `requires_env` is not plugin-level gating in this release.
   - Clarify project plugin discovery requires `HERMES_ENABLE_PROJECT_PLUGINS=1`.
   - Clarify command surfaces: shell `hermes plugins ...` and slash `/plugins`.
3. Decide whether API docs should state `/v1/responses` streaming behavior as non-streaming in this release, or align implementation and docs in code.
4. Add a short note in `AGENTS.md` explaining legacy guidance exception if `simple_term_menu` is retained for fallback compatibility.
