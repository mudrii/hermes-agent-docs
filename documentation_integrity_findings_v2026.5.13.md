# Documentation integrity findings — v2026.5.13

**Audit base:** docs HEAD `0007eb8f15ee54538f7fda1b123038635c06e63e`
**Source tag:** v2026.5.7 → commit `498bfc7bc12a937621b4215312049b1000726df3`

## Finding classes

This audit produced findings in four classes. Each finding lists severity, root cause, and remediation.

### Class A — factual misclaim (high severity)

A statement in the docs that **directly contradicts** the tag-state of the source code.

| ID | File:line | Old claim | Truth at tag | Remediation |
|---|---|---|---|---|
| A1 | `extending-the-dashboard.md:730` (+ 2 mirrors) | "Plugin routes are session-token gated by the dashboard." | `auth_middleware` excludes `/api/plugins/*`. | Inverted claim, added `:::warning` admonition naming the source file. |
| A2 | `web-dashboard.md:180` (+ 2 mirrors) | "The dashboard has no authentication of its own." | A session-token middleware DOES gate non-public `/api/*` — but plugin routes are exempt. | Rewrote intro paragraph to state both halves. |
| A3 | `reference/environment-variables.md:92` and `configuration.md:1223` (+ mirror) | 16-locale list | 8-locale set: `en, zh, ja, de, es, fr, tr, uk` (English fallback for unknown). | Replaced list. |
| A4 | `reference/environment-variables.md:532` | "`HERMES_SESSION_ID` is exported to tool subprocesses." | Substituted as the `${HERMES_SESSION_ID}` template token inside SKILL.md bodies during skill preprocessing (`agent/skill_preprocessing.py`). Not exported as a real env var at tag. | Narrowed wording to match preprocessing semantics. |
| A5 | `changelog.md:20` | Highlighted `ctx.llm` and "dashboard plugin API routes are covered by dashboard auth" as v0.13.0 wins. | `ctx.llm` is post-tag; plugin routes are NOT covered. | Rewrote line. |

### Class B — temporal misclaim (medium-high severity)

A feature is described as if released, but is only on `main` post-tag.

| ID | File:line | Feature | Banner pattern |
|---|---|---|---|
| B1 | `developer-guide/plugin-llm-access.md` (page) | `ctx.llm` plugin API | Page-level `:::warning`. |
| B2 | `plugins.md:114` + 2 mirrors | `ctx.llm.complete` row | Inline "**Current-main only — not in v0.13.0.**" prefix. |
| B3 | `messaging/line.md` + mirror | LINE platform plugin | Page-level `:::info`. |
| B4 | `messaging/teams-meetings.md` + mirror | `teams_pipeline` + CLI | Page-level `:::info`. |
| B5 | `computer-use.md` + 2 mirrors | Whole `computer_use` feature | Page-level `:::warning`. |
| B6 | `web-search.md` + 2 mirrors | Brave Search free + DDGS backends | ⚠ row markers + admonition. |
| B7 | `features/kanban-worker-lanes.md` + mirror | "Worker lane" terminology + non-Hermes lane plug-in contract | Page-level `:::info`. |
| B8 | `gateway.md:448` | LINE row | Inline note in row. |
| B9 | `reference/environment-variables.md:453` | Teams Meeting Summary Delivery env-var block | Section-level `:::info`. |

### Class C — completeness gap (medium severity)

A feature exists at the tag but is not documented (or only partially).

| ID | File:line | Gap | Fix |
|---|---|---|---|
| C1 | `providers.md:308` + mirror | StepFun provider missing from the API-key provider list. | Added block + base-URL entry. |
| C2 | `providers.md:308` + mirror | Azure AI Foundry provider missing. | Added block + base-URL entry. |
| C3 | `gateway.md:447` | Google Chat env var coverage incomplete (only project ID listed). | Expanded to include `GOOGLE_CHAT_SUBSCRIPTION_NAME`, `GOOGLE_CHAT_SERVICE_ACCOUNT_JSON`, `GOOGLE_CHAT_ALLOWED_USERS`, `GOOGLE_CHAT_HOME_CHANNEL`. |
| C4 | `reference/environment-variables.md` (allowlists) | Five platform allowlist env vars missing. | Added 5 rows. |
| C5 | `messaging/index.md` + mirror | Index body missed Feishu/Lark, WeCom, Weixin, BlueBubbles, QQ, IRC, Webhooks. | Updated body enumeration. |
| C6 | `mcp.md:270` | MCP idle keepalive (3-min) undocumented. | Added subsection citing `tools/mcp_tool.py` + PR #17003. |
| C7 | `mcp.md` (no anchor) | MEDIA tag placeholder behavior for tool results undocumented. | Added "Media content in MCP tool results" subsection. |

### Class D — generator drift (low severity, deferred)

Content auto-generated upstream that has drifted in cosmetics.

| ID | File:line | Issue | Decision |
|---|---|---|---|
| D1 | `reference/skills-catalog.md:67` | `kanban-orchestrator` description truncates with `...`. | NOT hand-patched. Fix upstream by editing the source skill's `description:` front-matter, then regenerate the catalog. |

## Severity totals

| Class | Count | Status |
|---|---|---|
| A — factual misclaim | 5 | All corrected |
| B — temporal misclaim | 9 | All banner-gated |
| C — completeness gap | 7 | All filled |
| D — generator drift | 1 | Deferred upstream |
| **Total** | **22** | **21 applied / 1 deferred** |

(Item count exceeds the 19-patch list because several "patches" combined two mirrored locations into one patch ID, and class B/C grouping splits them out per-file here.)

## Pattern observation

The dominant integrity failure mode in this docs repo is **temporal slip**: contributors edit docs against `main`, not the release tag, so post-tag features quietly accrete into release-described pages. Three structural mitigations would reduce future drift:

1. CI step that diffs the tag tree against doc claims using a registry of "feature → source-of-truth path."
2. Convention: every new doc page or section gets a `since:` front-matter field with the tag it lands in.
3. The `:::info Current-main only` admonition pattern (used here) should be promoted to a standard template in the docs contributor guide.

## Sign-off

This audit is complete. All Class A, B, and C findings have been applied to the docs repo. The single Class D finding is documented and routed upstream. The docs repo is ready for commit and push.

## Post-sync resolution (2026-05-13)

On 2026-05-13 the docs repo was fully resynced from `/Users/mudrii/src/hermes/hermes-agent/website/docs/` at `origin/main` HEAD, which includes the v0.13.0 release (tag `v2026.5.7`) plus all post-tag fixes on `main`. The sync modified 253 files, added 13 new files (`lsp.md`, `checkpoints.md`, `windows-native.md`, multiple `_category_.json` files, `user-stories.mdx`, 4 optional/mlops skills, `apple-macos-computer-use`, `productivity-teams-meeting-pipeline`, `creative-hyperframes`, `devops-watchers`), and deleted 60+ legacy root `.md` duplicates plus the obsolete root `features/` and `messaging/` directories.

**Source commit:** hermes-agent `origin/main` HEAD (post-tag); tag `v2026.5.7` baseline preserved in history.

### Status by finding class

- **Class A — factual misclaim: RESOLVED.** Docs now mirror upstream truth verbatim. The hand-corrected statements (plugin-route auth wording, locale list, `HERMES_SESSION_ID` semantics, changelog highlights) match the upstream source-of-truth at `main` HEAD.
- **Class B — temporal misclaim: STILL RELEVANT, but inverted.** Since the docs are now synced from `main` (not the tag), every feature previously banner-gated as "current-main only post-tag" is now in the docs as a documented, released feature relative to the live source tree. The banner pattern remains valuable for future audits where docs and source drift, but the specific B1–B9 items no longer require gating against this sync base.
- **Class C — completeness gap: RESOLVED.** A full mirror from upstream picks up every missing file, env var, provider entry, and platform page that the prior audit had to backfill by hand.
- **Class D — generator drift:** Unchanged; still routed upstream.

### New orphans (staged at `/tmp/sync-orphans/`)

7 files were identified as orphaned by the resync — they did not match a current upstream path. They are staged for review, not deleted.

| # | File | Reason |
|---|---|---|
| 1 | `dev-guide/...` (rename target #1) | Renamed upstream under `developer-guide/`; old path no longer exists in source. |
| 2 | `dev-guide/...` (rename target #2) | Same dev-guide → developer-guide rename. |
| 3 | `dev-guide/...` (rename target #3) | Same dev-guide → developer-guide rename. |
| 4 | bundled mlops skill #1 | Re-categorized upstream from `bundled/` → `optional/` skills tree. |
| 5 | bundled mlops skill #2 | Same bundled → optional re-categorization. |
| 6 | bundled mlops skill #3 | Same bundled → optional re-categorization. |
| 7 | bundled mlops skill #4 | Same bundled → optional re-categorization. |

These orphans reflect upstream restructuring (3 dev-guide path renames + 4 mlops skills moved from the bundled tier to the optional tier). They are safe to delete after a manual confirmation that the new paths cover the same content.

