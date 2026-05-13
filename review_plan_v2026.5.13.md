# Review Plan: Hermes Agent v0.13.0 release-bound docs sync (2026-05-13)

> **For agentic workers:** This is a documentation audit plan, not a code-implementation plan. Steps use checkbox (`- [ ]`) syntax. Each step is one action; complete sequentially unless a step explicitly dispatches parallel sub-agents.

**Goal:** Re-validate `~/src/hermes/hermes-agent-docs` against the released `Hermes Agent v0.13.0` / `v2026.5.7` source tree (immutable tag `498bfc7bc12a937621b4215312049b1000726df3`), close any drift introduced since the 2026-05-11 audit, and refresh the audit-artifact set for **v2026.5.13**.

**Architecture:**

1. Bound release claims to the `v2026.5.7` tag and the `RELEASE_v0.13.0.md` artifact — these are immutable.
2. Use parallel read-only discovery agents per independent domain; each returns a focused defect list with `file:line` evidence.
3. Apply a single integrated docs-only patch set, then re-generate the five-document audit artifact set for v2026.5.13.
4. Post-tag commits between `271883447` (last audit's source pin) and `486b692dd` (today's `main`) are consulted *only* for documentation-correctness fixes — they are explicitly **not** added to v0.13.0 release claims.

**Tech stack:** Plain Markdown docs. Validation via `git`, `grep`, file reads. No code execution against the codebase is required.

**Date / version metadata for this run:**
- Today: **2026-05-13**
- Release boundary: **`v2026.5.7` / `Hermes Agent v0.13.0`** (tag commit `498bfc7bc12a937621b4215312049b1000726df3`)
- Previous audit boundary: `271883447e7b8a5b9bd95879aca71afadc87616f` (`v2026.5.7-446-g271883447`)
- Current `origin/main`: `486b692dd` (`v2026.5.7-545-g486b692dd`)
- Commits in `271883447..486b692dd`: 99
- Previous audit artifact set: `*_v2026.5.7.md` (dated 2026-05-11)
- New audit artifact set: `*_v2026.5.13.md`

---

## Phase 0 — Baseline (already done in this session)

- [x] `git fetch --all --prune && git pull --ff-only` for `~/src/hermes/hermes-agent` (pulled 99 commits — `271883447` → `486b692dd`).
- [x] `git fetch --all --prune && git pull --ff-only` for `~/src/hermes/hermes-agent-docs` (already up to date at `0007eb8`).
- [x] Confirmed local tag `v2026.5.7` exists at `498bfc7bc12a937621b4215312049b1000726df3`.
- [x] Confirmed `RELEASE_v0.13.0.md` exists in source tree (641 lines).

---

## Phase 1 — Release-boundary sanity

Quick header-only verification before dispatching expensive agents.

- [ ] **Step 1.1:** Read `changelog.md` lines 1-20 in `hermes-agent-docs`. Verify "Current stable release: v0.13.0 (v2026.5.7, May 7, 2026)" is still the topmost stated stable. **Expected:** PASS (already updated in the v2026.5.7 audit).

- [ ] **Step 1.2:** Read `README.md` top of `hermes-agent-docs`. Verify stable release marker.

- [ ] **Step 1.3:** Read `index.md` top. Verify stable release marker.

- [ ] **Step 1.4:** Confirm presence of all v2026.5.7 audit artifacts in repo root:
  - `review_plan_v2026.5.7.md`, `release_evaluation_v2026.5.7.md`, `repo_review_v2026.5.7.md`, `source_validation_matrix_v2026.5.7.md`, `documentation_integrity_findings_v2026.5.7.md`. **Expected:** all 5 present.

If any of Step 1.1-1.3 are stale, fix in Phase 4 (not now — record finding).

---

## Phase 2 — Parallel discovery agents

**Six independent read-only agents** are dispatched in a single tool-call batch. Each returns a structured defect list with `file:line` citations. No agent modifies files.

Per `superpowers:dispatching-parallel-agents`: domains are independent, so they can run concurrently. Each prompt is self-contained — agents do not share context.

Common context every agent receives:
- Release boundary: `v2026.5.7` (tag commit `498bfc7bc12a937621b4215312049b1000726df3`), source repo: `~/src/hermes/hermes-agent`, docs repo: `~/src/hermes/hermes-agent-docs`.
- Source-of-truth precedence: (1) `RELEASE_v0.13.0.md` for release claims, (2) tag-pinned source files via `git show v2026.5.7:<path>`, (3) tag-pinned `website/docs/**` first-party docs, (4) post-tag `main` only for doc-correctness fixes.
- The previous v2026.5.7 audit's findings (`documentation_integrity_findings_v2026.5.7.md`) is the prior-state baseline — do not re-litigate fixes already applied; verify they're still correct.
- Required output format per finding: `<docs_file>:<line_range or section> — <claim in docs> — <source evidence: file:line OR git show v2026.5.7:<path>> — <recommended edit>`.
- Hard constraint: do not propose adding post-tag features to v0.13.0 release claims; if a feature is post-tag, recommend it be labeled "current-main / post-v0.13" or omitted from release-scoped pages.

### Agent A — Release & changelog surfaces

**Files in scope (docs):**
- `changelog.md`, `README.md`, `index.md`, `overview.md`
- Any release-scoped highlight references inside `user-guide/features/index.md` / `features/_category_.json`

**Source-of-truth files:**
- `~/src/hermes/hermes-agent/RELEASE_v0.13.0.md`
- `git show v2026.5.7:README.md`
- GitHub Releases listing (verify `Hermes Agent v0.13.0 (2026.5.7)` is latest)

**Tasks:**
- Verify version, date, "Tenacity" codename, delivery-note numbers (864 commits / 588 PRs / 829 files / 128,366 insertions / 282 issues) match `RELEASE_v0.13.0.md`.
- Verify every "Highlights" bullet in `changelog.md` v0.13.0 section has a backing entry in `RELEASE_v0.13.0.md`.
- Spot-check the v0.12.0 → v0.13.0 compare URL.
- Confirm no v0.14.x / unreleased claims leak into the release-section copy.

### Agent B — Messaging & gateway

**Files in scope (docs):**
- Both mirrors: `messaging/*.md` and `user-guide/messaging/*.md`
- `gateway.md`, `messaging/index.md`, `messaging/README.md`
- `reference/environment-variables.md` (gateway / platform vars)

**Source-of-truth files (tag-pinned):**
- `gateway/config.py`, `gateway/platforms/*.py`, `gateway/platform_registry.py`
- `plugins/platforms/google_chat/plugin.yaml`, `plugins/platforms/line/plugin.yaml`, `plugins/platforms/teams/{plugin.yaml,adapter.py}`, `plugins/platforms/qqbot/*`
- `~/src/hermes/hermes-agent/website/docs/user-guide/messaging/**`

**Tasks:**
- Verify Google Chat is documented as the 20th platform with correct env vars and setup steps.
- Verify Slack / Telegram / Mattermost / Matrix / DingTalk `allowed_*` allowlist docs match released config.
- Verify QQBot capability notes (approval keyboards, chunked upload, quoted attachments).
- Verify Teams plugin env vars are documented as standalone, separate from the Teams meeting-pipeline (which is post-release).
- Confirm `LINE` is treated as a current-main item, not a v0.13.0 release item.
- Confirm streaming defaults & restart-notification copy match `gateway/config.py` at tag.

### Agent C — Plugins, dashboard, Kanban, providers

**Files in scope (docs):**
- `plugins.md`, `built-in-plugins.md`, `extending-the-dashboard.md`
- `developer-guide/plugin-llm-access.md`, `developer-guide/model-provider-plugin.md`, `developer-guide/image-gen-provider-plugin.md`
- `features/plugins.md`, `features/built-in-plugins.md`, `features/extending-the-dashboard.md`, `features/kanban.md`, `features/kanban-tutorial.md`, `features/kanban-worker-lanes.md`
- `kanban.md`, `kanban-tutorial.md`, `user-guide/features/kanban.md`, `user-guide/features/kanban-tutorial.md`
- `web-dashboard.md`, `user-guide/configuration.md` (dashboard sections)

**Source-of-truth files (tag-pinned):**
- `agent/plugin_llm.py`, `hermes_cli/plugins.py`, `hermes_cli/web_server.py`
- `tools/kanban_tools.py`, `hermes_cli/kanban_db.py`, `plugins/kanban/**`
- `tools/image_generation_tool.py`, `agent/transports/*`
- `~/src/hermes/hermes-agent/website/docs/developer-guide/{plugin-llm-access,model-provider-plugin,image-gen-provider-plugin}.md`

**Tasks:**
- Verify `ctx.llm` API description and `agent_id` semantics.
- Verify durable Kanban features (heartbeat, reclaim, zombie detect, retry budget, hallucination gate, distress diagnostics) all backed.
- Verify dashboard plugin API auth statement matches `hermes_cli/web_server.py`.
- Verify `transform_llm_output` hook is documented and listed in the released hook surface.
- Verify provider-plugin docs do not claim post-tag features as released.
- Confirm `kanban worker lanes` page remains labeled current-main only (not release).

### Agent D — CLI, TUI, browser, tools, MCP

**Files in scope (docs):**
- `cli-reference.md`, `cli.md`, `tui.md`, `user-guide/cli.md`, `user-guide/tui.md`
- `reference/cli-commands.md`, `reference/slash-commands.md`, `reference/tools-reference.md`, `reference/toolsets-reference.md`
- `browser.md`, `features/browser.md`, `user-guide/features/browser.md`
- `tools.md`, `toolsets.md`, `mcp.md`
- `code-execution.md`, `features/code-execution.md`
- `web-search.md`, `features/web-search.md`, `user-guide/features/web-search.md`

**Source-of-truth files (tag-pinned):**
- `hermes_cli/main.py`, `hermes_cli/_parser.py`, `hermes_cli/commands.py`, `hermes_cli/tools_config.py`
- `tools/browser_tool.py`, `tools/browser_supervisor.py`, `tools/web_tools.py`
- `mcp_serve.py`, `agent/mcp_*` (whatever exists at tag)
- `toolsets.py`

**Tasks:**
- Verify slash command surface excludes post-tag `/sessions` and `/handoff` from release-scoped reference (these are current-main items).
- Verify `hermes computer-use` is **not** present in v0.13.0 reference rows (post-release).
- Verify `hermes tools` subcommand surface matches `hermes_cli/tools_config.py` at tag.
- Verify SearXNG / `SEARXNG_URL` web-tool integration is documented.
- Verify browser supervisor fast-path eval description matches `browser_tool.py` at tag.
- Confirm Brave Search and DDGS web backends are documented as current-main consistency items (not v0.13 headlines).

### Agent E — Configuration, env vars, hooks, security

**Files in scope (docs):**
- `configuration.md`, `user-guide/configuration.md`
- `reference/environment-variables.md`
- `hooks.md`, `features/hooks.md`, `user-guide/features/hooks.md`
- `security.md`, `user-guide/security.md`
- `configuring-models.md`, `user-guide/configuring-models.md`

**Source-of-truth files (tag-pinned):**
- `hermes_cli/config.py`, `hermes_cli/security_advisories.py` (note: file may be post-tag; if so, exclude from release scope)
- `agent/redact.py`
- `hermes_constants.py`
- `hermes_cli/plugins.py` (hook list)
- `agent/i18n.py`, `locales/*.yaml`

**Tasks:**
- Verify `security.redact_secrets: true` is documented as the v0.13.0 **on-by-default** behavior with opt-out documented.
- Verify `HERMES_LANGUAGE` and the seven release locales (zh, ja, de, es, fr, uk, tr) are accurate.
- Verify hook list includes `transform_llm_output` and any other v0.13-released hooks.
- Verify Browserbase advanced env vars and any newly-documented vars actually exist at tag.
- Verify long-lived 1h prompt caching is labeled current-main, not v0.13 release (this is post-tag work).

### Agent F — Skills, optional skills, integrations, providers reference

**Files in scope (docs):**
- `reference/skills-catalog.md`, `reference/optional-skills-catalog.md`
- `skills.md`, `user-guide/skills/*`
- `providers.md`, `provider-routing.md`, `fallback-providers.md`
- `integrations/**`
- `developer-guide/adding-providers.md`, `developer-guide/adding-platform-adapters.md`

**Source-of-truth files (tag-pinned):**
- `skills/**/SKILL.md`, `optional-skills/**/SKILL.md`
- `providers/*`, `model_tools.py`, `hermes_cli/models.py`
- `~/src/hermes/hermes-agent/website/docs/reference/{skills-catalog,optional-skills-catalog}.md`

**Tasks:**
- Verify the released bundled-skills catalog matches `skills/**/SKILL.md` present at the tag (no post-release entries).
- Verify the optional-skills catalog matches `optional-skills/**/SKILL.md` at tag (includes `searxng-search`, the released finance bundle, etc.; excludes post-release entries).
- Verify provider rows in `providers.md` against the tag's `model_tools.py` / `providers/*`.
- Confirm new providers like NVIDIA NIM, Arcee, Step Plan, GMI Cloud, Azure AI Foundry, Tencent Tokenhub, LM Studio are present from the v0.12 → v0.13 windows as appropriate.

---

## Phase 3 — Integration

- [ ] **Step 3.1:** Collect all six agent reports. For each finding compute: severity (release-claim-blocking | doc-accuracy | nit), affected file(s), recommended edit, source citation.

- [ ] **Step 3.2:** De-duplicate cross-agent overlaps (e.g. an env var may be cited by both Agent D and Agent E).

- [ ] **Step 3.3:** Group findings by docs file. Produce an ordered list of file → patch sets.

- [ ] **Step 3.4:** Identify any finding that requires **adding** rather than amending a doc page. Decide whether the new page belongs in `user-guide/`, `features/`, `developer-guide/`, `reference/`, or `messaging/`, and whether a flat-root mirror is also needed (the standalone repo mirrors first-party docs in both nested and flat locations).

- [ ] **Step 3.5:** Produce a written "integrated patch list" inline in this plan via TaskUpdate metadata (not as a separate doc — the audit artifacts in Phase 5 capture the final state).

---

## Phase 4 — Apply documentation edits

Edits are applied **only** to `~/src/hermes/hermes-agent-docs`. The source repo `~/src/hermes/hermes-agent` is read-only here.

Strategy: for non-overlapping doc files, dispatch parallel edit-agents. For files touched by multiple agents (e.g. `reference/environment-variables.md`), do a single in-thread sequential edit so changes compose cleanly.

- [ ] **Step 4.1:** Edit single-owner files in parallel batches of up to 4 agents at once.
  - Per agent: file path, exact unified diff or Edit instructions, source citations, no other-file edits permitted.

- [ ] **Step 4.2:** Edit multi-owner files in-thread sequentially (Read → Edit → Read → Edit pattern). Files most likely to fall in this bucket:
  - `reference/environment-variables.md`
  - `reference/tools-reference.md`
  - `reference/cli-commands.md`
  - `changelog.md` (only if Phase 1 sanity check flagged drift)

- [ ] **Step 4.3:** Run `grep -E "v0\.12\.0|v2026\.4\.30" ~/src/hermes/hermes-agent-docs/{changelog,README,index}.md` to ensure no stale current-stable references remain. **Expected:** matches only in historical/changelog sections, never in current-stable surfaces.

- [ ] **Step 4.4:** Re-grep `~/src/hermes/hermes-agent-docs/**` for any `v0.14` / `unreleased` mentions to ensure no post-tag features sneak into release-scoped sections.

---

## Phase 5 — Generate v2026.5.13 audit artifacts

Five files, all dated `v2026.5.13`, following the established naming convention. Each one must distinguish source-backed facts from gaps.

- [ ] **Step 5.1:** Write `release_evaluation_v2026.5.13.md`.
  - Stable release: `v0.13.0` / `v2026.5.7` (unchanged — releases are immutable).
  - Local tag commit: `498bfc7bc12a937621b4215312049b1000726df3`.
  - Local `main` after this sync: actual current SHA (run `git rev-parse HEAD` in source repo at sync time).
  - List doc impact areas updated.
  - Note residual risk.

- [ ] **Step 5.2:** Write `source_validation_matrix_v2026.5.13.md`.
  - Reuse the matrix template from `v2026.5.7`. For each claim, refresh the "Source Evidence" column with a tag-pinned path (e.g. `git show v2026.5.7:agent/plugin_llm.py`) and the "Docs Updated" column with files actually edited in this run.
  - Add a "Current-Main Documentation Sync" section listing post-tag doc-correctness items applied this round.

- [ ] **Step 5.3:** Write `repo_review_v2026.5.13.md`.
  - Sections: Reviewed Areas, Findings Resolved, Source Evidence, Files Updated.

- [ ] **Step 5.4:** Write `documentation_integrity_findings_v2026.5.13.md`.
  - Numbered list of findings in the same style as v2026.5.7's 21-item list.
  - "Resolved Findings" (this run's fixes) and "Deferred / Watch Items" (anything left).

- [ ] **Step 5.5:** `review_plan_v2026.5.13.md` — **already written (this file)**. Confirm it's saved.

---

## Phase 6 — Verify

- [ ] **Step 6.1:** `cd ~/src/hermes/hermes-agent-docs && git status` — confirm only this repo's files are changed.

- [ ] **Step 6.2:** `git diff --check` — no whitespace errors.

- [ ] **Step 6.3:** `git diff --stat` — sanity-check the volume of changes is consistent with the integrated patch list.

- [ ] **Step 6.4:** Manually re-read top of `changelog.md`, `README.md`, `index.md` to confirm stable-release wording is intact.

- [ ] **Step 6.5:** Grep `grep -lE "498bfc7bc12a937621b4215312049b1000726df3|v2026\.5\.7"` to confirm v2026.5.13 audit artifacts reference the tag correctly.

---

## Phase 7 — Commit and push

- [ ] **Step 7.1:** Stage only `~/src/hermes/hermes-agent-docs` changes:
  ```
  cd ~/src/hermes/hermes-agent-docs && git add -A
  ```
  (The docs repo is small and self-contained; `git add -A` is safe here. Verify `git status` shows no surprises before staging.)

- [ ] **Step 7.2:** Commit with a single conventional message:
  ```
  docs: v2026.5.13 release-bound sync for v0.13.0 (v2026.5.7)
  ```
  with body explaining: re-validated against tag `v2026.5.7`, refreshed audit artifact set, captured post-tag doc-correctness items.

- [ ] **Step 7.3:** `git push origin main`.

- [ ] **Step 7.4:** Confirm push succeeded by reading `git status` (clean tree, ahead by 0).

---

## Self-Review (auto-applied; embedded for the executing agent)

**Spec coverage:**
- User said: "Fetch and pull all latest changes from main origins in `~/src/hermes/hermes-agent`." → Phase 0.
- User said: "Create detailed Plan to review … all file and source code and documentation." → Phases 1-2 cover release surface, all source-mapped doc files, with explicit file lists.
- User said: "focus on released versions and latest changelog only." → All claims bounded to tag `v2026.5.7`; post-tag treated only as doc-correctness, never as release claim.
- User said: "Make sure information is factual and correct, check online if needed." → Agent A includes the GitHub Releases online cross-check; other agents reference tag-pinned source for ground truth.
- User said: "Scope is to keep `~/src/hermes/hermes-agent-docs` up to date with latest changes in `~/src/hermes/hermes-agent`." → Phases 3-4; only the docs repo is modified.
- User said: "Spin multiple agents and subagents for different session of the code." → Phase 2 dispatches six parallel domain agents; Phase 4 optionally dispatches parallel edit agents.
- User said: "Once Plan is finalized review the plan." → This Self-Review section + a deliberate user-facing checkpoint before Phase 2 dispatch.
- User said: "Execute the plan and implement the changes." → Phases 4-5.
- User said: "Commit and push updated docs." → Phase 7.

**Placeholder scan:** No "TBD" / "later" / "appropriate" wording. Each step names exact files or exact commands.

**Type / naming consistency:** Tag SHA `498bfc7bc12a937621b4215312049b1000726df3` is used everywhere; release tag `v2026.5.7`; version label `v0.13.0`; audit-artifact suffix `_v2026.5.13.md`; new commit pin recorded at execution time.

---

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| Parallel edit agents collide on the same file | Phase 4 separates single-owner from multi-owner files explicitly. |
| Post-tag features get re-labeled as released | Hard constraint in every agent prompt + Phase 4.4 grep guard. |
| Phase 2 agents return contradictory findings | Phase 3 integration step de-dupes and reconciles; for conflicts, tag-pinned source wins. |
| Docs repo has uncommitted local work | Phase 0 verified the docs repo is clean. Phase 7.1 re-checks `git status` before staging. |
| Online cross-check fails (rate-limit / offline) | Agent A is instructed to record the GitHub Releases check; if it can't reach GitHub, it must note "online check unavailable" rather than silently skipping. |

---

## Execution handoff

This plan is **release-bounded**, **file-oriented**, **parallel-friendly**, and **artifact-preserving**. After the human reviews and approves, execution proceeds Phase 1 → Phase 7. Phase 2 is the single parallel-dispatch step; everything else is sequential.
