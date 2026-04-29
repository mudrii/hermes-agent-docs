# Repo Review: Hermes Agent docs sync for v0.11.0

## Summary

The docs repo entered this pass at the `v0.10.0` (`v2026.4.16`) baseline. Against the released v0.11.0 source surface (`v2026.4.23`), gaps and drift were concentrated in seven subsystems:

1. **Transport layer** — no docs page described the new abstract base under `agent/transports/` or the four concrete transport classes that replaced the inline provider switches.
2. **TUI** — the Ink frontend + JSON-RPC `tui_gateway` backend (activated via `hermes --tui` or `HERMES_TUI=1`) was undocumented.
3. **Plugin / hook expansion** — `register_command`, `dispatch_tool`, `pre_tool_call` veto, `transform_tool_result`, `transform_terminal_output`, `register_image_gen_provider`, namespaced skill bundles, shell-script hook bridge, and dashboard plugin tabs were either undocumented or described with incorrect signatures.
4. **Image generation** — the docs claimed five separate backend plugins (`recraft`, `nano_banana_pro`, `gpt_image_2`, `xai_grok_imagine`, `openai_codex`) that do not exist in `plugins/image_gen/` at `v2026.4.23`. The actual provider plugins are `openai`, `xai`, `openai-codex`; FAL exposes Recraft V4 Pro and Nano Banana Pro as model variants, not as separate backends. The config key was also documented as `image_gen.backend` rather than `image_gen.provider`.
5. **Delegation orchestrator** — `delegation.orchestrator_enabled` was documented as defaulting to `false`; the actual default is `true` (it is a kill switch, not an opt-in).
6. **Credential pool cooldowns** — both 402 and 429 cooldowns were documented as "24 hours"; the actual value is 1 hour (`EXHAUSTED_TTL_DEFAULT_SECONDS = 60 * 60`).
7. **Reference catalogues** — total built-in tool count, browser sub-tool count, optional-skills count, and `hermes completion` shell list were all stale.

## Corrected gaps

### Added

- `tui.md` — Ink + JSON-RPC user guide, observability overlay, OSC-52 clipboard, status-bar features.
- `built-in-plugins.md` — bundled plugin reference covering `disk-cleanup` and the enable/disable flow.
- `guides/github-pr-review-agent.md` and `guides/webhook-github-pr-review.md` — adapted from the upstream guides at `v2026.4.23`.

### Updated (claim-level)

- `image-generation.md` — backend roster, config key, and registration API now match `plugins/image_gen/` and `tools/image_generation_tool.py` at `v2026.4.23`.
- `delegation.md` — `orchestrator_enabled` correctly described as defaulting to `true`; blocked-toolset rationale updated.
- `credential-pools.md` — 402/429 cooldowns corrected to 1 hour.
- `reference/cli-commands.md` — `hermes completion` lists bash, zsh, and fish; `hermes skills reset` added.
- `reference/tools-reference.md` — total tool count = 54, browser tool count = 11; fictitious `browser_dialog` row removed.
- `reference/optional-skills-catalog.md` — expanded from 21 entries to all 59 optional skills (totals: 59 optional + 72 bundled = 131).
- `security.md` — 35 vendor secret-prefix patterns documented; JWT / URL userinfo / sensitive-query-string redaction added; "20+ patterns" claim refreshed.
- `api-server.md` — `X-Hermes-Session-Id` and `Idempotency-Key` documented at the endpoint level.
- `reference/environment-variables.md` — `HERMES_REDACT_SECRETS` added.

### Updated (subsystem refresh against `RELEASE_v0.11.0.md`)

- `README.md`, `changelog.md` — version markers and v0.11.0 highlight bullets.
- `architecture.md`, `developer-guide/architecture.md`, `developer-guide/agent-loop.md`, `developer-guide/transport-layer.md` — Transport ABC + `/steer` flow.
- `plugins.md`, `hooks.md` — lifecycle hook list, `pre_tool_call` veto shape, `transform_terminal_output` kwargs, `register_command` and `dispatch_tool` surfaces.
- `messaging/qqbot.md`, `messaging/webhooks.md`, `messaging/index.md` — QQBot v2 setup wizard and direct-delivery webhook mode.
- `web-dashboard.md`, `dashboard-plugins.md` — six built-in themes, `dashboard.theme` config key, plugin-tab registration.
- `voice-mode.md`, `tts.md`, `messaging/open-webui.md` — `stt.enabled` flag and push-to-talk references confirmed against source.
- `configuration.md`, `profiles.md` — `delegation.max_spawn_depth`, `delegation.orchestrator_enabled`, `dashboard.theme`, `agent.api_max_retries`, `skills.guard_agent_created`, `stt.enabled`, `privacy.redact_pii`, `security.redact_secrets` all reflect source defaults at `v2026.4.23`.

## Residual notes

- Public release surface verification result is recorded in `documentation_integrity_findings_v2026.4.23.md`; any divergence between the GitHub release page and the local `RELEASE_v0.11.0.md` is preserved there rather than merged into normative product documentation.
- Items observed in source but absent from upstream `website/docs/**` at `v2026.4.23` are listed in the integrity findings as upstream documentation debt rather than docs-repo drift.
