# Review Plan: Hermes Agent v0.11.0 release-bound audit (2026-04-29)

## Scope and release boundary

This pass is restricted to the latest locally released Hermes Agent state present in `~/src/hermes/hermes-agent`:

- target release boundary: `v2026.4.23` / `Hermes Agent v0.11.0`
- previous release baseline: `v2026.4.16` / `Hermes Agent v0.10.0`
- only material in the `v2026.4.16..v2026.4.23` window is treated as normative for this pass

Any unreleased commits on `main` past `v2026.4.23` are explicitly out of scope.

## Primary sources

- local source tags in `~/src/hermes/hermes-agent`
  - `v2026.4.16`
  - `v2026.4.23`
- release note: `~/src/hermes/hermes-agent/RELEASE_v0.11.0.md`
- release compare window:
  - `https://github.com/NousResearch/hermes-agent/compare/v2026.4.16...v2026.4.23`
- source docs delta from `website/docs/**` at `v2026.4.23`
- docs repo target: `~/src/hermes/hermes-agent-docs`

## Online validation

External validation is used only as a secondary surface:

- GitHub releases UI for `v2026.4.23`
- GitHub latest release API / release pages when available

If public GitHub release metadata differs from the local release artifacts, the discrepancy is recorded in `documentation_integrity_findings_v2026.4.23.md` rather than silently normalized.

## Workstreams

The audit was decomposed into nine read-only subagent workstreams plus five edit/synthesis workstreams executed by the orchestrator:

**Read-only (parallel) — verify claims at `v2026.4.23`:**
1. Core agent + Transport ABC (`agent/transports/`, `/steer`, compressor, auto-prune)
2. Provider layer (NVIDIA NIM, Arcee AI, Step Plan, Gemini CLI OAuth, Vercel ai-gateway, GPT-5.5 over Codex OAuth, Bedrock Converse + IAM + inference profiles + Guardrails)
3. Plugin / skill / hook surface (`register_command`, `dispatch_tool`, `pre_tool_call`, `transform_tool_result`, `transform_terminal_output`, image_gen plugin backends, namespaced skill bundles, shell-script hooks, dashboard tabs, `hermes skills reset`)
4. CLI / TUI / slash commands (`hermes --tui`, `HERMES_TUI=1`, `tui_gateway`, `/steer`, `--ignore-user-config`, `--ignore-rules`, dynamic completion bash/zsh/fish, OSC-52 clipboard, `/clear` confirm)
5. Gateway / messaging adapters (QQBot v2 with QR setup wizard, webhook direct-delivery mode, PLATFORM_HINTS, 17-platform total)
6. Cron / delegation / curator (`wakeAgent` script-only gate, per-job `enabled_toolsets`, `delegation.max_spawn_depth` 1-3, `delegation.orchestrator_enabled` kill switch)
7. Tools / security / memory (35 vendor secret-prefix patterns, `HERMES_REDACT_SECRETS` snapshot semantics, credential pool 1-hour cooldowns, Honcho 5-tool surface)
8. API server / dashboard / voice (`X-Hermes-Session-Id`, `Idempotency-Key`, dashboard themes + plugin tabs, `stt.enabled`, push-to-talk)
9. Configuration / env vars / reference (`cfg_get()`, new env vars, profile token locks)

**Edit / synthesis (sequential) — apply findings to docs:**
10. Set up audit workspace and `.audit/` scratch
11. Author the four new content pages (`tui.md`, `built-in-plugins.md`, two PR-review guides)
12. Patch existing pages with v0.11.0 deltas (features, user-guide, messaging, reference, top-level)
13. Author this audit artifact set
14. Online cross-check + final integrity sweep (grep for stale `v0.10.0` / `v2026.4.16` markers)

## Exit criteria

- docs repo version markers reflect `v0.11.0 (v2026.4.23)`
- changelog includes a `v0.11.0` section that mirrors `RELEASE_v0.11.0.md`
- released source docs added in the `v2026.4.16..v2026.4.23` window are represented in the docs repo
- the Transport ABC, TUI, `/steer`, plugin lifecycle veto/transform surfaces, QQBot v2 setup, dashboard themes, and orchestrator-mode delegation are documented factually
- factual claims previously written to upstream docs but contradicted by source at `v2026.4.23` are corrected (image_gen backends, browser tool count, total tool count, optional skills count, credential-pool cooldown duration, orchestrator default, completion shells)
- any public-release-surface mismatch is preserved in `documentation_integrity_findings_v2026.4.23.md`
