# Memory Provider Plugins

Memory provider plugins are a released v0.7.0 feature. Hermes now exposes an abstract `MemoryProvider` interface plus a `MemoryManager` that always keeps the built-in provider and can attach at most one external provider plugin at a time.

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

The v0.7.0 source tree ships the following released providers under `plugins/memory/`:

- `honcho`
- `byterover`
- `hindsight`
- `holographic`
- `mem0`
- `openviking`
- `retaindb`

Honcho is the reference plugin and restores full parity with the old dedicated Honcho integration while conforming to the new provider interface.

## Configuration

Select a provider in `~/.hermes/config.yaml`:

```yaml
memory:
  provider: honcho
```

Provider-specific setup is handled by each plugin. Several plugins also write profile-scoped config files under `$HERMES_HOME`.

## Operational Model

Released v0.7.0 keeps the built-in memory system authoritative for local notes while allowing one provider plugin to extend or replace parts of the recall/write path. In practice:

- the built-in memory provider stays present
- one external provider may be registered
- provider-specific tools are exposed only when that provider is active
- prompt injection and persistence boundaries remain enforced by the main Hermes runtime

See [memory.md](../memory.md) for the built-in memory model, [honcho.md](../honcho.md) for the reference provider, and [plugins.md](../plugins.md) for the general plugin architecture.
