# Source Validation Matrix: v2026.5.7

| Claim / Docs Area | Source Evidence | Docs Updated |
|---|---|---|
| v0.13.0 / v2026.5.7 is current stable | `RELEASE_v0.13.0.md`, tag `v2026.5.7`, GitHub Releases latest listing | `changelog.md`, `README.md`, `index.md`, `release_evaluation_v2026.5.7.md` |
| Durable multi-agent Kanban is released | `hermes_cli/kanban_db.py`, `tools/kanban_tools.py`, `plugins/kanban/dashboard/manifest.json`, release note Kanban section | `user-guide/features/kanban.md`, `features/kanban.md`, `kanban.md`, `reference/tools-reference.md`, `reference/toolsets-reference.md` |
| `/goal` is a released persistent-goal feature | `RELEASE_v0.13.0.md`, first-party `website/docs/user-guide/features/goals.md` | `user-guide/features/goals.md`, `features/goals.md`, `goals.md` |
| LINE platform exists as a platform plugin | `plugins/platforms/line/plugin.yaml`, `plugins/platforms/line/adapter.py`, `tests/gateway/test_line_plugin.py` | `user-guide/messaging/line.md`, `messaging/line.md`, `reference/environment-variables.md` |
| Google Chat is documented as the 20th platform | `plugins/platforms/google_chat/plugin.yaml`, first-party `website/docs/user-guide/messaging/google_chat.md`, release note | `user-guide/messaging/google_chat.md`, `messaging/google_chat.md`, `user-guide/messaging/index.md`, `messaging/index.md` |
| Teams meeting pipeline and Graph webhook docs exist | `plugins/teams_pipeline/plugin.yaml`, first-party Teams docs | `user-guide/messaging/teams-meetings.md`, `user-guide/messaging/msgraph-webhook.md`, flat messaging mirrors |
| Plugins can make host-owned LLM calls | `agent/plugin_llm.py`, `hermes_cli/plugins.py` | `developer-guide/plugin-llm-access.md`, `user-guide/features/plugins.md`, `features/plugins.md`, `plugins.md` |
| Provider plugins are supported/documented | Provider plugin docs and release note provider surface | `developer-guide/model-provider-plugin.md`, `developer-guide/image-gen-provider-plugin.md` |
| Browser console eval uses supervisor fast path when available | `tools/browser_tool.py`, `tools/browser_supervisor.py`, `scripts/benchmark_browser_eval.py` | `user-guide/features/browser.md`, `features/browser.md`, `browser.md` |
| Static UI messages support 16 locales | `agent/i18n.py`, `locales/*.yaml`, `web/src/i18n/context.tsx` | `user-guide/configuration.md`, `configuration.md`, `reference/environment-variables.md` |
| Web tools include SearXNG and split backend selection | `RELEASE_v0.13.0.md`, first-party `website/docs/user-guide/features/web-search.md` | `user-guide/features/web-search.md`, `features/web-search.md`, `web-search.md` |
| Dashboard plugin API routes require dashboard auth | `hermes_cli/web_server.py`, `plugins/kanban/dashboard/plugin_api.py` | `user-guide/features/extending-the-dashboard.md`, `features/extending-the-dashboard.md` |

## Online Check

GitHub Releases was checked during the audit and listed `Hermes Agent v0.13.0 (2026.5.7)` as the latest release with tag `v2026.5.7`.
