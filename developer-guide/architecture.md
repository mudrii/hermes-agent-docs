# Architecture

This page is the top-level map of Hermes Agent internals, reviewed against the released v0.7.0 surface (`v2026.4.3`). The project is organized around a shared agent core that serves multiple platform frontends, a centralized provider router, a SQLite session store, and a set of loosely-coupled optional subsystems.

## High-Level Structure

```
hermes-agent/
+-- run_agent.py              AIAgent core loop (entry point for all platforms)
+-- cli.py                    Interactive terminal UI (prompt_toolkit TUI)
+-- model_tools.py            Tool discovery, schema building, and dispatch
+-- toolsets.py               Tool groupings and platform presets
+-- hermes_state.py           SQLite session/state database
+-- hermes_constants.py       Shared API endpoint constants
+-- hermes_time.py            Timezone-aware clock
+-- batch_runner.py           Batch trajectory generation for RL/SFT
+-- trajectory_compressor.py  Offline training data compression
|
+-- agent/                    Prompt building, compression, caching, metadata,
|                             trajectories, auxiliary client, adapters
+-- hermes_cli/               Command entrypoints, auth, setup, models, config
+-- tools/                    Tool implementations and terminal environments
+-- gateway/                  Messaging gateway, session routing, delivery,
|                             pairing, hooks
+-- cron/                     Scheduled job storage and scheduler
+-- plugins/memory/           External memory-provider plugins
+-- acp_adapter/              ACP editor integration server (VS Code, Zed, JetBrains)
+-- acp_registry/             ACP registry manifest + icon
+-- environments/             RL/benchmark environment framework
+-- skills/                   Bundled skill documents
+-- optional-skills/          Official optional skills
+-- tests/                    Test suite
```

## Component Diagram

```
+--- Entry Points ------------------------------------------------------+
|  hermes          Interactive terminal CLI (cli.py)                    |
|  hermes-agent    Programmatic runner (run_agent.py main())            |
|  hermes-acp      ACP editor integration server (acp_adapter/entry.py)|
|  API server      OpenAI-compatible /v1/chat/completions endpoint      |
|                  (gateway platform, activated via hermes gateway)     |
+------------------------------+----------------------------------------+
                               |
                               v
+--- Core Agent Loop ---------------------------------------------------+
|  AIAgent  (run_agent.py)                                              |
|                                                                       |
|  +-- Prompt Assembly -----------------------------------------------+ |
|  |  agent/prompt_builder.py                                         | |
|  |  SOUL.md identity, skills index, context files, platform hints   | |
|  +------------------------------------------------------------------+ |
|                                                                       |
|  +-- LLM Inference -------------------------------------------------+ |
|  |  Three API modes: chat_completions / codex_responses /           | |
|  |                   anthropic_messages                             | |
|  |  agent/auxiliary_client.py -> call_llm() / async_call_llm()     | |
|  |  agent/anthropic_adapter.py -> native Anthropic support          | |
|  |  agent/smart_model_routing.py -> cheap vs strong routing         | |
|  +------------------------------------------------------------------+ |
|                                                                       |
|  +-- Tool Execution ------------------------------------------------+ |
|  |  model_tools.py -> get_tool_definitions(), handle_function_call()| |
|  |  Sequential (single or interactive tools)                        | |
|  |  Concurrent via ThreadPoolExecutor (max 8 workers)               | |
|  +------------------------------------------------------------------+ |
|                                                                       |
|  +-- Context Management --------------------------------------------+ |
|  |  agent/context_compressor.py -> ContextCompressor                | |
|  |  agent/prompt_caching.py -> Anthropic cache_control markers      | |
|  |  agent/model_metadata.py -> context length resolution            | |
|  +------------------------------------------------------------------+ |
+------------------------------+----------------------------------------+
                               |
                               v
+--- Session Storage ---------------------------------------------------+
|  hermes_state.py -> SessionDB                                         |
|  SQLite WAL mode, schema v6, FTS5 full-text search                   |
|  Default: ~/.hermes/state.db (override via HERMES_HOME)              |
+------------------------------+----------------------------------------+
                               |
          +--------------------+------------------------+
          |                    |                        |
          v                    v                        v
+--- Messaging --------+ +--- ACP ----------------+ +--- Cron -----------+
|  gateway/            | |  acp_adapter/           | |  cron/             |
|  multi-platform      | |  stdio/JSON-RPC         | |  scheduler + jobs  |
|  gateway + API       | |  VS Code, Zed,          | |  SQLite-backed     |
|  Session routing     | |  JetBrains              | |  60s tick interval |
|  Delivery            | +------------------------+ +--------------------+
+----------------------+

+--- Optional Subsystems -----------------------------------------------+
|  plugins/memory/     -> external memory providers (Honcho, Mem0, etc.)|
|  environments/       -> RL benchmark framework                        |
|  tools/              -> built-in tool implementations and runtimes    |
|  skills/             -> bundled skills plus optional skill catalog    |
+----------------------------------------------------------------------+
```

## Major Subsystems

### Agent Loop

The core synchronous orchestration engine is `AIAgent` in `run_agent.py`. It is responsible for provider/API-mode selection, prompt construction, tool execution, retries and fallback, callbacks, compression, and persistence. See [Agent Loop Internals](./agent-loop.md).

### Prompt System

Prompt-building logic is split between `run_agent.py`, `agent/prompt_builder.py`, `agent/prompt_caching.py`, and `agent/context_compressor.py`. The system prompt is assembled once per session and kept stable for prompt caching. See [Prompt Assembly](./prompt-assembly.md) and [Context Compression](./context-compression.md).

### Provider/Runtime Resolution

Hermes has a shared runtime provider resolver used by CLI, gateway, cron, ACP, and auxiliary calls. It handles credential lookup, API mode detection, and base URL scoping. See [Provider Runtime](./provider-runtime.md).

### Tool Runtime

The tool registry, toolsets, terminal backends, process manager, and dispatch rules form a subsystem of their own. Tools self-register at import time and are dispatched through a central registry. See [Tools Runtime](./tools-runtime.md).

### Session Persistence

Historical session state is stored in SQLite with WAL mode, FTS5 full-text search, and lineage tracking across compression splits. See [Session Storage](./session-storage.md).

### Messaging Gateway

The gateway is a long-running orchestration layer for platform adapters, session routing, pairing, delivery, and cron ticking. See [Gateway Internals](./gateway-internals.md).

### Cron

Cron jobs are implemented as first-class agent tasks stored in `~/.hermes/cron/jobs.json` with support for one-shot delays, intervals, cron expressions, and skill-backed jobs. See [Cron Internals](./cron-internals.md).

### RL / Environments / Trajectories

Hermes ships a full environment framework for evaluation, RL integration, and SFT data generation. Trajectories are saved as JSONL in ShareGPT-compatible format. See [Trajectory Format](./trajectory-format.md).

## Design Themes

Several cross-cutting design themes appear throughout the codebase:

- **Prompt stability matters** -- the system prompt is assembled once per session and the stable prefix must remain stable for provider-side prompt caching to be effective
- **Tool execution must be observable and interruptible** -- every tool call fires callbacks; long LLM calls can be cancelled
- **Session persistence must survive long-running use** -- every message and token count is written to SQLite immediately; WAL mode allows concurrent readers
- **Platform frontends should share one agent core** -- `AIAgent` is identical whether called from CLI, gateway, ACP, or cron
- **Optional subsystems should remain loosely coupled** -- Honcho, MCP, and ACP are injected via callbacks and configuration, not hardwired into the agent loop
- **Graceful degradation everywhere** -- `_SafeWriter` wraps stdout/stderr to prevent crashes from broken pipes; streaming falls back to non-streaming on error; auxiliary models fall back through the provider chain

## Recommended Reading Order

1. This page
2. [Agent Loop Internals](./agent-loop.md)
3. [Prompt Assembly](./prompt-assembly.md)
4. [Provider Runtime](./provider-runtime.md)
5. [Tools Runtime](./tools-runtime.md)
6. [Session Storage](./session-storage.md)
7. [Gateway Internals](./gateway-internals.md)
8. [Context Compression](./context-compression.md)
9. [Trajectory Format](./trajectory-format.md)

## Source Files Reference

| File | Description |
|------|-------------|
| `run_agent.py` | `AIAgent` -- core orchestration loop (entry point for all platforms) |
| `cli.py` | Interactive terminal UI (prompt_toolkit TUI) |
| `model_tools.py` | Tool discovery, schema building, and dispatch |
| `toolsets.py` | Tool groupings and platform presets |
| `hermes_state.py` | `SessionDB` -- SQLite session and message store |
| `hermes_constants.py` | Shared API endpoint constants |
| `hermes_time.py` | Timezone-aware clock |
| `batch_runner.py` | Batch trajectory generation for RL/SFT |
| `trajectory_compressor.py` | `TrajectoryCompressor` -- offline training data compression |
| `agent/context_compressor.py` | `ContextCompressor` -- runtime context management |
| `agent/prompt_builder.py` | System prompt assembly functions |
| `agent/prompt_caching.py` | Anthropic cache_control markers |
| `agent/auxiliary_client.py` | Centralized LLM router for side tasks |
| `agent/model_metadata.py` | Context length resolution, token estimation |
| `agent/smart_model_routing.py` | Optional cheap model routing |
| `agent/anthropic_adapter.py` | Native Anthropic API translation |
| `agent/trajectory.py` | Trajectory save utilities |
| `gateway/` | Platform adapters and session routing |
| `cron/` | Scheduled job storage and scheduler |
