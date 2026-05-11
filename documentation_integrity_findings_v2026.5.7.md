# Documentation Integrity Findings: v2026.5.7

Date reviewed: 2026-05-11

## Resolved Findings

1. **Stable release marker was stale.** `README.md` and `changelog.md` still pointed to v0.12.0. They now point to v0.13.0 / v2026.5.7.
2. **Release audit artifacts were missing.** Added v2026.5.7 review, release evaluation, repo review, source validation matrix, and integrity findings.
3. **Messaging docs omitted new release platform coverage.** Added Google Chat docs and refreshed messaging indexes.
4. **Post-release current-main docs needed separation from release claims.** LINE, `/handoff`, slash-command access control, Telegram native draft streaming, and Kanban worker lanes are documented as current-main sync items rather than v0.13.0 release contents.
5. **Plugin docs omitted released API surface.** Added plugin LLM access, model-provider plugin, and image-generation provider plugin docs.
6. **Kanban docs were absent.** Added durable Kanban feature docs and tutorial pages.
7. **Browser docs missed released eval behavior.** Refreshed browser docs and corrected CDP availability wording.
8. **Environment-variable references were incomplete.** Refreshed environment-variable docs and added `HERMES_LANGUAGE` plus Browserbase advanced knobs.
9. **i18n release claims were over-broad.** Corrected v0.13.0 artifacts to state the seven release locales from `RELEASE_v0.13.0.md`; current-main locale expansion is not treated as a v0.13.0 release fact.
10. **Dashboard plugin API auth statement was wrong.** Corrected docs to state that plugin API routes use dashboard API authentication.
11. **CLI and tool references included unreleased computer-use command/toolset rows.** Removed those rows from released references while leaving current-main computer-use pages outside the v0.13.0 release claims.
12. **Security redaction default was contradicted.** Updated configuration docs to state that `security.redact_secrets` is on by default in v0.13.0 and documented opt-out.
13. **Gateway/messaging pages had stale platform defaults and missing v0.13.0 allowlists.** Updated streaming defaults, DingTalk/Home Assistant auto-enable wording, Slack/Mattermost/Matrix/DingTalk allowlist notes, and QQBot v0.13 capability notes.
14. **Plugin/provider docs overstated or misstated released APIs.** Corrected hook coverage, memory-provider discovery, pip entry-point group, plugin LLM `agent_id` semantics, and image-generation FAL default routing.

## Deferred / Watch Items

- Generated skills catalogs were refreshed from first-party docs where copied, but individual skill content was not exhaustively revalidated line by line.
- Some release-note items, such as every individual security fix PR, are summarized in docs rather than expanded into dedicated pages.
- First-party source docs should also receive the dashboard plugin auth, `HERMES_LANGUAGE`, Browserbase env, and CDP wording corrections in a future upstream documentation PR.
