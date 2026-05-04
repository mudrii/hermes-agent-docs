# Source Validation Matrix: Hermes Agent v0.12.0 (v2026.4.30)

| Area | Documentation Updated | Source Evidence |
|---|---|---|
| Release framing | `README.md`, `index.md`, `changelog.md` | `RELEASE_v0.12.0.md`, tag `v2026.4.30` |
| Curator | `curator.md`, `reference/cli-commands.md`, `reference/slash-commands.md`, `skills.md` | `agent/curator.py`, `hermes_cli/curator.py`, `hermes_cli/config.py` |
| Providers/model runtime | `providers.md`, `integrations/providers.md`, `reference/environment-variables.md`, `developer-guide/provider-runtime.md`, `configuring-models.md`, `reference/model-catalog.md` | `hermes_cli/auth.py`, `hermes_cli/runtime_provider.py`, `hermes_cli/model_catalog.py`, `hermes_cli/config.py` |
| Messaging platforms | `messaging/index.md`, `messaging/yuanbao.md`, `messaging/teams.md`, `gateway.md` | `gateway/platforms/yuanbao.py`, `plugins/platforms/teams/adapter.py`, `gateway/platform_registry.py` |
| Plugins/integrations | `built-in-plugins.md`, `plugins.md`, `spotify.md`, generated skill pages | `plugins/spotify/plugin.yaml`, `plugins/google_meet/plugin.yaml`, `plugins/observability/langfuse/plugin.yaml`, `plugins/hermes-achievements/README.md` |
| Skills catalogs | `reference/skills-catalog.md`, `reference/optional-skills-catalog.md`, `user-guide/skills/**` | `skills/**/SKILL.md`, `optional-skills/**/SKILL.md`, upstream generated `website/docs/user-guide/skills/**` |
| TTS/media | `tts.md`, `tools.md`, `toolsets.md`, `voice-mode.md`, `gateway.md` | `tools/tts_tool.py`, `gateway/platforms/base.py`, platform adapter `send_multiple_images` implementations |
| CLI/TUI/dashboard | `cli-reference.md`, `reference/cli-commands.md`, `reference/slash-commands.md`, `tui.md`, `web-dashboard.md` | `hermes_cli/_parser.py`, `hermes_cli/main.py`, `hermes_cli/commands.py`, `ui-tui/src/**`, `hermes_cli/web_server.py`, `web/src/**` |
| Image/ACP/prompt cache/security | `vision.md`, `acp.md`, `developer-guide/context-compression.md`, `security.md`, `configuration.md` | `agent/image_routing.py`, `acp_adapter/server.py`, `hermes_cli/config.py`, `run_agent.py`, `agent/redact.py` |
