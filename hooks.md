# Event Hooks

The hooks system lets you run custom code at key points in the agent lifecycle. Hooks fire automatically during gateway operation without blocking the main agent pipeline. A broken hook is logged and skipped — it never crashes the agent.

Hooks exist in two separate forms in Hermes:

1. **Gateway hooks** — standalone directories under `~/.hermes/hooks/`, each with a `HOOK.yaml` and `handler.py`. These fire during gateway operation (Telegram, Discord, Slack, WhatsApp).
2. **Plugin hooks** — registered via `ctx.register_hook()` inside a plugin's `register()` function. These fire during any Hermes session.

This document covers both.

## Gateway Hooks

Gateway hooks only fire when the gateway is running. The CLI does not currently load gateway hooks.

### Directory Structure

```text
~/.hermes/hooks/
└── my-hook/
    ├── HOOK.yaml      # declares which events to listen for
    └── handler.py     # Python handler function
```

### HOOK.yaml

```yaml
name: my-hook
description: Log all agent activity to a file
events:
  - agent:start
  - agent:end
  - agent:step
```

The `events` list determines which events trigger your handler. You can subscribe to any combination, including wildcards like `command:*`.

### handler.py

The handler function must be named `handle`. It receives `event_type` (string) and `context` (dict).

```python
import json
from datetime import datetime
from pathlib import Path

LOG_FILE = Path.home() / ".hermes" / "hooks" / "my-hook" / "activity.log"

async def handle(event_type: str, context: dict):
    """Called for each subscribed event. Must be named 'handle'."""
    entry = {
        "timestamp": datetime.now().isoformat(),
        "event": event_type,
        **context,
    }
    with open(LOG_FILE, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

**Handler rules:**
- Must be named `handle`
- Receives `event_type` (string) and `context` (dict)
- Can be `async def` or regular `def` — both work
- Errors are caught and logged, never crashing the agent

## Gateway Hook Lifecycle

On gateway startup, `HookRegistry.discover_and_load()` scans `~/.hermes/hooks/`. Each subdirectory containing `HOOK.yaml` and `handler.py` is loaded dynamically. Handlers are registered for their declared events. At each lifecycle point, `hooks.emit()` fires all matching handlers.

## Available Gateway Events

| Event | When it fires | Context keys |
|-------|---------------|--------------|
| `gateway:startup` | Gateway process starts | `platforms` (list of active platform names) |
| `session:start` | New messaging session created | `platform`, `user_id`, `session_id`, `session_key` |
| `session:end` | Session ends (user ran `/new` or `/reset`) | `platform`, `user_id`, `session_key` |
| `session:reset` | Session reset completed (new session entry created) | `platform`, `user_id`, `session_key` |
| `agent:start` | Agent begins processing a message | `platform`, `user_id`, `session_id`, `message` |
| `agent:step` | Each iteration of the tool-calling loop | `platform`, `user_id`, `session_id`, `iteration`, `tool_names` |
| `agent:end` | Agent finishes processing | `platform`, `user_id`, `session_id`, `message`, `response` |
| `command:*` | Any slash command executed | `platform`, `user_id`, `command`, `args` |

### Wildcard Matching

Handlers registered for `command:*` fire for any `command:` event (`command:model`, `command:reset`, etc.). Use a single subscription to monitor all slash commands.

## Gateway Hook Examples

### Telegram Alert on Long Tasks

Send a Telegram message when the agent takes more than 10 steps:

```yaml
# ~/.hermes/hooks/long-task-alert/HOOK.yaml
name: long-task-alert
description: Alert when agent is taking many steps
events:
  - agent:step
```

```python
# ~/.hermes/hooks/long-task-alert/handler.py
import os
import httpx

THRESHOLD = 10
BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
CHAT_ID = os.getenv("TELEGRAM_HOME_CHANNEL")

async def handle(event_type: str, context: dict):
    iteration = context.get("iteration", 0)
    if iteration == THRESHOLD and BOT_TOKEN and CHAT_ID:
        tools = ", ".join(context.get("tool_names", []))
        text = f"Agent has been running for {iteration} steps. Last tools: {tools}"
        async with httpx.AsyncClient() as client:
            await client.post(
                f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
                json={"chat_id": CHAT_ID, "text": text},
            )
```

### Command Usage Logger

Track which slash commands are used:

```yaml
# ~/.hermes/hooks/command-logger/HOOK.yaml
name: command-logger
description: Log slash command usage
events:
  - command:*
```

```python
# ~/.hermes/hooks/command-logger/handler.py
import json
from datetime import datetime
from pathlib import Path

LOG = Path.home() / ".hermes" / "logs" / "command_usage.jsonl"

def handle(event_type: str, context: dict):
    LOG.parent.mkdir(parents=True, exist_ok=True)
    entry = {
        "ts": datetime.now().isoformat(),
        "command": context.get("command"),
        "args": context.get("args"),
        "platform": context.get("platform"),
        "user": context.get("user_id"),
    }
    with open(LOG, "a") as f:
        f.write(json.dumps(entry) + "\n")
```

### Session Start Webhook

POST to an external service on new sessions:

```yaml
# ~/.hermes/hooks/session-webhook/HOOK.yaml
name: session-webhook
description: Notify external service on new sessions
events:
  - session:start
  - session:reset
```

```python
# ~/.hermes/hooks/session-webhook/handler.py
import httpx

WEBHOOK_URL = "https://your-service.example.com/hermes-events"

async def handle(event_type: str, context: dict):
    async with httpx.AsyncClient() as client:
        await client.post(WEBHOOK_URL, json={
            "event": event_type,
            **context,
        }, timeout=5)
```

## Plugin Hooks

Plugin hooks are registered inside a plugin's `register(ctx)` function. They fire during any Hermes session (CLI, gateway, or cron), not just the gateway.

```python
def register(ctx):
    ctx.register_hook("pre_tool_call", before_any_tool)
    ctx.register_hook("post_tool_call", after_any_tool)
    ctx.register_hook("pre_llm_call", before_llm_call)
    ctx.register_hook("post_llm_call", after_llm_call)
    ctx.register_hook("on_session_start", on_new_session)
    ctx.register_hook("on_session_end", on_session_end)
```

Available plugin hooks:

| Hook | When | Arguments |
|------|------|-----------|
| `pre_tool_call` | Before any tool runs | `tool_name`, `args`, `task_id` |
| `post_tool_call` | After any tool returns | `tool_name`, `args`, `result`, `task_id` |
| `pre_llm_call` | Before LLM API call | `messages`, `model` |
| `post_llm_call` | After LLM response | `messages`, `response`, `model` |
| `on_session_start` | Session begins | `session_id`, `platform` |
| `on_session_end` | Session ends | `session_id`, `platform` |

Plugin hook handlers receive keyword arguments. Always use `**kwargs` to stay forward-compatible:

```python
def _on_post_tool_call(tool_name, args, result, task_id, **kwargs):
    print(f"Tool {tool_name} returned {len(result)} bytes")
```

Plugin hooks are observers only. They cannot modify tool arguments or return values.

## Hook Lifecycle Summary

For gateway hooks:

1. `gateway:startup` — fires once when the gateway process starts
2. `session:start` — fires when a user's first message creates a session
3. `agent:start` — fires when the agent begins processing a message
4. `agent:step` — fires on each tool-calling iteration (includes `iteration` count and `tool_names`)
5. `agent:end` — fires when the agent produces its final response
6. `session:reset` — fires when the user runs `/new` or `/reset`
7. `command:*` — fires for every slash command with command name and args

For plugin hooks:

1. `on_session_start` — session created
2. `pre_llm_call` — just before each LLM API request
3. `pre_tool_call` — just before each tool execution
4. `post_tool_call` — just after each tool returns
5. `post_llm_call` — just after each LLM response
6. `on_session_end` — session ends

## Error Handling

Errors in any hook handler are caught and logged at warning level. They never propagate to the agent pipeline. A hook that raises an exception on every call will log warnings continuously but will not break Hermes.
