# Source-repo review — v2026.5.13

**Repo:** `~/src/hermes/hermes-agent` (read-only mirror; NOT modified by this audit)
**Tag-pinned commit:** `498bfc7bc12a937621b4215312049b1000726df3`
**Main HEAD:** `486b692ddd801f8f665d3fff023149fb1cb6509e`

## What we inspected

Six parallel read-only discovery agents (A–F) were dispatched in the brainstorming phase. Each agent surveyed one subsystem and returned a finding set keyed to `file:line`. Findings were then bucketed into three classes:

1. **Tag-accurate but undocumented** → add to docs.
2. **Documented but contradicted by tag code** → correct the docs.
3. **Documented as released but not in tag** → add a "current-main only" banner.

## Subsystem-by-subsystem inventory at the tag

### CLI surface (`hermes_cli/`)

- `auth.py` at tag enumerates the full `PROVIDER_REGISTRY`. The `stepfun` and `azure-foundry` entries were missing from the docs; both now in `providers.md` + mirror.
- `config.py` at tag has all five platform allowlist env vars (`SLACK_ALLOWED_CHANNELS`, `TELEGRAM_ALLOWED_CHATS`, `MATTERMOST_ALLOWED_CHANNELS`, `MATRIX_ALLOWED_ROOMS`, `DINGTALK_ALLOWED_CHATS`). All five now in `reference/environment-variables.md`.
- `web_server.py:auth_middleware` excludes `/api/plugins/*` from session-token gating. Three docs files corrected.
- `doctor.py` at tag has the StepFun health-check row at line 205 — consistent with the new doc entry.

### Plugin system (`plugins/`, `agent/plugin_*`)

- `plugins/platforms/` at tag: `google_chat`, `irc`, `teams` only. The docs had over-claimed LINE — now scoped to current-main.
- `agent/plugin_context.py` at tag has no `llm` attribute. `agent/plugin_llm.py` is wholly absent. All three docs locations gated.
- The `kanban-worker` skill is present at the tag; the lane *abstraction* (assignee identity contract, non-Hermes spawn registration) is on `main` only. The lanes page got a banner; the worker / orchestrator skill pages did not, because the skills themselves shipped.

### Kanban kernel (`tasks/kanban_*`)

- Dispatcher, per-board SQLite, claim TTL, stale-claim detection, stranded-task diagnostics, `task_runs` + `task_events` audit tables — all at tag.
- `expected_run_id` parameter exists on terminating tools at tag.
- The "lane" terminology is post-tag.

### Tools (`tools/`)

- `mcp_tool.py` at tag has the 3-minute idle keepalive (`_KEEPALIVE_INTERVAL = 180`) and reconnect-on-keepalive-fail logic. PR #17003 is at tag. Documented in `mcp.md`.
- `web_search.py` at tag has SearXNG support and the split web-tool backend selection. Brave Search free + DDGS backends are NOT at tag — markered in `web-search.md`.
- `computer_use` is wholly absent at tag (`git grep -l computer_use v2026.5.7 -- '*.py'` returns nothing). Page-level banner added.

### Auxiliary model client (`agent/auxiliary_client.py`)

- `_API_KEY_PROVIDER_AUX_MODELS_FALLBACK` at tag includes `"stepfun": "step-3.5-flash"`. Used as the default aux model for the StepFun provider.

### i18n (`agent/i18n/`)

- 8 locales at tag: `en, zh, ja, de, es, fr, tr, uk`. The docs previously listed 16. The 8 extras were hallucinated; removed.

### Gateway (`gateway/`)

- `gateway/platforms/` at tag does NOT contain `line.py`. Confirmed via `git show v2026.5.7:gateway/platforms/line.py` → fatal.
- Google Chat env-var coverage now reflects the full set in `gateway.md`.

## Read-only constraint

Per the user instruction, the source repo was not modified. Every `git show v2026.5.7:<path>` and `git grep` invocation was a read-only inspection of the tag tree. No `git checkout` was performed in the source worktree.

## Cross-agent integration

The six discovery agents' findings were merged into the 19-patch list during the brainstorming phase. Of those:

- 18 applied to docs.
- 1 deferred upstream (skills-catalog regen).

No internal conflicts between findings — each touched a distinct doc surface.

## Recommendation for upstream

These items belong in the source repo, not the docs:

1. Regenerate `reference/skills-catalog.md` upstream and refresh in docs (`kanban-orchestrator` description truncation at line 67).
2. Consider adding a `RELEASE_NOTES_DOC_SYNC.md` to the source repo enumerating which post-tag features have docs gated and which are tag-released. This would reduce the kind of drift this audit corrected.

## Post-sync repo state (2026-05-13)

**Source:** full resync from `hermes-agent/website/docs/` at origin/main HEAD (post-v2026.5.7 + post-tag fixes through 2026-05-13). 253 files updated, 13 added, 60+ root-level duplicate files deleted, 2 root-level duplicate directories (`features/`, `messaging/`) removed, 7 orphans identified.

**Canonical layout after cleanup:**

- `getting-started/`
- `user-guide/`
- `developer-guide/`
- `reference/`
- `guides/`
- `integrations/`
- Repo root: `README`, `changelog`, `index`, and audit artifacts only (no content pages)

**File counts:** ~330 total markdown files, down from ~488 prior to dedup removal.

**Audit-artifact series at v2026.5.13 (5 documents):**

1. `documentation_integrity_findings_v2026.5.13.md`
2. `release_evaluation_v2026.5.13.md`
3. `repo_review_v2026.5.13.md`
4. `review_plan_v2026.5.13.md`
5. `source_validation_matrix_v2026.5.13.md`

**Historical audit dates preserved:** v2026.4.8, v2026.4.13, v2026.4.16, v2026.4.23, v2026.4.30, v2026.5.6, v2026.5.7, v2026.5.13 (8 cohorts on disk).

**Branch:** `sync/v2026.5.7` — to be merged via PR per the review plan.

**Build infrastructure:** none in this repo. Docusaurus configuration lives in the consumer site repo; this repo is a content-only mirror.
