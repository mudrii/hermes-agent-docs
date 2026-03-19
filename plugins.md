# Plugins

The plugin system is the primary extension mechanism in Hermes Agent, introduced as a first-class feature in v0.3.0. Plugins let you add custom tools, slash commands, lifecycle hooks, and bundled skills without forking or modifying core Hermes code.

A plugin is a directory dropped into `~/.hermes/plugins/`. When Hermes starts, it discovers and loads all plugins automatically. Your tools appear alongside built-in tools immediately — the model can call them without any extra configuration.

## Plugin Directory Structure

Every plugin is a directory with at minimum a `plugin.yaml` manifest and an `__init__.py` registration module. The conventional layout for a complete plugin is:

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml      # manifest — declares what the plugin provides
├── __init__.py      # register() — wires schemas to handlers and hooks
├── schemas.py       # tool schemas (what the LLM reads to decide when to call)
└── tools.py         # tool handlers (the code that runs when the LLM calls)
```

Additional files like `data/`, `skill.md`, or any supporting modules are allowed and can be read at import time using `Path(__file__).parent`.

## Plugin Manifest (plugin.yaml)

The manifest declares the plugin's identity and optional requirements:

```yaml
name: my-plugin
version: 1.0.0
description: A description of what this plugin does
author: Your Name

provides:
  tools: true
  hooks: true

requires_env:        # optional: gate plugin loading on env vars
  - MY_API_KEY       # plugin is disabled if this variable is not set
```

If `requires_env` lists any variable that is missing from the environment, the plugin is skipped at load time with a clear message. No crash, no error in the agent — just a notice that the plugin is disabled due to a missing key.

## What Plugins Can Do

| Capability | How |
|-----------|-----|
| Add custom tools | `ctx.register_tool(name, schema, handler)` |
| Register lifecycle hooks | `ctx.register_hook("post_tool_call", callback)` |
| Ship data files | `Path(__file__).parent / "data" / "file.yaml"` |
| Bundle skills | Copy `skill.md` to `~/.hermes/skills/` at load time |
| Gate on env vars | `requires_env: [API_KEY]` in `plugin.yaml` |
| Conditional tool availability | `check_fn=lambda: _has_optional_lib()` in `register_tool` |
| Distribute via pip | `[project.entry-points."hermes_agent.plugins"]` in `pyproject.toml` |

## Plugin Discovery

Hermes discovers plugins from three sources, checked in order:

| Source | Path | Use case |
|--------|------|----------|
| User plugins | `~/.hermes/plugins/` | Personal plugins |
| Project plugins | `.hermes/plugins/` | Project-specific plugins |
| pip entry points | `hermes_agent.plugins` entry_points group | Distributed packages |

Each source can have multiple plugins. All discovered plugins are loaded on startup.

## Registration Module (__init__.py)

The `register(ctx)` function is the plugin entry point. It is called exactly once at startup. If it raises an exception, the plugin is disabled but Hermes continues normally.

```python
"""My plugin — registration."""

import logging
from . import schemas, tools

logger = logging.getLogger(__name__)

def register(ctx):
    """Wire schemas to handlers and register hooks."""
    ctx.register_tool(
        name="my_tool",
        toolset="my-plugin",
        schema=schemas.MY_TOOL,
        handler=tools.my_tool_handler,
    )
    ctx.register_hook("post_tool_call", _on_post_tool_call)

def _on_post_tool_call(tool_name, args, result, task_id, **kwargs):
    logger.debug("Tool called: %s", tool_name)
```

`ctx.register_tool` parameters:

| Parameter | Description |
|-----------|-------------|
| `name` | Tool name (must be unique across all plugins) |
| `toolset` | Toolset group name (shown in banner) |
| `schema` | OpenAI function-calling schema dict |
| `handler` | Python callable that executes the tool |
| `check_fn` | Optional callable returning bool; `False` hides tool from model |

## Tool Schemas

Schemas are OpenAI function-calling format dicts. The `description` field is critical — the LLM reads it to decide when to use your tool.

```python
MY_TOOL = {
    "name": "my_tool",
    "description": (
        "A precise description of what this tool does and when to use it. "
        "Include examples of valid inputs."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "input": {
                "type": "string",
                "description": "What this parameter accepts",
            },
        },
        "required": ["input"],
    },
}
```

## Tool Handlers

Handlers are Python functions that execute when the LLM calls your tool:

```python
import json

def my_tool_handler(args: dict, **kwargs) -> str:
    """Handler contract:
    1. Receives args dict
    2. Does the work
    3. Returns a JSON string — always, even on error
    4. Accepts **kwargs for forward compatibility
    """
    value = args.get("input", "").strip()
    if not value:
        return json.dumps({"error": "No input provided"})

    try:
        result = do_something(value)
        return json.dumps({"result": result})
    except Exception as e:
        return json.dumps({"error": str(e)})
```

**Handler rules:**
1. Signature must be `def handler(args: dict, **kwargs) -> str`
2. Always return a JSON string — never raise, never return a dict
3. Catch all exceptions and return error JSON
4. Accept `**kwargs` so Hermes can pass additional context in the future

## Complete Example: Calculator Plugin

This example is taken from the official build-a-plugin guide and demonstrates a plugin with two tools and a hook.

### plugin.yaml

```yaml
name: calculator
version: 1.0.0
description: Math calculator — evaluate expressions and convert units
provides:
  tools: true
  hooks: true
```

### schemas.py

```python
CALCULATE = {
    "name": "calculate",
    "description": (
        "Evaluate a mathematical expression and return the result. "
        "Supports arithmetic (+, -, *, /, **), functions (sqrt, sin, cos, "
        "log, abs, round, floor, ceil), and constants (pi, e). "
        "Use this for any math the user asks about."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "expression": {
                "type": "string",
                "description": "Math expression to evaluate (e.g., '2**10', 'sqrt(144)')",
            },
        },
        "required": ["expression"],
    },
}

UNIT_CONVERT = {
    "name": "unit_convert",
    "description": (
        "Convert a value between units. Supports length (m, km, mi, ft, in), "
        "weight (kg, lb, oz, g), temperature (C, F, K), data (B, KB, MB, GB, TB), "
        "and time (s, min, hr, day)."
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "value": {"type": "number", "description": "The numeric value to convert"},
            "from_unit": {"type": "string", "description": "Source unit (e.g., 'km', 'lb', 'F')"},
            "to_unit": {"type": "string", "description": "Target unit (e.g., 'mi', 'kg', 'C')"},
        },
        "required": ["value", "from_unit", "to_unit"],
    },
}
```

### __init__.py

```python
import logging
from . import schemas, tools

logger = logging.getLogger(__name__)
_call_log = []

def _on_post_tool_call(tool_name, args, result, task_id, **kwargs):
    _call_log.append({"tool": tool_name, "session": task_id})
    if len(_call_log) > 100:
        _call_log.pop(0)
    logger.debug("Tool called: %s (session %s)", tool_name, task_id)

def register(ctx):
    ctx.register_tool(name="calculate", toolset="calculator",
                      schema=schemas.CALCULATE, handler=tools.calculate)
    ctx.register_tool(name="unit_convert", toolset="calculator",
                      schema=schemas.UNIT_CONVERT, handler=tools.unit_convert)
    ctx.register_hook("post_tool_call", _on_post_tool_call)
```

After starting Hermes, verify with `/plugins`:

```
Plugins (1):
  ✓ calculator v1.0.0 (2 tools, 1 hooks)
```

## Hook Registration in Plugins

Plugins can subscribe to any combination of lifecycle hooks. Hooks are observers — they cannot modify arguments or return values. If a hook crashes, it is logged and skipped.

```python
def register(ctx):
    ctx.register_hook("pre_tool_call", before_any_tool)
    ctx.register_hook("post_tool_call", after_any_tool)
    ctx.register_hook("on_session_start", on_new_session)
    ctx.register_hook("on_session_end", on_session_end)
    ctx.register_hook("pre_llm_call", before_llm)
    ctx.register_hook("post_llm_call", after_llm)
```

Available hooks in plugins:

| Hook | When | Arguments |
|------|------|-----------|
| `pre_tool_call` | Before any tool runs | `tool_name`, `args`, `task_id` |
| `post_tool_call` | After any tool returns | `tool_name`, `args`, `result`, `task_id` |
| `pre_llm_call` | Before LLM API call | `messages`, `model` |
| `post_llm_call` | After LLM response | `messages`, `response`, `model` |
| `on_session_start` | Session begins | `session_id`, `platform` |
| `on_session_end` | Session ends | `session_id`, `platform` |

## Shipping Data Files

Put any files in your plugin directory and read them at import time:

```python
from pathlib import Path
import yaml

_PLUGIN_DIR = Path(__file__).parent
_DATA_FILE = _PLUGIN_DIR / "data" / "config.yaml"

with open(_DATA_FILE) as f:
    _DATA = yaml.safe_load(f)
```

## Bundling a Skill

Include a `skill.md` in your plugin directory and install it at registration time:

```python
import shutil
from pathlib import Path

def _install_skill():
    try:
        from hermes_cli.config import get_hermes_home
        dest = get_hermes_home() / "skills" / "my-plugin" / "SKILL.md"
    except Exception:
        dest = Path.home() / ".hermes" / "skills" / "my-plugin" / "SKILL.md"

    if dest.exists():
        return  # don't overwrite user edits

    source = Path(__file__).parent / "skill.md"
    if source.exists():
        dest.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy2(source, dest)

def register(ctx):
    ctx.register_tool(...)
    _install_skill()
```

## Distributing via pip

For sharing plugins publicly, declare an entry point in your Python package:

```toml
# pyproject.toml
[project.entry-points."hermes_agent.plugins"]
my-plugin = "my_plugin_package"
```

```bash
pip install hermes-plugin-my-plugin
# Plugin auto-discovered on next hermes startup
```

## Loading Order and Discovery Details

On startup, Hermes:

1. Scans `~/.hermes/plugins/` for subdirectories containing `plugin.yaml`
2. Scans `.hermes/plugins/` in the current working directory
3. Checks installed packages for the `hermes_agent.plugins` entry points group
4. For each discovered plugin, validates `requires_env` and skips if any variable is missing
5. Calls `register(ctx)` on each valid plugin
6. Any plugin whose `register()` raises an exception is disabled; Hermes continues

## Managing Plugins

Inside a CLI session:

```
/plugins              # list loaded plugins with tool and hook counts
```

No forking or source modification is required. The plugin system is designed so that all extensions are isolated — a broken plugin never crashes Hermes.
