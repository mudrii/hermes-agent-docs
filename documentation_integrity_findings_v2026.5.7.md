# Documentation Integrity Findings: v2026.5.7

Date reviewed: 2026-05-10

## Resolved Findings

1. **Stable release marker was stale.** `README.md` and `changelog.md` still pointed to v0.12.0. They now point to v0.13.0 / v2026.5.7.
2. **Release audit artifacts were missing.** Added v2026.5.7 review, release evaluation, repo review, source validation matrix, and integrity findings.
3. **Messaging docs omitted new platforms.** Added LINE and Google Chat docs and refreshed messaging indexes.
4. **Teams meeting workflow docs were missing.** Added Teams meeting pipeline and Microsoft Graph webhook setup docs.
5. **Plugin docs omitted released API surface.** Added plugin LLM access, model-provider plugin, and image-generation provider plugin docs.
6. **Kanban docs were absent.** Added durable Kanban feature docs and tutorial pages.
7. **Browser docs missed released eval behavior.** Refreshed browser docs and corrected CDP availability wording.
8. **Environment-variable references were incomplete.** Refreshed environment-variable docs and added `HERMES_LANGUAGE` plus Browserbase advanced knobs.
9. **i18n docs undercounted locales.** Updated configuration docs to list all 16 supported locale codes.
10. **Dashboard plugin API auth statement was wrong.** Corrected docs to state that plugin API routes use dashboard API authentication.

## Deferred / Watch Items

- Generated skills catalogs were refreshed from first-party docs where copied, but individual skill content was not exhaustively revalidated line by line.
- Some release-note items, such as every individual security fix PR, are summarized in docs rather than expanded into dedicated pages.
- First-party source docs should also receive the dashboard plugin auth, `HERMES_LANGUAGE`, Browserbase env, and CDP wording corrections in a future upstream documentation PR.
