# Source validation matrix — v2026.5.13 audit

**Tag commit:** `498bfc7bc12a937621b4215312049b1000726df3` (v2026.5.7)
**Source HEAD:** `486b692ddd801f8f665d3fff023149fb1cb6509e`

Every doc claim modified in this audit is cross-referenced to a source file path at the tag commit (or to a confirmed-absent path with the verification command).

## Claim → source-file proof

| Doc file:line claim | Source-of-truth at tag | Verification |
|---|---|---|
| `reference/environment-variables.md:92` — locale list `en, zh, ja, de, es, fr, tr, uk` | i18n bundles in tag tree | `git ls-tree -r v2026.5.7 -- 'agent/i18n/' \| grep locale` |
| `reference/environment-variables.md:453` — `teams_pipeline` is post-tag | `plugins/teams_pipeline/` absent | `git show v2026.5.7:plugins/teams_pipeline/__init__.py` → fatal |
| `reference/environment-variables.md:532` — `HERMES_SESSION_ID` is a template token, not a subprocess env var | `agent/skill_preprocessing.py` substitutes `${HERMES_SESSION_ID}` in SKILL.md body | `git show v2026.5.7:agent/skill_preprocessing.py \| grep HERMES_SESSION_ID` |
| `reference/environment-variables.md` — new rows for `SLACK_ALLOWED_CHANNELS`, `TELEGRAM_ALLOWED_CHATS`, `MATTERMOST_ALLOWED_CHANNELS`, `MATRIX_ALLOWED_ROOMS`, `DINGTALK_ALLOWED_CHATS` | Each declared in `hermes_cli/config.py` at tag | `git grep -n ALLOWED_CHATS v2026.5.7 -- 'hermes_cli/config.py'` |
| `messaging/teams-meetings.md` banner | `teams_pipeline` plugin absent at tag | `git ls-tree v2026.5.7 plugins/ \| grep teams` shows only `plugins/platforms/teams` |
| `messaging/line.md` banner | LINE plugin absent at tag | `git ls-tree v2026.5.7 plugins/platforms/` returns `google_chat`, `irc`, `teams` only |
| `messaging/index.md` body — Feishu/Lark, WeCom, Weixin, BlueBubbles, QQ, IRC, Webhooks present; LINE removed | `plugins/platforms/` directory listing at tag | `git ls-tree v2026.5.7 plugins/platforms/` |
| `gateway.md:447` — Google Chat env vars | `hermes_cli/config.py` GOOGLE_CHAT_* entries at tag | `git grep -n GOOGLE_CHAT_ v2026.5.7 -- 'hermes_cli/config.py'` |
| `developer-guide/plugin-llm-access.md` page-level warning | `agent/plugin_llm.py` absent | `git cat-file -p v2026.5.7:agent/plugin_llm.py` → fatal |
| `plugins.md:114` + 2 mirrors — `ctx.llm.complete` current-main only | `PluginContext` at tag has no `llm` attribute | `git show v2026.5.7:agent/plugin_context.py \| grep -n 'self.llm\|llm ='` → zero matches |
| `extending-the-dashboard.md:730` + 2 mirrors — plugin routes NOT session-gated | `hermes_cli/web_server.py:auth_middleware` excludes `/api/plugins/*` | `git show v2026.5.7:hermes_cli/web_server.py \| sed -n '224,234p'` |
| `web-dashboard.md:180` + 2 mirrors — corrected auth description | same `auth_middleware` | as above |
| `computer-use.md` page-level warning | `computer_use` feature wholly absent at tag | `git grep -l computer_use v2026.5.7 -- '*.py'` → empty |
| `web-search.md` Brave/DDGS markers | Backends absent at tag | `git show v2026.5.7:tools/web_search.py \| grep -c -i 'brave\|ddgs'` → 0 |
| `features/kanban-worker-lanes.md` banner | Lane-abstraction terminology is post-tag (kernel ships; lane terms evolve on main) | `git grep -ni 'worker lane' v2026.5.7 -- 'skills/'` finds zero matches |
| `providers.md` StepFun entry | `STEPFUN_API_KEY` + region inference at tag | `git grep -n STEPFUN_API_KEY v2026.5.7 -- 'hermes_cli/'` |
| `providers.md` Azure AI Foundry entry | `azure-foundry` ProviderConfig at tag | `git show v2026.5.7:hermes_cli/auth.py \| sed -n '409,416p'` |
| `providers.md` base-URL list addition (`STEPFUN_BASE_URL`, `AZURE_FOUNDRY_BASE_URL`) | Each declared as `base_url_env_var` in respective ProviderConfig at tag | `git grep -n 'base_url_env_var' v2026.5.7 -- 'hermes_cli/auth.py'` |
| `mcp.md` keepalive subsection | `_KEEPALIVE_INTERVAL = 180` in `tools/mcp_tool.py` at tag | `git show v2026.5.7:tools/mcp_tool.py \| sed -n '1125,1165p'` |
| `mcp.md` MEDIA tag subsection | `[MEDIA: image/...]`, etc., in test assertions at tag | `git show v2026.5.7:tests/tools/test_mcp_image_content.py \| grep -n MEDIA` |
| `changelog.md:20` — rewrite | combines findings above (ctx.llm absent + plugin route auth correction) | both verified above |

## Verification policy

For *additive* claims (StepFun, Azure Foundry, keepalive, MEDIA tags): the change adds doc content for a feature present at the tag. The verification proves *presence at tag*.

For *banner* claims (LINE, teams_pipeline, ctx.llm, computer_use, Brave/DDGS, worker-lane abstraction): the change adds a "current-main only" banner without removing the body. The verification proves *absence at tag*.

For *correction* claims (locale list, plugin-route auth, `HERMES_SESSION_ID` semantics): the change replaces a previously-incorrect statement with a tag-accurate one. The verification proves *current code contradicts the old doc text*.

## Patches not applied

| ID | File:line | Reason |
|---|---|---|
| #19 | `reference/skills-catalog.md:67` | Auto-generated description truncation; fix belongs in the source skill's front-matter and a regen of the catalog, not a hand-patch. |

## Provenance

- Doc commit base: `0007eb8f15ee54538f7fda1b123038635c06e63e`
- Source tag: `v2026.5.7` (annotated → commit `498bfc7bc1...`)
- Audit performed entirely against the tag tree; `main` HEAD `486b692ddd...` consulted only to confirm features were on `main` but absent at the tag.

## Post-sync validation (2026-05-13)

After the full resync of the docs repo from `/Users/mudrii/src/hermes/hermes-agent/website/docs/` at `origin/main` HEAD, the following claims were re-validated against the new docs tree to confirm the mirror is faithful.

| Claim | Verified location | Status |
|---|---|---|
| `HERMES_REDACT_SECRETS` env var documented | `reference/environment-variables.md` | Present at expected path |
| `MESSAGING_CWD` env var documented | `reference/environment-variables.md` | Present at expected path |
| Provider rename "Qwen Cloud" (PR #24835) | Source: `hermes_cli/models.py:911`, `hermes_cli/auth.py:287` (`ProviderEntry.name = "Qwen Cloud"`). Display label only — config provider key remains `alibaba` (`user-guide/configuration.md` provider registry list unchanged). No user-facing doc rename required. | Verified (display-only rename in interactive picker) |
| `cli.md` covers Shift+Enter / Kitty terminal keybindings | `cli.md` | 6 hits confirmed in the page |
| 26 messaging platform pages present | `messaging/` directory tree | All 26 platform pages confirmed present |
| New skill: `apple-macos-computer-use` | `skills/optional/apple-macos-computer-use/` | Path confirmed |
| New skill: `productivity-teams-meeting-pipeline` | `skills/optional/productivity-teams-meeting-pipeline/` | Path confirmed |
| New skill: `creative-hyperframes` | `skills/optional/creative-hyperframes/` | Path confirmed |
| New skill: `devops-watchers` | `skills/optional/devops-watchers/` | Path confirmed |
| New skill: `mlops-inference-outlines` | `skills/optional/mlops-inference-outlines/` | Path confirmed |
| New skill: `mlops-training-axolotl` | `skills/optional/mlops-training-axolotl/` | Path confirmed |
| New skill: `mlops-training-trl-fine-tuning` | `skills/optional/mlops-training-trl-fine-tuning/` | Path confirmed |
| New skill: `mlops-training-unsloth` | `skills/optional/mlops-training-unsloth/` | Path confirmed |

### Verification policy (post-sync addendum)

Because the docs are now a full mirror of upstream `main` (not a hand-curated overlay on the tag), per-claim source-file verification reduces to a single repo-level check: docs-tree equality with `hermes-agent/website/docs/` at the sync commit. Individual env-var, provider, and skill-path checks above are spot-confirmations of that equality rather than independent proofs.

