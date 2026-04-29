# Source Validation Matrix: v2026.4.16..v2026.4.23

| Source release surface | Docs repo target | Status |
|------------------------|------------------|--------|
| `RELEASE_v0.11.0.md` | `changelog.md`, `README.md` | Updated |
| `agent/transports/{anthropic,chat_completions,responses_api,bedrock}.py` | `architecture.md`, `developer-guide/architecture.md`, `developer-guide/agent-loop.md`, `developer-guide/transport-layer.md` | Updated |
| `website/docs/user-guide/tui.md`, `ui-tui/`, `tui_gateway/` | `tui.md` | Added |
| `website/docs/user-guide/features/built-in-plugins.md`, `plugins/builtin/` | `built-in-plugins.md` | Added |
| `website/docs/guides/github-pr-review-agent.md` | `guides/github-pr-review-agent.md` | Added |
| `website/docs/guides/webhook-github-pr-review.md` | `guides/webhook-github-pr-review.md` | Added |
| `plugins/image_gen/{openai,xai,openai-codex}/plugin.yaml`, `tools/image_generation_tool.py`, `agent/image_gen_provider.py` | `image-generation.md` | Updated |
| `tools/delegate_tool.py`, `hermes_cli/config.py` (`orchestrator_enabled` default) | `delegation.md` | Updated |
| `agent/credential_pool.py` (`EXHAUSTED_TTL_DEFAULT_SECONDS`) | `credential-pools.md` | Updated |
| `hermes_cli/cli.py` (`completion` shells, `skills reset` subcommand) | `reference/cli-commands.md` | Updated |
| `tools/registry.py`, `tools/browser_tools/*.py` | `reference/tools-reference.md` | Updated |
| Bundled + optional skill manifests (59 optional + 72 bundled) | `reference/optional-skills-catalog.md`, `reference/skills-catalog.md` | Updated |
| `agent/redact.py` (35 prefix patterns + generic forms) | `security.md` | Updated |
| `gateway/platforms/api_server.py` (`X-Hermes-Session-Id`, `Idempotency-Key`) | `api-server.md` | Updated |
| `agent/redact.py` (`HERMES_REDACT_SECRETS` env), import-time snapshot | `reference/environment-variables.md` | Updated |
| `gateway/platforms/qqbot/adapter.py`, QR setup wizard | `messaging/qqbot.md`, `messaging/index.md` | Updated |
| `gateway/webhook/*.py` (direct-delivery mode) | `messaging/webhooks.md` | Updated |
| `web/dashboard/themes/*`, `agent/dashboard.py` (six built-ins, `dashboard.theme`) | `web-dashboard.md`, `dashboard-plugins.md` | Updated |
| `agent/voice_tools.py`, `tools/voice_mode.py`, `gateway/stt.py` (`stt.enabled`, push-to-talk) | `voice-mode.md`, `tts.md`, `messaging/open-webui.md` | Verified |
| `agent/cron.py` (`wakeAgent`, per-job `enabled_toolsets`) | `cron.md` | Updated |
| `tools/honcho_tools.py` (5-tool surface) | `honcho.md` | Verified |
| `hermes_cli/setup_*.py` (NVIDIA NIM, Arcee AI, Step Plan, Gemini CLI OAuth, Vercel ai-gateway, Codex GPT-5.5) | `providers.md`, `integrations/providers.md`, `developer-guide/adding-providers.md` | Updated |
| `agent/bedrock_adapter.py` (Converse API, IAM, inference profiles, Guardrails) | `guides/aws-bedrock.md` | Verified |
| `tools/skill_*.py` (`hermes skills reset`, namespaced skill bundles, `xitter` → `xurl`) | `skills.md`, `reference/skills-catalog.md` | Updated |
| `tools/plugin_loader.py` (`register_command`, `dispatch_tool`, `pre_tool_call`, `transform_tool_result`, `transform_terminal_output`) | `plugins.md`, `hooks.md`, `guides/build-a-hermes-plugin.md` | Updated |
| `hermes_cli/cli.py` (`--ignore-user-config`, `--ignore-rules`, Doctor "Command Installation" check) | `reference/cli-commands.md`, `getting-started/installation.md`, `getting-started/updating.md` | Updated |
| `hermes_cli/config.py` (`delegation.max_spawn_depth`, `delegation.orchestrator_enabled`, `dashboard.theme`, `agent.api_max_retries`, `skills.guard_agent_created`, `stt.enabled`, `privacy.redact_pii`, `security.redact_secrets`, `mcp_servers`) | `configuration.md`, `reference/environment-variables.md`, `profiles.md` | Updated |
| `agent/cli/slash_commands.py` (`/steer` plus the existing slash command set) | `reference/slash-commands.md` | Updated |
