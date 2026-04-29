# Documentation Integrity Findings: Hermes Agent v0.11.0 (v2026.4.23)

**Audit date:** 2026-04-29
**Target release:** `v2026.4.23` / `Hermes Agent v0.11.0` (commit `e77d44c31ec82597d96c5fe44db71a8ed059e3db`)
**Previous baseline:** `v2026.4.16` / `Hermes Agent v0.10.0`
**Scope:** Deep factual validation of `hermes-agent-docs` against the `v2026.4.23` source tag, following the v0.11.0 sync pass.

## Method

- Authoritative source surface read only via `git show v2026.4.23:<path>` to avoid contamination from post-release commits on `origin/main`.
- Nine parallel explorer subagents covered: core agent + transports, providers, plugins/skills/hooks, CLI/TUI/slash, gateway/messaging, cron/delegation/curator, tools/security/memory, API server/dashboard/voice, configuration/env vars/reference.
- Public release status re-verified against `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.23` during Task 14.

## Findings applied

| # | Severity | File(s) | Issue | Evidence at v2026.4.23 |
|---|----------|---------|-------|------------------------|
| 1 | Critical | `image-generation.md` | Backend roster invented five plugins that do not exist (`recraft`, `nano_banana_pro`, `gpt_image_2`, `xai_grok_imagine`, `openai_codex`) | `plugins/image_gen/` contains only `openai/`, `xai/`, `openai-codex/`; FAL is served by `tools/image_generation_tool.py` directly |
| 2 | Critical | `image-generation.md` | Config key documented as `image_gen.backend`; plugin API documented as `register_image_gen` | `agent/image_gen_provider.py` reads `image_gen.provider`; the registration entry point is `register_image_gen_provider(name, handler, env_var, plugin_id)` |
| 3 | High | `delegation.md` | `delegation.orchestrator_enabled` documented as defaulting to `false` (opt-in) | `hermes_cli/config.py:749` defaults to `True`; the flag is a kill switch, not an opt-in |
| 4 | High | `credential-pools.md` | 402 / 429 cooldown documented as "24 hours" | `agent/credential_pool.py` `EXHAUSTED_TTL_DEFAULT_SECONDS = 60 * 60` (1 hour) |
| 5 | High | `reference/tools-reference.md` | Total tool count claimed 55; browser sub-tool count claimed 12 with a `browser_dialog` row | `tools/registry.py` registers 54 tools at v2026.4.23; `tools/browser_tools/*.py` exposes 11 sub-tools (no `browser_dialog`) |
| 6 | High | `reference/optional-skills-catalog.md` | Catalogued only 21 of 59 optional skills; header claimed pre-v0.11.0 totals | All 59 SKILL.md frontmatter blocks dumped from `git show v2026.4.23:skills/...` |
| 7 | High | `reference/cli-commands.md` | `hermes completion` documented as bash/zsh only; `hermes skills reset` subcommand absent | `hermes_cli/cli.py` `completion` accepts `bash|zsh|fish`; `skills reset [--restore] [--yes]` subcommand exists |
| 8 | Medium | `security.md` | Secret-prefix table listed only 22 patterns; "20+ patterns" claim repeated elsewhere | `agent/redact.py` `_PREFIX_PATTERNS` contains 35 vendor prefixes plus generic Authorization / JWT / DB-URL / URL-userinfo / sensitive-query-string / Telegram-bot / private-key forms |
| 9 | Medium | `api-server.md` | `X-Hermes-Session-Id` and `Idempotency-Key` mentioned only inside the CORS section | `gateway/platforms/api_server.py` reads `X-Hermes-Session-Id` from request and emits it on every response (non-streaming + SSE); `Idempotency-Key` keys a 5-minute response cache |
| 10 | Medium | `reference/environment-variables.md` | `HERMES_REDACT_SECRETS` env var was undocumented | `agent/redact.py` snapshots `os.getenv("HERMES_REDACT_SECRETS", "")` at import time so runtime mutation cannot disable redaction |

(Findings #1–#10 have all been corrected and committed during this pass.)

## Findings verified as already correct (no change required)

- `README.md` — version claim `v0.11.0 (v2026.4.23)` matches `RELEASE_v0.11.0.md`.
- `tools.md` — toolset inventory matches `tools/registry.py` at `v2026.4.23` after the count fix (#5).
- `dashboard-plugins.md` — plugin tab interface matches `web/dashboard/` and `agent/dashboard.py`.
- `guides/aws-bedrock.md` — Converse API, IAM, inference profiles, and Guardrails support verified against `agent/bedrock_adapter.py`.
- `messaging/qqbot.md` — config keys (`app_id`, `client_secret`, `markdown_support`, `dm_policy`, `allow_from`, `stt`) and env vars match `gateway/platforms/qqbot/adapter.py`.
- `voice-mode.md` — `stt.enabled` already documented at line 239; push-to-talk references already present in `tts.md`.
- `profiles.md` — token-lock isolation list (Telegram, Discord, Slack, WhatsApp, Signal) matches the upstream `website/docs/user-guide/profiles.md` text at `v2026.4.23`.

## Upstream gaps observed (not corrected in this pass)

The following items are present in `v2026.4.23` source but are not documented in upstream `website/docs/**` at that tag. They represent upstream documentation debt rather than hermes-agent-docs drift; they are not authored here because this pass syncs docs to the released documentation surface, not beyond it:

- **Token-lock coverage for QQBot and Weixin** — `gateway/platforms/qqbot/adapter.py` implements an in-process `_token_lock` for duplicate-credential detection, and Weixin enforces its own credential bind, but `website/docs/user-guide/profiles.md` at `v2026.4.23` still names only Telegram, Discord, Slack, WhatsApp, and Signal. The docs-repo `profiles.md` matches upstream verbatim and is intentionally left aligned.
- **`pre_gateway_dispatch` plugin hook** — referenced in earlier prerelease docs but not present in `v2026.4.23` source; any docs claiming this hook were removed in this pass.
- **Cron `wakeAgent` defaults** — present in source (`agent/cron.py`) but only partially documented upstream.

## Out-of-scope items (deferred to next release)

The following landed on `origin/main` **after** `v2026.4.23` and are NOT treated as released content:

- Any provider, skill, plugin, or tool added after `e77d44c31ec82597d96c5fe44db71a8ed059e3db`.
- Any post-tag rename of slash commands, env vars, or config keys.

These will be covered when the next release tag is cut.

## Cosmetic drifts left unchanged

- `tool-gateway.md` — minor heading wording drift carried over from the v2026.4.16 pass; no factual divergence at `v2026.4.23`.
- `messaging/index.md` vs `messaging/README.md` — README serves GitHub folder rendering; `index.md` carries Docusaurus frontmatter. Both are intentionally kept in sync.
- `providers.md` (root) — docs-repo-only file augmenting `integrations/providers.md` with Bedrock and tool-gateway specifics. Intentional layering; preserved.

## Public release surface

The local `RELEASE_v0.11.0.md` is the authoritative artifact for this pass. Online cross-check of `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.23` was performed in Task 14; any divergence between the public release page and the local artifact is recorded immediately below this section in commit history rather than silently merged into product documentation.

## Exit state

- All v2026.4.23 factual findings from this audit are corrected and committed.
- Audit artifact set (`review_plan`, `release_evaluation`, `repo_review`, `source_validation_matrix`, `documentation_integrity_findings`) is present for `v2026.4.23` and mirrors the structure of the `v2026.4.16` set.
- Docs repo is aligned with the `v2026.4.23` release boundary; post-tag work on `origin/main` is excluded.
