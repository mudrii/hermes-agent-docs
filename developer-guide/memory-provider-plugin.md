# Memory Provider Plugins

Memory provider plugins were introduced in v0.7.0. Hermes exposes an abstract `MemoryProvider` interface plus a `MemoryManager` that always keeps the built-in provider and can attach at most one external provider plugin at a time. v0.8.0 added new session lifecycle hooks, per-user memory scoping, and the Supermemory provider.

## Released Architecture

Hermes memory is now split into two layers:

- **Built-in provider** -- local `MEMORY.md` / `USER.md` storage and the existing `memory` tool
- **External provider plugin** -- an optional plugin selected by `memory.provider` and loaded from `plugins/memory/<name>/`

The built-in provider is always registered first and cannot be removed. The external provider can add context injection, sync behavior, and provider-specific tools.

## Provider Interface

Released providers implement the `MemoryProvider` ABC in `agent/memory_provider.py`. The manager lifecycle is wired in `run_agent.py` and `agent/memory_manager.py`.

At a high level, a provider can:

- return its provider name
- expose provider-specific tool schemas
- initialize per-session state
- contribute memory/context text to the system prompt
- handle provider-specific tool invocations
- flush or sync state on session boundaries

The manager also maintains tool-to-provider routing so provider-specific tools resolve back to the correct memory backend.

## Released Providers

The v0.8.0 source tree ships the following released providers under `plugins/memory/`:

- `honcho`
- `byterover`
- `hindsight`
- `holographic`
- `mem0`
- `openviking`
- `retaindb`
- `supermemory` *(added in v0.8.0)*

Honcho is the reference plugin and restores full parity with the old dedicated Honcho integration while conforming to the new provider interface.

## Configuration

Select a provider in `~/.hermes/config.yaml`:

```yaml
memory:
  provider: honcho
```

Provider-specific setup is handled by each plugin. Several plugins also write profile-scoped config files under `$HERMES_HOME`.

## Operational Model

The built-in memory system stays authoritative for local notes while allowing one provider plugin to extend or replace parts of the recall/write path. In practice:

- the built-in memory provider stays present
- one external provider may be registered
- provider-specific tools are exposed only when that provider is active
- prompt injection and persistence boundaries remain enforced by the main Hermes runtime

## Session Lifecycle Hooks (v0.8.0)

Two new session lifecycle hooks were added in v0.8.0 alongside the existing `on_session_start` and `on_session_end`:

| Hook | When it fires |
|------|---------------|
| `on_session_start` | Agent starts a new session |
| `on_session_end` | Session ends normally |
| `on_session_finalize` | Session is fully committed and flushed (after `/new`, `/reset`, or session expiry) |
| `on_session_reset` | User or gateway explicitly resets the session |

Plugins register hooks via `ctx.register_hook(hook_name, callback)`. The complete set of valid hook names is defined in `VALID_HOOKS` in `hermes_cli/plugins.py`. Registering an unknown hook name produces a warning but is not fatal, so forward-compatible plugins do not break.

The `on_session_finalize` and `on_session_reset` hooks enable memory providers to perform cleanup or state transitions at predictable session boundaries. Gateway coverage for both hooks was added in v0.8.0.

## Per-User Memory Scoping (v0.8.0)

`initialize()` now receives a `user_id` keyword argument when called from a gateway session. This allows memory providers to scope storage and retrieval to the specific user who sent the message, enabling correct isolation in multi-user thread conversations.

```python
def initialize(self, session_id: str, **kwargs) -> None:
    user_id = kwargs.get("user_id")  # set for gateway sessions; None for CLI
```

The `thread_gateway_user_id` is threaded through the memory manager initialization path so all providers receive the correct user identity even in shared-thread sessions (see [memory.md](../memory.md) for shared thread session details).

## CLI Plugin Registration (v0.8.0)

Plugins can now register CLI subcommands via `ctx.register_cli_command()`. This is how the Honcho plugin exposes `hermes honcho ...` — the CLI no longer relies on static command registration hardcoded to the dedicated Honcho integration.

```python
def register(ctx) -> None:
    ctx.register_memory_provider(MyMemoryProvider())
    ctx.register_cli_command(
        name="myplugin",
        help="Manage MyPlugin settings",
        setup_fn=my_setup_fn,
        handler_fn=my_handler_fn,
    )
```

See [memory.md](../memory.md) for the built-in memory model, [honcho.md](../honcho.md) for the reference provider, and [plugins.md](../plugins.md) for the general plugin architecture.
