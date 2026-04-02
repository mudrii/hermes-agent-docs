# Tools Runtime

Hermes tools are self-registering functions grouped into toolsets and executed through a central registry/dispatch system.

Primary files: `tools/registry.py`, `model_tools.py`, `toolsets.py`, `tools/terminal_tool.py`, `tools/environments/*`

## Tool Registration Model

Each tool module calls `registry.register()` at import time. `model_tools.py` is responsible for importing/discovering tool modules and building the schema list used by the model.

### How registry.register() Works

Every tool file in `tools/` calls `registry.register()` at module level to declare itself:

```python
registry.register(
    name="terminal",               # Unique tool name (used in API schemas)
    toolset="terminal",            # Toolset this tool belongs to
    schema={...},                  # OpenAI function-calling schema
    handler=handle_terminal,       # The function that executes when called
    check_fn=check_terminal,       # Optional: returns True/False for availability
    requires_env=["SOME_VAR"],     # Optional: env vars needed (for UI display)
    is_async=False,                # Whether the handler is an async coroutine
    description="Run commands",    # Human-readable description
    emoji="...",                   # Emoji for spinner/progress display
)
```

Each call creates a `ToolEntry` stored in the singleton `ToolRegistry._tools` dict keyed by tool name. If a name collision occurs across toolsets, a warning is logged and the later registration wins.

### Discovery: _discover_tools()

When `model_tools.py` is imported, it calls `_discover_tools()` which imports every tool module in order. Each import triggers the module's `registry.register()` calls. Errors in optional tools (e.g. missing `fal_client` for image generation) are caught and logged -- they don't prevent other tools from loading.

After core tool discovery, MCP tools and plugin tools are also discovered:

1. **MCP tools** -- `tools.mcp_tool.discover_mcp_tools()` reads MCP server config and registers tools from external servers
2. **Plugin tools** -- `hermes_cli.plugins.discover_plugins()` loads user/project/pip plugins that may register additional tools

## Tool Availability Checking (check_fn)

Each tool can optionally provide a `check_fn` -- a callable that returns `True` when the tool is available and `False` otherwise. Typical checks include:

- **API key present** -- e.g. `lambda: bool(os.environ.get("SERP_API_KEY"))` for web search
- **Service running** -- e.g. checking if the Honcho server is configured
- **Binary installed** -- e.g. verifying `playwright` is available for browser tools

Key behaviors:
- Check results are cached per-call -- if multiple tools share the same `check_fn`, it only runs once
- Exceptions in `check_fn()` are treated as "unavailable" (fail-safe)
- `is_toolset_available()` checks whether a toolset's `check_fn` passes, used for UI display and toolset resolution

## Toolset Resolution

Toolsets are named bundles of tools. Hermes resolves them through explicit enabled/disabled toolset lists, platform presets, dynamic MCP toolsets, and curated special-purpose sets.

### How get_tool_definitions() Filters Tools

The main entry point is `model_tools.get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode)`:

1. If `enabled_toolsets` is provided -- only tools from those toolsets are included
2. If `disabled_toolsets` is provided -- start with ALL toolsets, then subtract the disabled ones
3. If neither -- include all known toolsets
4. Registry filtering -- the resolved tool name set is passed to `registry.get_definitions()`, which applies `check_fn` filtering and returns OpenAI-format schemas
5. Dynamic schema patching -- `execute_code` and `browser_navigate` schemas are adjusted to only reference tools that actually passed filtering

## Dispatch

### Dispatch Flow: model tool_call -> handler execution

```
Model response with tool_call
    |
run_agent.py agent loop
    |
model_tools.handle_function_call(name, args, task_id, user_task)
    |
[Agent-loop tools?] -> handled directly by agent loop
    |                   (todo, memory, session_search, delegate_task)
[Plugin pre-hook] -> invoke_hook("pre_tool_call", ...)
    |
registry.dispatch(name, args, **kwargs)
    |
Look up ToolEntry by name
    |
[Async handler?] -> bridge via _run_async()
[Sync handler?]  -> call directly
    |
Return result string (or JSON error)
    |
[Plugin post-hook] -> invoke_hook("post_tool_call", ...)
```

### Error Wrapping

All tool execution is wrapped in error handling at two levels:

1. `registry.dispatch()` catches any exception from the handler and returns `{"error": "Tool execution failed: ExceptionType: message"}` as JSON
2. `handle_function_call()` wraps the entire dispatch in a secondary try/except that returns `{"error": "Error executing tool_name: message"}`

This ensures the model always receives a well-formed JSON string.

### Agent-Loop Tools

Four tools are intercepted before registry dispatch because they need agent-level state (TodoStore, MemoryStore, etc.):

- `todo` -- planning/task tracking
- `memory` -- persistent memory writes
- `session_search` -- cross-session recall
- `delegate_task` -- spawns subagent sessions

### Async Bridging

When a tool handler is async, `_run_async()` bridges it to the sync dispatch path:

- **CLI path (no running loop)** -- uses a persistent event loop to keep cached async clients alive
- **Gateway path (running loop)** -- spins up a disposable thread with `asyncio.run()`
- **Worker threads (parallel tools)** -- uses per-thread persistent loops stored in thread-local storage

## Dangerous Command Approval

The terminal tool integrates a dangerous-command approval system defined in `tools/approval.py`:

1. **Pattern detection** -- `DANGEROUS_PATTERNS` is a list of `(regex, description)` tuples covering destructive operations (recursive deletes, filesystem formatting, SQL destructive operations, remote code execution, etc.)
2. **Detection** -- before executing any terminal command, `detect_dangerous_command(command)` checks against all patterns
3. **Approval prompt** -- CLI mode uses an interactive prompt; gateway mode uses an async approval callback; smart approval can optionally auto-approve low-risk commands that match patterns
4. **Session state** -- approvals are tracked per-session; once approved, subsequent matching commands don't re-prompt
5. **Permanent allowlist** -- the "allow permanently" option writes the pattern to `config.yaml`'s `command_allowlist`

## Concurrency

Tool calls may execute sequentially or concurrently:

- **Sequential** -- for single tool calls or interactive tools (e.g. `clarify`)
- **Concurrent** -- for multiple non-interactive tools via `ThreadPoolExecutor` (max 8 workers)

The parallelization decision (`_should_parallelize_tool_batch()`) checks that the batch has more than 1 call, no tool is in `_NEVER_PARALLEL_TOOLS`, every tool is either read-only safe or path-scoped targeting non-overlapping paths, and all arguments parse as valid JSON. Message ordering is preserved when reinserting tool responses.
