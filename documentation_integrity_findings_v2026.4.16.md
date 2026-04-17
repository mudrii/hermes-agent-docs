# Documentation Integrity Findings: Hermes Agent v0.10.0 (v2026.4.16)

**Audit date:** 2026-04-17
**Target release:** `v2026.4.16` / `Hermes Agent v0.10.0` (commit `1dd6b5d5fb94cac59e93388f9aeee6bc365b8f42`)
**Previous baseline:** `v2026.4.13` / `Hermes Agent v0.9.0`
**Scope:** Deep factual validation of `hermes-agent-docs` against the v2026.4.16 source tag, following the initial v0.10.0 sync pass.

## Method

- Authoritative source surface read only via `git show v2026.4.16:<path>` to avoid contamination from post-release commits on `origin/main`.
- Three parallel explorer subagents covered: (A) Tool Gateway / AWS Bedrock / Providers, (B) messaging platforms, (C) changelog / README / tools / features.
- Public release status re-verified against `github.com/NousResearch/hermes-agent/releases`.

## Findings applied

| # | Severity | File | Issue | Commit |
|---|----------|------|-------|--------|
| 1 | Critical | `changelog.md`, `release_evaluation_v2026.4.16.md` | Stale "public-release-surface lagged" note — v2026.4.16 is now the latest published release | `aaad269` |
| 2 | High | `changelog.md` v0.10.0 Highlights | Missing four released features from the v2026.4.13..v2026.4.16 window | `126e458` |
| 3 | High | `plugins.md` | Missing `ctx.register_command()` row in "What plugins can do" | `b1a40f0` |
| 4 | High | `web-dashboard.md` | Missing entire Themes section (~70 lines) | `2c9cd57` |
| 5 | Medium | `messaging/README.md` | QQ Bot capability row claimed Threads and Streaming support that `gateway/platforms/qqbot.py` does not implement | `8e3b27a` |
| 6 | Medium | `messaging/webhook.md` | Orphaned duplicate of `webhooks.md` with stale deliver-targets list; no counterpart upstream at v2026.4.16 | `a17e410` |

### Added to v0.10.0 changelog highlights (finding #2)

- **Ollama Cloud built-in provider** (commit `1b61ec47`) — dynamic model discovery wired into `/model` picker
- **Podman support** (commit `8548893d`, PR [#10066](https://github.com/NousResearch/hermes-agent/pull/10066)) — `find_docker()` autodetects Podman for rootless container workflows
- **Dashboard theme system** (commit `3f6c4346`) — live theme switching with six built-ins (Hermes Teal, Midnight, Ember, Mono, Cyberpunk, Rosé) plus user themes under `~/.hermes/dashboard-themes/`
- **Plugin `register_command()`** (commit `498b995c`, PR [#10626](https://github.com/NousResearch/hermes-agent/pull/10626)) — plugins can now register `/slash` commands

## Findings verified as already correct (no change required)

- `README.md` — version claim `v0.10.0 (v2026.4.16)` is accurate; no post-release features leaked in
- `tools.md` — toolset inventory matches `tools/registry.py` at v2026.4.16
- `image-generation.md` — correctly describes FAL.ai FLUX 2 Pro + Clarity Upscaler (no post-release Recraft V4 Pro / Nano Banana Pro claims)
- `tts.md` — six providers (Edge TTS, ElevenLabs, OpenAI TTS, MiniMax, Mistral Voxtral, NeuTTS) match v2026.4.16 source
- `dashboard-plugins.md` — plugin interface matches source at v2026.4.16
- `guides/aws-bedrock.md` — inference profile IDs, credential chain, and Guardrails support verified against `agent/bedrock_adapter.py`
- `messaging/qqbot.md` — config keys (`app_id`, `client_secret`, `markdown_support`, `dm_policy`, `allow_from`, `stt`) and env vars match `gateway/platforms/qqbot.py`
- `messaging/feishu.md` — env var table at lines 400–421 matches source
- `messaging/matrix.md` — auth and configuration coverage verified
- Curated Nous Portal model list in docs correctly omits `claude-opus-4.7` (which was added post-release in PR #11398 and is NOT in v2026.4.16)

## Upstream gaps observed (not corrected in this pass)

The following items are present in v2026.4.16 **source code** but are not documented in upstream `website/docs/` at that tag. They represent upstream documentation debt rather than hermes-agent-docs drift:

- **Ollama Cloud provider catalog entry** — `hermes_cli/models.py:545` defines `ProviderEntry("ollama-cloud", ...)` but `website/docs/integrations/providers.md` at v2026.4.16 only documents local Ollama, not Ollama Cloud.

These are not corrected here because this pass syncs docs to the v2026.4.16 release surface; authoring content that upstream did not ship is out of scope. They can be filed upstream or addressed in a future pass.

## Out-of-scope items (deferred to next release)

The following landed on `origin/main` **after** v2026.4.16 and are NOT treated as released content:

- Google Gemini Cloud Code Assist OAuth (`agent/gemini_cloudcode_adapter.py`, `agent/google_code_assist.py`, `agent/google_oauth.py`)
- `claude-opus-4.7` in the Nous Portal curated model list (PR #11398)
- Image generation Recraft V4 Pro and Nano Banana Pro (PR #11406)
- `tools/mcp_oauth_manager.py` consolidation (PR #11383)
- Feishu P2P event handler registrations

These will be covered when the next release tag is cut.

## Cosmetic drifts left unchanged

- `tool-gateway.md` — minor heading wording drift ("Included tools" vs upstream's "What's Included"). No factual divergence, so left as-is.
- `messaging/README.md` vs `messaging/index.md` — README serves GitHub folder rendering; `index.md` carries Docusaurus frontmatter. Both are intentionally kept in sync.
- `providers.md` (root) — docs-repo-only file augmenting `integrations/providers.md` with Bedrock details. Intentional layering; preserved.

## Exit state

- All v2026.4.16 factual findings from this audit are corrected and committed.
- Public release surface status note is current.
- Docs repo remains aligned with the v2026.4.16 release boundary.
