# Plugins

The plugin system is the primary extension mechanism in Hermes Agent, introduced as a first-class feature in v0.3.0. Plugins let you add custom tools, slash commands, lifecycle hooks, and bundled skills without forking or modifying core Hermes code.

A plugin is typically stored under `$HERMES_HOME/plugins/` (defaults to `~/.hermes/plugins/`). When Hermes starts, it discovers and loads all eligible plugins automatically. Your tools appear alongside built-in tools immediately — the model can call them without any extra configuration.

## Plugin Directory Structure

Every plugin is a directory with at minimum a `plugin.yaml` manifest and an `__init__.py` registration module. The conventional layout for a complete plugin is:

```
${HERMES_HOME}/plugins/my-plugin/
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

provides_tools:      # list of tool names this plugin provides
  - my_tool
provides_hooks:      # list of hook names this plugin provides
  - post_tool_call

requires_env:        # optional: declare env vars required by plugin tools
  - MY_API_KEY       # plugin is disabled if this variable is not set
```

The manifest file can be named `plugin.yaml` or `plugin.yml` (both are accepted).

`requires_env` is not a hard plugin-level switch in this release. The list is preserved and forwarded into tool registration, but Hermes does not currently skip a plugin's `register()` call based on missing env variables.

### Manifest Fields Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Plugin name (defaults to directory name if omitted) |
| `version` | string | No | Version string |
| `description` | string | No | Human-readable description |
| `author` | string | No | Author name |
| `requires_env` | list | No | Environment variables associated with plugin-provided tools; used when tools are registered/filtered |
| `provides_tools` | list | No | List of tool names this plugin registers |
| `provides_hooks` | list | No | List of hook names this plugin registers |

## What Plugins Can Do

| Capability | How |
|-----------|-----|
| Add custom tools | `ctx.register_tool(name, toolset, schema, handler)` |
| Register lifecycle hooks | `ctx.register_hook("post_tool_call", callback)` |
| Ship data files | `Path(__file__).parent / "data" / "file.yaml"` |
| Bundle skills | Copy `skill.md` to `${HERMES_HOME}/skills/` at load time |
| Gate on env vars | `requires_env: [API_KEY]` in `plugin.yaml` |
| Conditional tool availability | `check_fn=lambda: _has_optional_lib()` in `register_tool` |
| Distribute via pip | `[project.entry-points."hermes_agent.plugins"]` in `pyproject.toml` |

## Plugin Discovery

Hermes discovers plugins from three sources:

| Source | Path | Use case |
|--------|------|----------|
| User plugins | `${HERMES_HOME}/plugins/` (or `~/.hermes/plugins/`) | Personal plugins |
| Project plugins | `.hermes/plugins/` (opt-in) | Project-specific plugins |
| pip entry points | `hermes_agent.plugins` entry_points group | Distributed packages |

Project plugin discovery is opt-in: only loads when `HERMES_ENABLE_PROJECT_PLUGINS=1`.

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

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | str | Yes | Tool name (must be unique across all plugins) |
| `toolset` | str | Yes | Toolset group name (shown in banner, used for toolset-level enable/disable) |
| `schema` | dict | Yes | OpenAI function-calling schema dict |
| `handler` | Callable | Yes | Python callable that executes the tool |
| `check_fn` | Callable | No | Returns bool; `False` hides tool from model (e.g., check for optional dependency) |
| `requires_env` | list | No | Environment variables required for this tool |
| `is_async` | bool | No | Set to `True` if the handler is an async function |
| `description` | str | No | Short description (separate from schema description) |
| `emoji` | str | No | Emoji shown in spinner and progress output |

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
provides_tools:
  - calculate
  - unit_convert
provides_hooks:
  - post_tool_call
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

After starting Hermes, verify plugin load state:

```
Plugins (1):
  ✓ calculator v1.0.0 (2 tools, 1 hooks)
```

The command surface is split:

- Shell workflow: `hermes plugins list` / `hermes plugins install|update|remove <name>`
- In-session workflow: `/plugins` to list loaded plugins with tool and hook counts

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
    ctx.register_hook("pre_api_request", before_api_call)
    ctx.register_hook("post_api_request", after_api_call)
    ctx.register_hook("on_session_finalize", on_session_finalize)
    ctx.register_hook("on_session_reset", on_session_reset)
```

Available hooks in plugins (`VALID_HOOKS` as of v0.8.0):

| Hook | When | Arguments |
|------|------|-----------|
| `pre_tool_call` | Before any tool runs | `tool_name`, `args`, `task_id` |
| `post_tool_call` | After any tool returns | `tool_name`, `args`, `result`, `task_id` |
| `pre_llm_call` | Once per user turn, before the API call loop | `session_id`, `user_message`, `conversation_history`, `is_first_turn`, `model`, `platform` |
| `post_llm_call` | After the turn's tool-calling loop completes | `session_id`, `user_message`, `assistant_response`, `conversation_history`, `model`, `platform` |
| `pre_api_request` | Before each individual LLM API request | `session_id`, `message_count`, `tool_count`, `model`, `platform` |
| `post_api_request` | After each individual LLM API response | `session_id`, `usage`, `model`, `platform` |
| `on_session_start` | New session created (not on continuation) | `session_id`, `model`, `platform` |
| `on_session_end` | End of every `run_conversation` call | `session_id`, `completed`, `interrupted`, `model`, `platform` |
| `on_session_finalize` | Session permanently closed (`/new`, `/reset`, or process exit) | `session_id`, `platform` |
| `on_session_reset` | New session created after a reset | `session_id`, `platform` |

The `pre_llm_call` and `post_llm_call` hooks fire once per user turn. `pre_api_request` and `post_api_request` fire on every individual API call within a turn (useful for per-request tracing). `pre_llm_call` can optionally return a dict with a `"context"` key whose string value is appended to the user message for that turn only — it is not persisted to the session DB or prompt cache. All four session/LLM hooks were activated in v0.5.0 (PR [#3542](https://github.com/NousResearch/hermes-agent/pull/3542)). The API-level hooks and session lifecycle hooks (`on_session_finalize`, `on_session_reset`) were added in v0.8.0.

## Plugin Lifecycle Hooks

Prior to v0.5.0, only `pre_tool_call` and `post_tool_call` fired. The four session-level and LLM-level hooks (`pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`) were defined but not wired into the agent loop. As of v0.5.0 (PR [#3542](https://github.com/NousResearch/hermes-agent/pull/3542)) the original six hooks are fully active. In v0.8.0 four more hooks were added: `pre_api_request`, `post_api_request`, `on_session_finalize`, and `on_session_reset`.

### Execution order within a turn

```
on_session_start            ← once, when a brand-new session is created
pre_llm_call                ← once per user turn
  [per-API-call loop]
    pre_api_request         ← before each individual LLM API request (v0.8.0)
    post_api_request        ← after each individual LLM API response (v0.8.0)
    [tool-calling]
      pre_tool_call         ← before each tool
      post_tool_call        ← after each tool
post_llm_call               ← once per user turn, after the loop completes
on_session_end              ← at the end of every run_conversation call
on_session_finalize         ← once, when session is permanently closed (v0.8.0)
on_session_reset            ← once, when a new session is created after reset (v0.8.0)
```

### Complete plugin example with all ten hooks

```python
# ~/.hermes/plugins/my-observer/__init__.py
import json
import logging
from datetime import datetime
from pathlib import Path

logger = logging.getLogger(__name__)

_LOG = Path.home() / ".hermes" / "logs" / "my-observer.jsonl"


def _log(record: dict) -> None:
    _LOG.parent.mkdir(parents=True, exist_ok=True)
    with open(_LOG, "a") as f:
        f.write(json.dumps({"ts": datetime.now().isoformat(), **record}) + "\n")


def _on_session_start(session_id, model, platform, **kwargs):
    _log({"event": "session_start", "session_id": session_id,
          "model": model, "platform": platform})


def _on_pre_llm_call(session_id, user_message, conversation_history,
                     is_first_turn, model, platform, **kwargs):
    _log({"event": "pre_llm_call", "session_id": session_id,
          "turn_msg_len": len(user_message), "history_len": len(conversation_history)})
    # Optionally inject ephemeral context into this turn's system prompt:
    # return {"context": "Today is Monday. Reminder: prefer metric units."}


def _on_post_llm_call(session_id, user_message, assistant_response,
                      conversation_history, model, platform, **kwargs):
    _log({"event": "post_llm_call", "session_id": session_id,
          "response_len": len(assistant_response or "")})


def _on_session_end(session_id, completed, interrupted, model, platform, **kwargs):
    _log({"event": "session_end", "session_id": session_id,
          "completed": completed, "interrupted": interrupted})


def _on_pre_tool_call(tool_name, args, task_id, **kwargs):
    _log({"event": "pre_tool_call", "tool": tool_name, "task_id": task_id})


def _on_post_tool_call(tool_name, args, result, task_id, **kwargs):
    _log({"event": "post_tool_call", "tool": tool_name, "task_id": task_id,
          "result_len": len(result or "")})


# v0.8.0 hooks — fire on every individual LLM API call within a turn
def _on_pre_api_request(session_id, message_count, tool_count, model, platform, **kwargs):
    _log({"event": "pre_api_request", "session_id": session_id,
          "message_count": message_count, "tool_count": tool_count})


def _on_post_api_request(session_id, usage, model, platform, **kwargs):
    _log({"event": "post_api_request", "session_id": session_id, "usage": usage})


# v0.8.0 hooks — permanent session lifecycle events
def _on_session_finalize(session_id, platform, **kwargs):
    _log({"event": "session_finalize", "session_id": session_id, "platform": platform})


def _on_session_reset(session_id, platform, **kwargs):
    _log({"event": "session_reset", "session_id": session_id, "platform": platform})


def register(ctx):
    ctx.register_hook("on_session_start", _on_session_start)
    ctx.register_hook("pre_llm_call", _on_pre_llm_call)
    ctx.register_hook("post_llm_call", _on_post_llm_call)
    ctx.register_hook("on_session_end", _on_session_end)
    ctx.register_hook("pre_tool_call", _on_pre_tool_call)
    ctx.register_hook("post_tool_call", _on_post_tool_call)
    ctx.register_hook("pre_api_request", _on_pre_api_request)
    ctx.register_hook("post_api_request", _on_post_api_request)
    ctx.register_hook("on_session_finalize", _on_session_finalize)
    ctx.register_hook("on_session_reset", _on_session_reset)
```

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

1. Built-in tools are discovered first — `model_tools._discover_tools()` imports every core tool module
2. MCP tools are discovered next — `discover_mcp_tools()` connects to configured MCP servers
3. Plugin tools are discovered last — `discover_plugins()` scans all three sources and calls `register(ctx)` for each plugin
4. `get_plugin_toolsets()` then groups plugin-registered tool names by their toolset key so they appear in `hermes tools` and standalone processes (PR [#3457](https://github.com/NousResearch/hermes-agent/pull/3457), v0.5.0)

Within plugin discovery, sources are checked in this order:

1. `~/.hermes/plugins/` (or `$HERMES_HOME/plugins/`) — subdirectories with `plugin.yaml` or `plugin.yml`
2. `.hermes/plugins/` in the current working directory — **only when** `HERMES_ENABLE_PROJECT_PLUGINS=1`
3. Installed packages — packages in the `hermes_agent.plugins` entry points group

For each plugin found, Hermes imports the module and calls `register(ctx)`. Any plugin whose `register()` raises an exception is disabled with an error logged; Hermes continues loading remaining plugins. Discovery runs only once per process (idempotent).

## Managing Plugins

Inside a CLI session:

```
/plugins              # list loaded plugins with tool and hook counts
```

No forking or source modification is required. The plugin system is designed so that all extensions are isolated — a broken plugin never crashes Hermes.

## Enabling and Disabling Plugins (v0.6.0)

Added in v0.6.0 (PR [#3747](https://github.com/NousResearch/hermes-agent/pull/3747)).

You can disable a plugin without removing it from disk. Disabled plugins are skipped entirely at startup — their `register()` function is never called and none of their tools or hooks are loaded.

```bash
hermes plugins enable <plugin-name>
hermes plugins disable <plugin-name>
hermes plugins list                   # shows enabled/disabled status for each plugin
```

Disable state is persisted in config so it survives restarts. The plugin's directory is left intact, making it easy to re-enable later. This is useful for temporarily turning off a plugin that has a conflicting tool name or a missing dependency, without losing the plugin's configuration or bundled data.

## Message Injection (v0.6.0)

Added in v0.6.0 (PR [#3778](https://github.com/NousResearch/hermes-agent/pull/3778)) by @winglian.

Plugins can inject messages into the active conversation on behalf of the user using `ctx.inject_message()`. This allows plugins to push external events — webhook payloads, timer alerts, background task completions — directly into the conversation stream without the user typing anything.

**Signature:**

```python
ctx.inject_message(content: str, role: str = "user") -> bool
```

**How it works:**

- If the agent is **idle** (waiting for user input), the message is queued as the next input and starts a new turn immediately.
- If the agent is **mid-turn** (actively running tools), the message interrupts the current operation — the same effect as the user typing a new message and pressing Enter mid-turn.
- Returns `True` if the message was queued successfully. Returns `False` if no CLI reference is available (inject_message is only available in CLI mode; it does not work in gateway mode).

Injected messages appear in session transcripts exactly as if the user typed them.

**Example — webhook receiver plugin:**

```python
import threading
from http.server import HTTPServer, BaseHTTPRequestHandler

def register(ctx):
    def _start_server():
        class Handler(BaseHTTPRequestHandler):
            def do_POST(self):
                length = int(self.headers.get("Content-Length", 0))
                body = self.rfile.read(length).decode()
                ctx.inject_message(f"Webhook received: {body}")
                self.send_response(200)
                self.end_headers()
        HTTPServer(("", 9876), Handler).serve_forever()

    t = threading.Thread(target=_start_server, daemon=True)
    t.start()
```

**Use cases:** timer alerts, background job completions, webhook receivers, messaging bridge adapters, IoT sensor events.

## CLI Subcommand Registration (v0.8.0)

Plugins can register their own `hermes <command>` subcommands without modifying `main.py`. Use `ctx.register_cli_command()` inside `register()`:

```python
def _setup_my_subparser(subparser):
    subparser.add_argument("action", choices=["status", "sync"])
    subparser.set_defaults(func=_handle_my_command)

def _handle_my_command(args):
    if args.action == "status":
        print("My plugin status: OK")

def register(ctx):
    ctx.register_cli_command(
        name="my-plugin",
        help="Manage my-plugin",
        setup_fn=_setup_my_subparser,
        handler_fn=_handle_my_command,  # optional
    )
```

After the plugin loads, `hermes my-plugin status` works as a first-class CLI subcommand. The `setup_fn` receives an argparse subparser and can add nested sub-subparsers, arguments, and defaults. The optional `handler_fn` parameter sets `set_defaults(func=handler_fn)` on the subparser automatically, so you can omit the explicit `set_defaults` call inside `setup_fn` when a single top-level handler covers all actions.

## Prompting for Required Env Vars During Install (v0.8.0)

When `requires_env` entries in `plugin.yaml` include a description, Hermes prompts the user for any missing values during `hermes plugins install`:

```yaml
name: my-plugin
requires_env:
  - MY_API_KEY
  - name: MY_WEBHOOK_URL
    description: "Webhook endpoint for delivery"
    required: true
```

Simple string entries are checked for presence. Dict entries (rich format) support `description`, `url`, and `secret` metadata. All variables are prompted regardless of format; empty input skips gracefully with a reminder to set the value in `~/.hermes/.env` later.

## Memory Provider Plugins (Released in v0.7.0)

Released `v0.7.0` turns external memory backends into a first-class plugin surface. Providers implement the `MemoryProvider` interface and register through the same plugin loading path as other Hermes extensions.

Released provider plugins include:

- `honcho`
- `openviking`
- `mem0`
- `hindsight`
- `holographic`
- `retaindb`
- `byterover`

The built-in file-backed memory (`MEMORY.md` / `USER.md`) remains active. Hermes can then layer one external provider alongside it for cross-session retrieval and persistence. Configure the active provider with `hermes memory setup` or by setting `memory.provider` in `~/.hermes/config.yaml`.

See [developer-guide/memory-provider-plugin.md](developer-guide/memory-provider-plugin.md) for the implementation contract and [memory.md](memory.md) for operator-facing setup details.
