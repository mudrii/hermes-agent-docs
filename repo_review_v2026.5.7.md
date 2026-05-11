# Repo Review: Hermes Agent v0.13.0 Docs Sync

Date reviewed: 2026-05-11

## Reviewed Areas

- Release/changelog/version surfaces
- Messaging and gateway platforms
- Plugins, dashboard, plugin LLM access, and Kanban
- CLI, browser, tools, runtime, i18n, and environment-variable references

## Findings Resolved

- `changelog.md` and `README.md` still identified v0.12.0 / v2026.4.30 as current stable. They now identify v0.13.0 / v2026.5.7.
- Messaging docs omitted Google Chat as a v0.13.0 release platform. First-party setup docs were imported into both `user-guide/messaging/` and the flat `messaging/` mirror.
- Post-release current-main docs for LINE, `/handoff`, slash-command access control, Telegram native draft streaming, and Kanban worker lanes were synced separately and are not represented as v0.13.0 release contents.
- Plugin docs omitted `ctx.llm`, model-provider plugins, image-generation provider plugins, and the updated plugin capability table. Those pages were imported.
- Built-in plugin and dashboard docs listed removed in-repo demo plugins and missed current dashboard plugin/security behavior. First-party docs were imported and plugin API auth wording was corrected against `hermes_cli/web_server.py`.
- Kanban docs were absent. First-party Kanban feature and tutorial pages were imported into user-guide, feature, and flat mirrors.
- Browser docs missed `browser_console(expression=...)` supervisor eval behavior and had stale CDP wording. The first-party browser page was imported and CDP wording was corrected.
- Environment-variable references missed Kanban variables, `HERMES_LANGUAGE`, Browserbase advanced knobs, and several provider env vars. The reference was refreshed and patched; LINE variables are present as current-main docs.
- Configuration docs were checked against the release boundary so v0.13.0 locale claims match the seven release locales.
- CLI/config references were corrected for released v0.13.0 behavior: removed unreleased `hermes computer-use` command/toolset rows from release references, documented `hermes tools` subcommands, added missing gateway/memory command options, and corrected `security.redact_secrets` to on-by-default.
- Gateway and messaging pages were refreshed for released v0.13.0 allowlists, restart notifications, streaming defaults, DingTalk/Home Assistant auto-enable behavior, and QQBot approval/upload/quoted-attachment capabilities.
- Plugin/provider docs were corrected for released source behavior: hook list, memory-provider discovery, plugin entry-point group, plugin LLM `agent_id` semantics, and image-generation FAL default routing.

## Source Evidence

- `RELEASE_v0.13.0.md`
- `plugins/platforms/line/plugin.yaml`
- `plugins/platforms/line/adapter.py`
- `plugins/platforms/google_chat/plugin.yaml`
- `agent/plugin_llm.py`
- `hermes_cli/plugins.py`
- `plugins/kanban/dashboard/manifest.json`
- `hermes_cli/kanban_db.py`
- `tools/kanban_tools.py`
- `tools/browser_tool.py`
- `tools/browser_supervisor.py`
- `agent/i18n.py`
- `web/src/i18n/context.tsx`

## Files Updated

The sync added or refreshed first-party docs under:

- `developer-guide/`
- `user-guide/features/`
- `user-guide/messaging/`
- `reference/`
- `guides/`
- flat mirrors such as `features/`, `messaging/`, and selected root pages

Historical audit artifacts from previous releases were preserved.
