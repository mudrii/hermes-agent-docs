# Toolsets

Toolsets are logical groupings of tools for specific scenarios. They control which tools the LLM can access during a session.

---

## Individual Toolsets (25)

Each maps to one or more specific tools:

| Toolset | Tools |
|---------|-------|
| `web` | web_search, web_extract |
| `search` | web_search |
| `vision` | vision_analyze |
| `image_gen` | image_generate |
| `terminal` | terminal, process |
| `moa` | mixture_of_agents |
| `skills` | skills_list, skill_view, skill_manage |
| `browser` | All 11 browser tools |
| `cronjob` | cronjob |
| `rl` | All 10 rl_* tools |
| `file` | read_file, write_file, patch, search_files |
| `tts` | text_to_speech |
| `todo` | todo |
| `memory` | memory |
| `session_search` | session_search |
| `clarify` | clarify |
| `code_execution` | execute_code |
| `delegation` | delegate_task |
| `honcho` | honcho_profile, honcho_search, honcho_context, honcho_conclude |
| `homeassistant` | ha_list_entities, ha_get_state, ha_list_services, ha_call_service |
| `messaging` | send_message |

---

## Composite Toolsets

### `safe`

Safe set for operations without shell access:

```
image_generate, mixture_of_agents, vision_analyze, web_extract, web_search
```

Use when you want web research and vision but no terminal execution.

### `debugging`

Systematic debugging workflow:

```
patch, process, read_file, search_files, terminal, web_extract, web_search, write_file
```

---

## Platform Toolsets (9)

All messaging platforms share the same comprehensive toolset. The following toolsets are used internally when gateway dispatches to a platform:

| Toolset | Used By |
|---------|---------|
| `hermes-cli` | Interactive terminal CLI |
| `hermes-telegram` | Telegram gateway |
| `hermes-discord` | Discord gateway |
| `hermes-slack` | Slack gateway |
| `hermes-whatsapp` | WhatsApp gateway |
| `hermes-signal` | Signal gateway |
| `hermes-email` | Email gateway |
| `hermes-homeassistant` | Home Assistant gateway |
| `hermes-gateway` | Union of all platform toolsets |

Each platform toolset includes 35+ tools:
- All 11 browser tools
- File operations (4)
- Web tools (2)
- Terminal tools (2)
- Vision + image generation
- Skills management (3)
- Memory + session search
- Todo + clarify
- Code execution + delegation
- Cron scheduling
- Honcho memory (4)
- Home Assistant (4)
- Messaging delivery
- Mixture of agents
- RL tools (10)
- TTS

### Platform Differences

- **`hermes-acp`** (editor integration) — excludes: audio/TTS, clarify tool
- **`hermes-cli`** — full set including voice input

---

## Dynamic Toolsets

### MCP Toolsets

Each configured MCP server generates a toolset:

```
mcp-<server_name>
```

For example, an MCP server named `filesystem` creates toolset `mcp-filesystem`.

### Wildcard

```
all
*
```

Both expand to all registered toolsets.

---

## Using Toolsets

### In Config

```yaml
# ~/.hermes/config.yaml
toolsets: ["hermes-cli"]      # Default
```

### Via CLI Flag

```bash
hermes --toolset debugging
hermes --toolset safe
```

### Enable/Disable Individual Toolsets

```bash
# Add a toolset mid-session
/tools +browser

# Remove a toolset mid-session
/tools -rl
```

### Custom Toolsets

```python
from toolsets import create_custom_toolset

create_custom_toolset(
    name="my-workflow",
    description="Tools for my specific workflow",
    tools=["terminal", "web_search", "read_file"],
    includes=["vision"]
)
```

---

## Toolset Resolution

`resolve_toolset(name)` recursively expands composed toolsets:

```python
resolve_toolset("hermes-cli")
# → expands includes recursively
# → deduplicates tools
# → returns flat list of tool names

resolve_multiple_toolsets(["web", "terminal", "vision"])
# → merges and deduplicates

validate_toolset("debugging")
# → returns True/False

get_toolset_info("safe")
# → {"description": ..., "tools": [...], "includes": [...]}
```

---

## Conditional Tool Loading

Tools can declare availability checks. If `check_fn()` returns False (e.g., required API key missing), the tool is excluded from the LLM schema even if the toolset is enabled.

This prevents the LLM from attempting to use tools that will fail:

```python
def check_image_generate():
    return bool(os.environ.get("FAL_KEY"))

registry.register(
    name="image_generate",
    toolset="image_gen",
    check_fn=check_image_generate,
    requires_env=["FAL_KEY"],
    ...
)
```

When `FAL_KEY` is not set, `image_generate` is silently excluded from all platform toolsets.
