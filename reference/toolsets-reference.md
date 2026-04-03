# Toolsets Reference

Toolsets are named bundles of tools that you can enable with `hermes chat --toolsets ...`, configure per platform, or resolve inside the agent runtime.

**Toolset count:** 40 static toolsets defined in `toolsets.py` (21 core, 2 composite, 17 platform). Additional dynamic toolsets (`mcp-<server>`, plugin-provided) are generated at runtime and vary per installation.

| Toolset | Kind | Resolves to |
|---------|------|-------------|
| `browser` | core | `browser_back`, `browser_click`, `browser_close`, `browser_console`, `browser_get_images`, `browser_navigate`, `browser_press`, `browser_scroll`, `browser_snapshot`, `browser_type`, `browser_vision`, `web_search` |
| `clarify` | core | `clarify` |
| `code_execution` | core | `execute_code` |
| `cronjob` | core | `cronjob` |
| `debugging` | composite | `patch`, `process`, `read_file`, `search_files`, `terminal`, `web_extract`, `web_search`, `write_file` |
| `delegation` | core | `delegate_task` |
| `file` | core | `patch`, `read_file`, `search_files`, `write_file` |
| `hermes-acp` | platform | `browser_back`, `browser_click`, `browser_close`, `browser_console`, `browser_get_images`, `browser_navigate`, `browser_press`, `browser_scroll`, `browser_snapshot`, `browser_type`, `browser_vision`, `delegate_task`, `execute_code`, `memory`, `patch`, `process`, `read_file`, `search_files`, `session_search`, `skill_manage`, `skill_view`, `skills_list`, `terminal`, `todo`, `vision_analyze`, `web_extract`, `web_search`, `write_file` |
| `hermes-cli` | platform | `browser_back`, `browser_click`, `browser_close`, `browser_console`, `browser_get_images`, `browser_navigate`, `browser_press`, `browser_scroll`, `browser_snapshot`, `browser_type`, `browser_vision`, `clarify`, `cronjob`, `delegate_task`, `execute_code`, `ha_call_service`, `ha_get_state`, `ha_list_entities`, `ha_list_services`, `honcho_conclude`, `honcho_context`, `honcho_profile`, `honcho_search`, `image_generate`, `memory`, `mixture_of_agents`, `patch`, `process`, `read_file`, `search_files`, `send_message`, `session_search`, `skill_manage`, `skill_view`, `skills_list`, `terminal`, `text_to_speech`, `todo`, `vision_analyze`, `web_extract`, `web_search`, `write_file` |
| `hermes-api-server` | platform | Same as hermes-cli minus `clarify` and `text_to_speech`, plus `cronjob` |
| `hermes-dingtalk` | platform | Same as hermes-cli |
| `hermes-feishu` | platform | Same as hermes-cli |
| `hermes-wecom` | platform | Same as hermes-cli |
| `hermes-discord` | platform | Same as hermes-cli |
| `hermes-email` | platform | Same as hermes-cli |
| `hermes-gateway` | composite | Union of all messaging platform toolsets |
| `hermes-homeassistant` | platform | Same as hermes-cli |
| `hermes-matrix` | platform | Same as hermes-cli |
| `hermes-mattermost` | platform | Same as hermes-cli |
| `hermes-signal` | platform | Same as hermes-cli |
| `hermes-slack` | platform | Same as hermes-cli |
| `hermes-sms` | platform | Same as hermes-cli |
| `hermes-telegram` | platform | Same as hermes-cli |
| `hermes-whatsapp` | platform | Same as hermes-cli |
| `homeassistant` | core | `ha_call_service`, `ha_get_state`, `ha_list_entities`, `ha_list_services` |
| `honcho` | core | `honcho_conclude`, `honcho_context`, `honcho_profile`, `honcho_search` |
| `image_gen` | core | `image_generate` |
| `memory` | core | `memory` |
| `messaging` | core | `send_message` |
| `moa` | core | `mixture_of_agents` |
| `rl` | core | `rl_check_status`, `rl_edit_config`, `rl_get_current_config`, `rl_get_results`, `rl_list_environments`, `rl_list_runs`, `rl_select_environment`, `rl_start_training`, `rl_stop_training`, `rl_test_inference` |
| `safe` | composite | `image_generate`, `mixture_of_agents`, `vision_analyze`, `web_extract`, `web_search` |
| `search` | core | `web_search` |
| `session_search` | core | `session_search` |
| `skills` | core | `skill_manage`, `skill_view`, `skills_list` |
| `terminal` | core | `process`, `terminal` |
| `todo` | core | `todo` |
| `tts` | core | `text_to_speech` |
| `vision` | core | `vision_analyze` |
| `web` | core | `web_extract`, `web_search` |

## Toolset kinds

- **core** -- A single logical group of tools (e.g., `file` = read/write/patch/search).
- **composite** -- Combines multiple core toolsets (e.g., `debugging` = file + terminal + web).
- **platform** -- The default tool set for a specific runtime surface (CLI, Telegram, Discord, etc.). Platform toolsets include the full shared tool list (`_HERMES_CORE_TOOLS` in `toolsets.py`) which covers web, terminal, file, vision, image generation, MoA, skills, browser, TTS, memory, session search, clarify, code execution, delegation, cronjob, messaging, honcho, and Home Assistant tools.

## Dynamic toolsets

- `mcp-<server>` -- generated at runtime for each configured MCP server.
- Custom toolsets can be created in configuration and resolved at startup.
- Wildcards: `all` and `*` expand to every registered toolset.

## Usage examples

```bash
# Start a session with only web and terminal tools
hermes chat --toolsets web,terminal

# Use the safe toolset (no terminal, no file writes)
hermes chat --toolsets safe

# Enable everything
hermes chat --toolsets all

# List available toolsets during a session
/toolsets
```
