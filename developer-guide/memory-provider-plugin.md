# Memory Provider Plugins

**Status: Planned for a future release.** Memory provider plugins are not part of v0.6.0. The plugin implementations (byterover, holographic, honcho-plugin, mem0, openviking, retaindb) exist on the development branch but have not been released.

## Current State (v0.6.0)

Hermes Agent v0.6.0 uses a dual-layer memory architecture:

- **Built-in memory** -- local files (`MEMORY.md` and `USER.md`) managed by the `memory` tool, injected into the system prompt as a frozen snapshot at session start
- **Honcho integration** -- optional cloud-backed AI-native user modeling via the Honcho API, providing cross-session dialectic reasoning and dual-peer representations

Both layers can run independently or together in `hybrid` mode (the default when Honcho is enabled).

## Planned Plugin Interface

A memory provider plugin framework will allow third-party memory backends to be registered alongside or in place of the built-in memory system. The plugin interface will:

- Allow plugins to register custom memory storage and retrieval backends
- Support the same lifecycle hooks as the existing plugin system (`on_session_start`, `on_session_end`, etc.)
- Integrate with the existing `memory` tool surface so the agent's memory operations route transparently to the configured backend

See [plugins.md](../plugins.md) for the current plugin architecture and [memory.md](../memory.md) for the built-in memory system.
