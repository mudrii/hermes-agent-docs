# Release Evaluation: Hermes Agent release-bound docs baseline as of 2026-04-29

## 1) Local release confirmation

- local release tag present in source repo: `v2026.4.23`
- release note present in source repo: `RELEASE_v0.11.0.md`
- target commit: `e77d44c31ec82597d96c5fe44db71a8ed059e3db`
- prior release tag: `v2026.4.16`

## 2) Public-release-surface status

The public release surface is checked against `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.23` during Task 14. Any divergence between the public release page and the local `RELEASE_v0.11.0.md` is recorded in `documentation_integrity_findings_v2026.4.23.md` rather than silently normalized.

The local tagged source tree plus `RELEASE_v0.11.0.md` are the authoritative release boundary used for docs synchronization in this pass.

## 3) Release-window scope

Window audited:

- `v2026.4.16..v2026.4.23`

Aggregate diff size:

- 1,219 commits
- 1,137 files changed
- +187,779 / −21,599 lines

Highest-impact released areas in this window:

- **Transport ABC** — formal abstract base under `agent/transports/` with `AnthropicTransport`, `ChatCompletionsTransport`, `ResponsesApiTransport`, `BedrockTransport`. Streaming, retries, prompt cache, and credential refresh remain on `AIAgent`.
- **TUI** — Ink-based frontend (`ui-tui/`) with a Python JSON-RPC backend (`tui_gateway/`); activated via `hermes --tui` or `HERMES_TUI=1`. Sticky composer, OSC-52 clipboard, status bar with stopwatch and git branch, light-theme preset, `/clear` confirm, subagent-spawn observability overlay.
- **New providers** — NVIDIA NIM, Arcee AI, Step Plan, Google Gemini CLI OAuth (`HERMES_GEMINI_PROJECT_ID`), Vercel ai-gateway native path, GPT-5.5 routed over Codex OAuth, Bedrock native (Converse API + IAM + inference profiles + Guardrails). xAI Grok upgraded to Responses API.
- **Plugin surface expansion** — `register_command`, `dispatch_tool`, `pre_tool_call` veto, `transform_tool_result`, `transform_terminal_output`, image_gen backend registration via `register_image_gen_provider`, namespaced skill bundles, shell-script hook bridge, dashboard plugin tabs.
- **`/steer`** mid-run nudge command (preserves prompt cache).
- **Webhook direct-delivery mode** — zero-LLM push payloads alongside the existing agent-routed mode.
- **Delegation orchestrator role** — configurable `delegation.max_spawn_depth` (1–3, default 1) plus `delegation.orchestrator_enabled` (default `true`, set `false` to force every parent into a leaf even when depth > 1).
- **QQBot** as the 17th platform — QQ Official API v2 with a QR setup wizard, emoji reactions, and DM/group policy gating.
- **Auxiliary models UI** screen for compression / vision / session_search / title_generation overrides; main-model-first auto-routing.
- **Cron** — `wakeAgent` gate (script-only jobs that never invoke the LLM) and per-job `enabled_toolsets`.
- **Skills changes** — new skills (`concept-diagrams`, `architecture-diagram`, `pixel-art`, `baoyu-comic`, `baoyu-infographic`, `page-agent`, `drug-discovery`, `touchdesigner-mcp`, `adversarial-ux-test`); `xitter` → `xurl` rename; `hermes skills reset` clears bundled-skill manifest tracking.
- **CLI flags** — `--ignore-user-config`, `--ignore-rules`. Doctor "Command Installation" check. Dynamic shell completion supports bash, zsh, **and fish**.
- **Live-switching dashboard themes** and dashboard plugin tabs.
- **Image generation** — pluggable backend surface; bundled provider plugins are `openai`, `xai`, `openai-codex`. FAL remains the legacy default served directly by `tools/image_generation_tool.py` and exposes many model variants (FLUX 2 Pro, Recraft V4 Pro, Nano Banana Pro, etc.) selectable per call.
- **Security** — 35 vendor secret-prefix patterns in `agent/redact.py` plus generic Authorization / JWT / DB-URL / URL userinfo / sensitive-query-string redaction; `HERMES_REDACT_SECRETS` snapshotted at import time so runtime mutation cannot disable the redactor mid-session.

## 4) Source/docs delta mapped in this pass

Added or newly represented in `hermes-agent-docs`:

- `tui.md` (new — Ink-based TUI guide)
- `built-in-plugins.md` (new — bundled plugin reference, e.g. `disk-cleanup`)
- `guides/github-pr-review-agent.md` (new)
- `guides/webhook-github-pr-review.md` (new)

Updated in this pass (claim-level fixes against v2026.4.23 source):

- `image-generation.md` — backend roster aligned with `plugins/image_gen/` (`fal`, `openai`, `xai`, `openai-codex`); fictitious `recraft` / `nano_banana_pro` / `gpt_image_2` / `xai_grok_imagine` / `openai_codex` backend names removed; config key corrected from `image_gen.backend` to `image_gen.provider`; plugin API call corrected from `register_image_gen` to `register_image_gen_provider`.
- `delegation.md` — `delegation.orchestrator_enabled` documented as defaulting to `true` (kill switch); blocked-toolset rationale updated.
- `credential-pools.md` — 402 / 429 cooldown corrected from "24 hours" to "1 hour" per `agent/credential_pool.py`.
- `reference/cli-commands.md` — `hermes completion` documents bash, zsh, and fish (was bash/zsh only); `hermes skills reset` row added.
- `reference/tools-reference.md` — total tool count corrected from 55 to 54; browser tool count corrected from 12 to 11; fictitious `browser_dialog` row removed.
- `reference/optional-skills-catalog.md` — expanded from 21 entries to all 59 optional skills shipped at v2026.4.23 (header now states 59 optional + 72 bundled = 131 total).
- `security.md` — secret-prefix table expanded from 22 rows to 35 vendor prefixes; "20+ patterns" claim refreshed; JWT, URL userinfo, and sensitive-query-string redaction documented.
- `api-server.md` — `X-Hermes-Session-Id` (request + response header) and `Idempotency-Key` (5-minute cache) documented at the endpoint level rather than only in the CORS section.
- `reference/environment-variables.md` — `HERMES_REDACT_SECRETS` added to the Agent Behavior section.

## 5) External links used for validation

- GitHub releases page: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.23`
- compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.16...v2026.4.23`
- README at tag: `https://github.com/NousResearch/hermes-agent/blob/v2026.4.23/README.md`

## 6) Outcome

After this pass, `hermes-agent-docs` is updated from a `v0.10.0` baseline to a `v0.11.0` release-bound baseline. The four highest-leakage drift items from the previous baseline (image_gen backend roster, registry-shadowing claims, credential-pool cooldown, orchestrator default) are corrected and traced to source; the new TUI, built-in-plugins, and PR-review guides are present; and the audit artifact set mirrors the structure of the prior `v2026.4.16` set.
