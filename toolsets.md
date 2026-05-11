# Toolsets

Toolsets are named bundles of tools. They control which tools the AI can access during a session. Toolsets can be enabled or disabled per platform, composed from other toolsets, and extended with custom definitions.

## What Toolsets Are

A toolset is a named collection of tool names with a description. Toolsets serve as the abstraction between "what the user wants to enable" and "which specific functions the model can call."

When you run `hermes chat --toolsets "web,terminal"`, Hermes resolves those two toolset names to their constituent tool names, filters the registry for tools that pass their `check_fn`, and provides the resulting schemas to the model.

Toolsets are defined in `toolsets.py` as a Python dict. Each entry has:
- `description` — human-readable summary
- `tools` — list of tool names directly in this toolset
- `includes` — list of other toolset names to compose from (resolved recursively, with cycle detection)

Three kinds of toolsets exist:
- **Core** — single-purpose toolsets (e.g., `web`, `terminal`, `file`)
- **Composite** — toolsets that include other toolsets (e.g., `debugging`, `safe`)
- **Platform** — full-featured toolsets for a specific deployment surface (e.g., `hermes-cli`, `hermes-telegram`)

## Core Toolsets

| Toolset | Tools | Description |
|---------|-------|-------------|
| `browser` | `browser_back`, `browser_cdp`, `browser_click`, `browser_console`, `browser_dialog`, `browser_get_images`, `browser_navigate`, `browser_press`, `browser_scroll`, `browser_snapshot`, `browser_type`, `browser_vision`, `web_search` | Browser automation for web interaction (navigate, click, type, scroll, iframes, hold-click) with web search for finding URLs. v0.11.0 adds CDP-gated tools such as `browser_cdp` and `browser_dialog`. |
| `clarify` | `clarify` | Ask the user clarifying questions (multiple-choice or open-ended) |
| `code_execution` | `execute_code` | Run Python scripts that call tools programmatically (reduces LLM round trips) |
| `cronjob` | `cronjob` | Cronjob management — create, list, update, pause, resume, remove, and trigger scheduled tasks. v0.11.0 jobs accept `wakeAgent: true|false` and per-job `enabled_toolsets` to bound token overhead. See [cron.md](./cron.md). |
| `delegation` | `delegate_task` | Spawn subagents with isolated context for complex subtasks |
| `file` | `patch`, `read_file`, `search_files`, `write_file` | File manipulation — read, write, patch (with fuzzy matching), and search (content + files) |
| `homeassistant` | `ha_call_service`, `ha_get_state`, `ha_list_entities`, `ha_list_services` | Home Assistant smart home control and monitoring |
| `image_gen` | `image_generate` | Creative image generation. v0.11.0 makes the backend pluggable: ships `openai`, `openai-codex`, and `xai` plugins under `plugins/image_gen/` alongside the legacy FAL backend (FLUX 2 Pro plus Recraft V4 Pro, Nano Banana Pro, GPT Image 2, etc., via the multi-model picker). |
| `memory` | `memory` | Persistent memory across sessions (personal notes and user profile) |
| `messaging` | `send_message` | Cross-platform messaging — send messages to Telegram, Discord, Slack, SMS, etc. |
| `moa` | `mixture_of_agents` | Advanced reasoning via multiple frontier LLMs collaborating |
| `rl` | `rl_check_status`, `rl_edit_config`, `rl_get_current_config`, `rl_get_results`, `rl_list_environments`, `rl_list_runs`, `rl_select_environment`, `rl_start_training`, `rl_stop_training`, `rl_test_inference` | RL training tools for running reinforcement learning on Tinker-Atropos |
| `search` | `web_search` | Web search only (no content extraction/scraping) |
| `session_search` | `session_search` | Search and recall past conversations with summarization |
| `skills` | `skill_manage`, `skill_view`, `skills_list` | Access, create, edit, and manage skill documents |
| `terminal` | `process`, `terminal` | Terminal/command execution and process management |
| `todo` | `todo` | Task planning and tracking for multi-step work |
| `tts` | `text_to_speech` | Text-to-speech — convert text to audio. Providers: Edge TTS (free), ElevenLabs, OpenAI, MiniMax, **Mistral Voxtral**, **Google Gemini TTS** (v0.11.0), **KittenTTS** local (v0.11.0), **xAI TTS** (v0.11.0), and NeuTTS (local, free) |
| `vision` | `vision_analyze` | Image analysis and vision tools |
| `web` | `web_extract`, `web_search` | Web research and content extraction tools |

> **v0.8.0:** The `honcho` toolset was removed. Honcho is now a memory provider plugin — its tools are injected by `MemoryManager` rather than the toolset system. See [honcho.md](./honcho.md) or [memory.md](./memory.md).

## Composite Toolsets

Composite toolsets include other toolsets via the `includes` field. They are resolved recursively — if `A` includes `B` and `B` includes `C`, resolving `A` yields all tools from `A`, `B`, and `C`.

| Toolset | Kind | Resolves to |
|---------|------|-------------|
| `debugging` | composite | `patch`, `process`, `read_file`, `search_files`, `terminal`, `web_extract`, `web_search`, `write_file` |
| `safe` | composite | `image_generate`, `vision_analyze`, `web_extract`, `web_search` |

`debugging` includes `web` and `file` in addition to its direct tools (`terminal`, `process`). `safe` includes `web`, `vision`, and `image_gen` — it does not include `mixture_of_agents`.

## Platform Toolsets

Platform toolsets are large all-in-one toolsets for specific deployment surfaces. All messaging platform toolsets share the same core tool list (`_HERMES_CORE_TOOLS`), which includes every tool that ships with Hermes. The `hermes-acp` toolset is a coding-focused subset for editor integrations.

### `_HERMES_CORE_TOOLS` (shared by all messaging platform toolsets)

```
web_search, web_extract,
terminal, process,
read_file, write_file, patch, search_files,
vision_analyze, image_generate,
skills_list, skill_view, skill_manage,
browser_navigate, browser_snapshot, browser_click, browser_type,
browser_scroll, browser_back, browser_press, browser_cdp, browser_dialog,
browser_get_images, browser_vision, browser_console,
text_to_speech,
todo, memory,
session_search,
clarify,
kanban_show, kanban_complete, kanban_block, kanban_heartbeat,
kanban_comment, kanban_create, kanban_link,
execute_code, delegate_task,
cronjob,
send_message,
ha_list_entities, ha_get_state, ha_list_services, ha_call_service
```

Tools gated on environment variables or runtime conditions (`send_message`, `ha_*`, `image_generate`, `mixture_of_agents`) are silently excluded when their `check_fn` returns `False`.

| Toolset | Kind | Description |
|---------|------|-------------|
| `hermes-acp` | platform | Editor integration (VS Code, Zed, JetBrains) — coding-focused tools without messaging, audio, or clarify UI. Includes `web_search`, `web_extract`, `terminal`, `process`, file tools, `vision_analyze`, skills tools, all browser tools, `todo`, `memory`, `session_search`, `execute_code`, `delegate_task`. |
| `hermes-api-server` | platform | OpenAI-compatible API server — full agent tools accessible via HTTP without interactive UI tools (`clarify`, `send_message`). Added in v0.5.0 (PR [#3304](https://github.com/NousResearch/hermes-agent/pull/3304)). |
| `hermes-cli` | platform | Full interactive CLI toolset — all default tools including cronjob management. Uses `_HERMES_CORE_TOOLS`. |
| `hermes-discord` | platform | Discord bot toolset — full access with dangerous command approval. Same tools as `hermes-cli`. |
| `hermes-email` | platform | Email bot toolset (IMAP/SMTP). Same tools as `hermes-cli`. |
| `hermes-homeassistant` | platform | Home Assistant bot toolset — smart home event monitoring and control. Same tools as `hermes-cli`. |
| `hermes-qqbot` | platform | QQBot toolset (17th supported messaging platform, v0.11.0) — QQ Official API v2 adapter. Same tools as `hermes-cli`. |
| `hermes-signal` | platform | Signal bot toolset — encrypted messaging, full access. Same tools as `hermes-cli`. |
| `hermes-slack` | platform | Slack bot toolset — full access for workspace use. Same tools as `hermes-cli`. |
| `hermes-sms` | platform | SMS bot toolset via Twilio. Same tools as `hermes-cli`. |
| `hermes-telegram` | platform | Telegram bot toolset — full access for personal use. Same tools as `hermes-cli`. |
| `hermes-whatsapp` | platform | WhatsApp bot toolset. Same tools as `hermes-cli`. |
| `hermes-gateway` | composite | Union of all messaging platform toolsets. See [Toolsets Reference](reference/toolsets-reference.md) for the generated current list, including newer platform toolsets such as BlueBubbles, Mattermost, Matrix, DingTalk, Feishu, WeCom, Weixin, Webhook, Yuanbao, and QQBot. |

## Plugin Toolsets (v0.5.0)

Plugin-provided toolsets are now visible in `hermes tools` and standalone processes. Prior to v0.5.0 (PR [#3457](https://github.com/NousResearch/hermes-agent/pull/3457)), plugin tools registered successfully but their toolset entries were invisible to the toolset TUI and command-line listing.

The `get_plugin_toolsets()` function in `hermes_cli/plugins.py` groups all plugin-registered tool names by their toolset key, resolves the plugin description, and returns `(key, label, description)` tuples. Each plugin toolset is prefixed with a plug icon in the `hermes tools` display.

Plugin toolsets appear under a separate "Plugin Toolsets" section in `hermes tools` and can be enabled or disabled per platform in `config.yaml` using the toolset name, exactly like built-in toolsets:

```yaml
toolsets:
  enabled:
    - my-plugin        # toolset name declared in ctx.register_tool(..., toolset="my-plugin")
```

Plugin tools are discovered after built-in tools and MCP tools. The discovery order in `model_tools.py` is:

1. `_discover_tools()` — all core tool modules
2. `discover_mcp_tools()` — MCP server tools
3. `discover_plugins()` — plugin tools (registered via `ctx.register_tool`)

Toolset resolution in `get_plugin_toolsets()` runs after all three discovery steps complete, ensuring plugin toolsets are fully populated before they are read.

## Dynamic Toolsets

Two kinds of toolsets are created at runtime rather than defined statically in `toolsets.py`:

### `mcp-<server>`

When an MCP server successfully connects and contributes at least one registered tool, Hermes creates a toolset named `mcp-<server>` (e.g., `mcp-github`, `mcp-filesystem`). This allows targeting a specific MCP server by its toolset name in platform overrides.

MCP tools are also injected into all `hermes-*` platform toolsets so they are automatically available in default sessions.

### Custom toolsets at runtime

The `create_custom_toolset()` function in `toolsets.py` adds a new entry to `TOOLSETS` at runtime:

```python
from toolsets import create_custom_toolset

create_custom_toolset(
    name="my_custom",
    description="My custom toolset for specific tasks",
    tools=["web_search"],
    includes=["terminal", "vision"]
)
```

### Wildcards

The special names `all` and `*` expand to every tool across every registered toolset:

```bash
hermes chat --toolsets all
```

## Toolset Distributions

`toolset_distributions.py` defines **distributions** — probability-based toolset selection configurations used during batch data generation runs. A distribution maps toolset names to their probability (%) of being included for any given prompt.

Distributions are not used during normal interactive chat — they are exclusively for batch data generation where you want stochastic variation in tool availability.

| Distribution | Description |
|-------------|-------------|
| `default` | All available tools 100% of the time (web, vision, image_gen, terminal, file, moa, browser all at 100%) |
| `image_gen` | Heavy focus on image generation — image_gen/vision at 90%, web at 55%, terminal at 45%, moa at 10% |
| `research` | Web research focus — web at 90%, browser at 70%, vision at 50%, moa at 40%, terminal at 10% |
| `science` | Scientific research — web/terminal/file at 94%, vision at 65%, browser at 50%, image_gen at 15%, moa at 10% |
| `development` | Development tasks — terminal/file at 80%, moa at 60%, web at 30%, vision at 10% |
| `safe` | All tools except terminal — web at 80%, browser at 70%, vision/image_gen at 60%, moa at 50% |
| `balanced` | Equal probability of all toolsets — all at 50% |
| `minimal` | Only web tools — web at 100% |
| `terminal_only` | Terminal and file tools — terminal/file at 100% |
| `terminal_web` | Terminal and file with web — terminal/file/web all at 100% |
| `creative` | Image generation and vision — image_gen/vision at 90%, web at 30% |
| `reasoning` | Heavy mixture of agents — moa at 90%, web at 30%, terminal at 20% |
| `browser_use` | Full browser interaction — browser at 100%, web at 80%, vision at 70% |
| `browser_only` | Only browser tools — browser at 100% |
| `browser_tasks` | Browser-focused with occasional vision/terminal — browser at 97%, terminal at 15%, vision at 12% |
| `terminal_tasks` | Terminal-focused — terminal/file/web all at 97%, browser at 75%, vision at 50%, image_gen at 10% |
| `mixed_tasks` | Mixed browser + terminal — browser/terminal/file all at 92%, web at 35%, vision/image_gen at 15% |

Each toolset in a distribution is sampled independently based on its probability. Multiple toolsets can be active simultaneously. If no toolsets are selected (possible with very low probabilities), the distribution ensures at least the highest-probability toolset is always included.

## How to Configure Toolsets per Platform

### From the command line

```bash
# Enable specific toolsets
hermes chat --toolsets "web,terminal"

# Enable all tools
hermes chat --toolsets all

# See all available tools
hermes tools
```

### In config.yaml

```yaml
# ~/.hermes/config.yaml
platforms:
  cli:
    toolsets:
      - hermes-cli
  telegram:
    toolsets:
      - hermes-telegram
      - mcp-github          # Include a specific MCP server toolset
```

### Enabling or disabling individual toolsets

```yaml
toolsets:
  enabled:
    - web
    - terminal
    - file
  disabled:
    - rl
    - moa
```

## How to Create a Custom Toolset

### In toolsets.py (permanent, committed to repo)

Add a new entry to the `TOOLSETS` dict:

```python
TOOLSETS["my-workflow"] = {
    "description": "Tools for my specific workflow",
    "tools": ["web_search", "read_file", "patch"],
    "includes": ["terminal"]     # Also includes terminal and process
}
```

### At runtime via create_custom_toolset()

```python
from toolsets import create_custom_toolset

create_custom_toolset(
    name="my_custom",
    description="Custom toolset for a specific task",
    tools=["web_search"],
    includes=["terminal", "vision"]
)
```

This is what Hermes uses internally when registering MCP server toolsets.

### Via config.yaml (without code changes)

Custom toolsets can be defined in `~/.hermes/config.yaml` and resolved at startup:

```yaml
custom_toolsets:
  research-heavy:
    description: "Research toolset with all web and vision tools"
    includes:
      - web
      - browser
      - vision
```

## Toolset Resolution

The `resolve_toolset(name)` function in `toolsets.py` resolves a toolset name to a flat list of all tool names, recursively following `includes`:

```python
from toolsets import resolve_toolset, resolve_multiple_toolsets

# Resolve one toolset
tools = resolve_toolset("debugging")
# Returns: ['terminal', 'process', 'web_search', 'web_extract',
#           'read_file', 'write_file', 'patch', 'search_files']

# Resolve multiple toolsets (deduplicated)
tools = resolve_multiple_toolsets(["web", "vision", "terminal"])
```

Cycle detection prevents infinite recursion if a toolset transitively includes itself.

## Toolset Reference Summary

The complete set of named toolsets available in a default Hermes installation, taken directly from `toolsets.py`:

```
Core:
  browser, clarify, code_execution, cronjob, delegation, file,
  homeassistant, image_gen, memory, messaging, moa, rl,
  search, session_search, skills, terminal, todo, tts, vision, web

Composite:
  debugging, safe

Platform:
  hermes-acp, hermes-api-server, hermes-cli, hermes-discord, hermes-email,
  hermes-gateway, hermes-homeassistant, hermes-qqbot, hermes-signal, hermes-slack,
  hermes-sms, hermes-telegram, hermes-whatsapp

Dynamic (created at runtime):
  mcp-<server>    # one per connected MCP server
  custom toolsets from create_custom_toolset()
```

Wildcards: `all` and `*` expand to every registered toolset.
