# Release evaluation — v0.13.0 (v2026.5.7)

**Audit date:** 2026-05-13
**Released-version boundary:** tag `v2026.5.7` → commit `498bfc7bc12a937621b4215312049b1000726df3` (annotated tag object `e19fc91cb82cb4e0dc74f6d63a89d9b6c2135241`)
**Source HEAD at audit time:** `486b692ddd801f8f665d3fff023149fb1cb6509e` (post-tag main)
**Docs HEAD at audit start:** `0007eb8f15ee54538f7fda1b123038635c06e63e`

## Scope

This evaluation answers: *what is actually in the v0.13.0 release artifact, and what does the docs repo currently claim about it?*

The released set is locked to the contents of the `v2026.5.7` tag object. Anything added on `main` after `498bfc7bc1...` is **post-tag** and must not be marketed as part of v0.13.0 without an explicit "current-main only" banner.

## Confirmed at tag (no banner required)

| Feature | Source evidence at tag |
|---|---|
| Kanban dispatcher + per-board SQLite | `tasks/kanban_*` files in tree at tag |
| `kanban-worker` skill | `skills/devops/kanban-worker/SKILL.md` at tag |
| `kanban-orchestrator` skill | `skills/devops/kanban-orchestrator/SKILL.md` at tag |
| Persistent `/goal` | `agent/goal.py` at tag |
| Checkpoints v2 with disk guardrails | `agent/checkpoints.py` at tag |
| Google Chat platform plugin | `plugins/platforms/google_chat/` at tag |
| Teams platform plugin (chat side) | `plugins/platforms/teams/` at tag |
| IRC platform plugin | `plugins/platforms/irc/` at tag |
| StepFun provider | `hermes_cli/auth.py:234`, `agent/auxiliary_client.py:265`, `agent/model_metadata.py:49,62,297,298` at tag |
| Azure AI Foundry provider | `hermes_cli/auth.py:409-415`, `hermes_cli/main.py:1988-1989`, `hermes_cli/config.py:2631` at tag |
| MCP idle keepalive (3-min `list_tools` ping) | `tools/mcp_tool.py:1130-1165` at tag (PR #17003) |
| MCP MEDIA tag placeholders for image/audio/video | `tests/tools/test_mcp_image_content.py:69,73,78,137` at tag |
| Platform allowlists (Slack/Telegram/Mattermost/Matrix/DingTalk) | env-var rows in `hermes_cli/config.py` at tag |
| Locale set (8): en/zh/ja/de/es/fr/tr/uk | i18n bundles in tag tree |
| Dashboard session-token middleware on non-public `/api/*` | `hermes_cli/web_server.py:auth_middleware` at tag |

## Confirmed POST-tag (banner required in docs)

| Feature | Why not at tag | Banner location applied |
|---|---|---|
| `ctx.llm.complete` plugin LLM API | `agent/plugin_llm.py` absent at tag (`git show v2026.5.7:agent/plugin_llm.py` → not found) | `developer-guide/plugin-llm-access.md`, `plugins.md` + 2 mirrors, `changelog.md:20` (re-scoped) |
| LINE platform plugin | `plugins/platforms/line/` and `gateway/platforms/line.py` absent at tag | `messaging/line.md` + mirror, `messaging/index.md` + mirror, `gateway.md:448` |
| Teams meeting summary delivery (`teams_pipeline` plugin + CLI) | `plugins/teams_pipeline/` absent at tag | `messaging/teams-meetings.md` + mirror, `reference/environment-variables.md:453` |
| Kanban worker-lane abstraction (lane identity, non-Hermes spawn plug-in) | Lane contract terminology absent at tag (the kernel ships; the **lane** abstraction is on `main`) | `features/kanban-worker-lanes.md` + mirror |
| `computer_use` feature (whole feature) | `git grep computer_use v2026.5.7 -- '*.py'` → zero matches | `computer-use.md` + 2 mirrors |
| Brave Search free backend + DDGS web-search backends | Backends added on `main` after tag | `web-search.md` + 2 mirrors |

## Banner pattern used

```
:::info Current-main only — not in v0.13.0
[feature] ships on `main` after the v0.13.0 (v2026.5.7) release. For
v0.13.0 stable use, treat [adjacent page] as the source of truth.
:::
```

This is preferred over deleting the post-tag content because:

1. Users reading docs on `main` need to find the feature.
2. Users on the v0.13.0 binary need to know it isn't theirs yet.
3. Removal would force a re-doc cycle when the next release tag promotes the feature.

## Locale list correction (Agent E finding)

The docs previously listed 16 locales. The tag tree contains 8 release locales: `en, zh, ja, de, es, fr, tr, uk` (with English fallback for unknown values). The extra 8 were hallucinated. Corrected in:

- `reference/environment-variables.md:92`
- `configuration.md:1223` + `user-guide/configuration.md:1223`

## Dashboard auth scope correction (Agent C finding)

The previous docs claimed plugin routes were "covered by dashboard auth." Source evidence contradicts this:

```python
# hermes_cli/web_server.py, auth_middleware
if path.startswith("/api/") and path not in _PUBLIC_API_PATHS \
   and not path.startswith("/api/plugins/"):
    # ...session-token check...
```

The middleware explicitly **excludes** `/api/plugins/*`. Corrected in:

- `extending-the-dashboard.md:730` + 2 mirrors (`:::warning` admonition)
- `web-dashboard.md:180` + 2 mirrors (rewrote intro paragraph)
- `changelog.md:20` (rewrote the v0.13.0 highlight to state plugin authors must apply their own access checks)

## Auto-generated content noted (not edited)

`reference/skills-catalog.md:67` contains a trailing-ellipsis truncation in the `kanban-orchestrator` description. This file is auto-generated from skill front-matter upstream — hand-patching it would diverge from the generator. Recommendation: fix the source skill's `description:` field upstream and regenerate. **No docs edit applied.**

## Sign-off

19-patch set planned → 18 applied (1 deferred to upstream regen). All applied patches are factual and traceable to source files at the tag commit. Banner pattern keeps post-tag content visible without misclaiming it as released.

## Post-sync evaluation (2026-05-13)

- **Latest SemVer overlay:** v0.13.0 remains the most recent SemVer release; no new SemVer published between 2026-05-07 and 2026-05-13.
- **Latest CalVer tag:** v2026.5.7 is still the most recent upstream weekly tag. v2026.5.13 has **no corresponding upstream tag** — it is a docs-side audit cadence marker only.
- **Post-tag commits captured:** 412 lines of multi-file commits affecting `hermes-agent/website/docs/` were pulled from origin/main HEAD past the v2026.5.7 tag. Commit list snapshot retained at `/tmp/post-tag-doc-commits.txt` during sync.
- **Post-tag features now reflected in docs:**
  - Qwen Cloud rename (#24835)
  - Camofox externally-managed sessions (#24584, 62fd905)
  - LINE platform support (50f9fee)
  - `/handoff` cross-platform transfer (373c4d6)
  - `ctx.llm` plugin access (5aa755e)
  - LSP semantic diagnostics + follow-up (#24168, #24709)
  - Per-turn file-mutation verifier (#24498)
  - `hermes update` cua-driver refresh (#24063)
- **Release-banner posture:** unchanged — post-tag content remains gated/labelled rather than represented as part of an existing SemVer release.
