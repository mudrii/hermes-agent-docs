# Documentation Integrity Findings (Hermes Agent v0.3.0 release docs)

## Scope

This file captures final integrity findings for `hermes-agent-docs` against released code base at `v2026.3.17`.

## Finding format

- `SEVERITY`: Critical / High / Medium / Low
- `AREA`: Streaming, Plugins, Provider routing, Gateway, Tooling, General
- `EVIDENCE`: exact source-path evidence and claim location
- `RISK`: user/operator impact
- `STATUS`: fixed / open / deferred

## Findings

1. High — Plugin manifest `requires_env` is over-claimed in docs
- `AREA`: Tools & Plugins
- `EVIDENCE`: `plugins.md` states missing env vars skip plugin loading. Runtime parses `requires_env` in manifests (`hermes_cli/plugins.py:237-249`) but does not enforce it before plugin import/`register()` (`hermes_cli/plugins.py:288-334`).
- `RISK`: Medium-to-high operational mismatch: users may assume a plugin is protected by env gates when it can still execute registration/import.
- `STATUS`: **Open** (documentation correction)
- `FILES`: `plugins.md`, `hermes_cli/plugins.py`

2. Medium — Plugin discovery mode is underspecified
- `AREA`: Tools & Plugins
- `EVIDENCE`: `plugins.md` presents project plugins as always enabled, but runtime scans `.hermes/plugins/` only when `HERMES_ENABLE_PROJECT_PLUGINS=1` (`hermes_cli/plugins.py:194-197`).
- `RISK`: Medium; users may be confused when `.hermes/plugins/` is ignored by default.
- `STATUS`: **Open** (documentation correction)
- `FILES`: `plugins.md`, `hermes_cli/plugins.py`

3. Medium — Streaming doc symbol drift
- `AREA`: Streaming
- `EVIDENCE`: `streaming.md` mentions `_streamed_msg_id`; runtime uses `already_sent` (`gateway/stream_consumer.py:68-77`, `gateway/run.py:2260-2266`, `gateway/run.py:5078-5079`).
- `RISK`: Medium; moderate doc-to-implementation drift for troubleshooting and onboarding.
- `STATUS`: **Open** (scheduled fix)
- `FILES`: `streaming.md`

4. Medium — API streaming endpoint mismatch
- `AREA`: Streaming / API
- `EVIDENCE`: `streaming.md` suggests `/v1/responses` streaming SSE (`response.output_text.delta`, `response.completed`), while implementation returns non-streaming JSON for `POST /v1/responses` in `gateway/platforms/api_server.py` and streams only `/v1/chat/completions`.
- `RISK`: Medium; integrators may build clients for unsupported streaming behavior.
- `STATUS`: **Open** (documentation correction)
- `FILES`: `streaming.md`, `gateway/platforms/api_server.py`

5. Medium — AGENTS process guidance drift
- `AREA`: Tooling / developer guidance
- `EVIDENCE`: `AGENTS.md` instruction to avoid `simple_term_menu` conflicts with implementation fallback usage in `hermes_cli/main.py` and `hermes_cli/auth.py`.
- `RISK`: Medium; contributor guidance and implementation choices can diverge.
- `STATUS`: **Open** (documentation clarification)
- `FILES`: `AGENTS.md`, `hermes_cli/main.py`, `hermes_cli/auth.py`

6. Low — Plugin command surface ambiguity
- `AREA`: Tools & Plugins
- `EVIDENCE`: `plugins.md` demonstrates slash-only workflow in one place, while runtime exposes shell `hermes plugins <...>` in `hermes_cli/main.py:3533-3570` and slash `/plugins` in `hermes_cli/commands.py:118`.
- `RISK`: Low; mainly operator usability and onboarding clarity.
- `STATUS`: **Open** (wording fix)
- `FILES`: `plugins.md`, `hermes_cli/main.py`, `hermes_cli/commands.py`

## Coverage cross-check matrix

- Verified docs vs runtime:
  - `streaming.md` -> `run_agent.py`, `gateway/run.py`, `gateway/stream_consumer.py`, `gateway/platforms/api_server.py`
  - `plugins.md` -> `hermes_cli/plugins.py`, `hermes_cli/plugins_cmd.py`, `hermes_cli/commands.py`
  - `changelog.md` -> `RELEASE_v0.2.0.md`, `RELEASE_v0.3.0.md`, `git diff` artifacts

## Acceptance statement

This release review identifies four active doc-code mismatches for release-capability documentation (plugin loading semantics and streaming endpoint behavior are currently the highest impact items). No critical runtime-breaking discrepancy was found in Hermes core behavior itself for `v2026.3.17`.
