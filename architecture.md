# Architecture

Hermes Agent is a Python-based AI agent platform organized around a persistent agent loop, a centralized tool registry, a multi-platform messaging gateway, and a three-tier memory system.

---

## High-Level System Design

```
┌─ Entry Points ──────────────────────────────────────────┐
│  hermes (interactive CLI)                                │
│  hermes-agent (programmatic runner)                     │
│  hermes-acp (editor integration via ACP protocol)       │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─ Core Agent Loop ─────────────────────────────────────  ┐
│  AIAgent (run_agent.py — 6,211 lines)                   │
│  ├── Prompt assembly (agent/prompt_builder.py)          │
│  ├── LLM inference (provider router)                    │
│  ├── Tool execution (model_tools.py + tools/registry.py)│
│  ├── Context compression (agent/context_compressor.py)  │
│  ├── Prompt caching (agent/prompt_caching.py)           │
│  └── Anthropic adapter (agent/anthropic_adapter.py)     │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─ Session Storage ─────────────────────────────────────  ┐
│  hermes_state.py — SQLite (WAL mode, FTS5 search)       │
└──────────────────────────────────────────────────────────┘

┌─ Messaging Gateway ────────────────────────────────────  ┐
│  gateway/ — 8 platform adapters                         │
│  ├── Telegram, Discord, Slack, WhatsApp                 │
│  ├── Signal, Email (IMAP/SMTP), Home Assistant          │
│  └── Session routing, delivery, cron, STT               │
└──────────────────────────────────────────────────────────┘

┌─ Specialized Subsystems ──────────────────────────────   ┐
│  cron/          — Scheduled task runner                  │
│  honcho_integration/ — Cross-session user memory        │
│  acp_adapter/   — Editor protocol (VS Code, Zed, etc.)  │
│  environments/  — RL/benchmark evaluation               │
└──────────────────────────────────────────────────────────┘
```

---

## Agent Loop (ReAct Pattern)

The core agent implements a **Reason → Act → Observe** loop:

```
User Input
    ↓
Build System Prompt
  ├── Identity (SOUL.md or default)
  ├── Context files (AGENTS.md, .cursorrules)
  ├── Skill definitions (dynamically loaded)
  ├── Honcho user context (if enabled)
  └── Platform hints (CLI vs Telegram vs Discord)
    ↓
LLM Call (via Provider Router)
  ├── Anthropic (with prompt caching + extended thinking)
  ├── OpenAI-compatible (OpenRouter, Nous Portal, etc.)
  └── Streaming SSE
    ↓
Parse Response
  ├── Text → stream to output
  └── Tool calls → dispatch to registry
    ↓
Execute Tools (parallel, up to 8 workers)
  ├── Each tool: check_fn → handler → JSON result
  ├── Sequential fallback for clarify tool
  └── Iteration budget shared with subagents
    ↓
Append tool results to messages
    ↓
Loop (until no tool calls or max_turns reached)
    ↓
Return final response
```

---

## Key Design Principles

From the official architecture documentation:

1. **Prompt stability** — System prompt assembled once per session start; skills and context loaded at initialization
2. **Observable/interruptible tool execution** — All tool calls fire progress callbacks; can be cancelled mid-turn
3. **Session persistence** — Every message, tool call, and token count persisted to SQLite immediately
4. **Shared agent core** — Identical AIAgent class serves CLI, messaging gateway, ACP, and cron jobs
5. **Loose coupling** — Optional subsystems (Honcho, MCP, ACP) injected via callbacks, not hardwired

---

## AIAgent Class

**File:** `run_agent.py` (6,211 lines)

### Initialization Parameters

```python
AIAgent(
    base_url: str,                 # LLM endpoint URL
    api_key: str,                  # Provider API key
    provider: str,                 # Provider identifier
    model: str = "anthropic/claude-opus-4.6",
    max_iterations: int = 90,      # Shared with subagents
    enabled_toolsets: list = [],
    disabled_toolsets: list = [],
    session_db: HermesState = None,
    session_id: str = None,
    honcho_session_key: str = None,
    honcho_config: dict = None,
    platform: str = "cli",         # cli | telegram | discord | ...
    checkpoints_enabled: bool = False,
)
```

### Callbacks

| Callback | Fires When |
|----------|-----------|
| `tool_progress_callback` | Tool starts or completes |
| `thinking_callback` | Extended thinking block received |
| `reasoning_callback` | Reasoning step completed |
| `step_callback` | Iteration milestone |
| `clarify_callback` | User clarification requested |

### Context Management

- **Compression trigger:** 50% of token budget (configurable)
- **Compression model:** Configured auxiliary model (default: Gemini 3 Flash)
- **Prompt caching:** Applied for Anthropic models (cache control headers)
- **Token estimation:** Rough approximation without full tokenization (performance)

---

## Tool System Architecture

### Registry Pattern

All tools self-register at import time via a central registry:

```python
# tools/registry.py
class ToolRegistry:
    _tools: Dict[str, ToolEntry]
    _toolset_checks: Dict[str, Callable]

    def register(name, toolset, schema, handler, check_fn, requires_env)
    def get(name) -> ToolEntry
    def dispatch(name, args, ...) -> result
    def list_available(toolsets) -> list
```

### Tool Registration Flow

1. Each tool file (e.g., `web_tools.py`) calls `registry.register()` at import
2. `model_tools.py` imports all tool modules, triggering discovery
3. `AIAgent` queries registry for enabled tools, builds OpenAI-compatible schemas
4. LLM receives tool schemas; tool calls dispatched back through registry

### Parallel Execution

- Up to **8 concurrent tool workers** (via `asyncio.gather`)
- `clarify` tool forces sequential execution (requires user interaction)
- Tool delay configurable for rate-limiting
- Retry with exponential backoff on transient errors

### IterationBudget

Tracks LLM calls across parent agent and all subagents. Prevents runaway recursive loops when agents delegate to other agents.

---

## Session Storage (SQLite)

**File:** `hermes_state.py` (824 lines)

### Schema (v4)

```sql
sessions(
    id TEXT PRIMARY KEY,
    source TEXT,               -- 'cli' | 'telegram' | 'discord' | ...
    user_id TEXT,
    model TEXT,
    model_config TEXT,
    system_prompt TEXT,
    parent_session_id TEXT,    -- Lineage chain for compressed sessions
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    end_reason TEXT,
    message_count INTEGER,
    tool_call_count INTEGER,
    input_tokens INTEGER,
    output_tokens INTEGER,
    title TEXT
)

messages(
    id TEXT PRIMARY KEY,
    session_id TEXT,
    role TEXT,                 -- user | assistant | tool
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,
    tool_name TEXT,
    timestamp TIMESTAMP,
    token_count INTEGER,
    finish_reason TEXT
)

messages_fts                   -- FTS5 virtual table (auto-indexed)
```

### Key Features

- **WAL mode** — concurrent readers + single writer (gateway-safe)
- **FTS5 triggers** — auto-index all messages on insert/update/delete
- **Session lineage** — `parent_session_id` chain for compression-split sessions
- **Source tagging** — filter by platform origin
- **Token tracking** — cumulative input/output counts per session
- **Prefix resolution** — resolve `"sess_abc"` to full UUID

**Default path:** `~/.hermes/state.db` (override via `HERMES_HOME`)

---

## Prompt Assembly

**File:** `agent/prompt_builder.py`

System prompt constructed from (in order):

1. **Identity block** — from `~/.hermes/SOUL.md` or built-in default
2. **Context files** — `AGENTS.md`, `.cursorrules` in current directory (scanned for prompt injection before loading)
3. **Skills index** — names + descriptions of all loaded skills
4. **Memory guidance** — instructions for using memory tool
5. **Session search guidance** — instructions for using session_search tool
6. **Platform hints** — formatting and behavior rules per platform
7. **Honcho context** — prefetched user model (if Honcho enabled)

### Prompt Injection Protection

Before loading any context file, the prompt builder scans for injection patterns:
- Instructions to ignore previous directives
- Attempts to override the system prompt
- Commands to exfiltrate data

Suspicious files are rejected with a warning.

---

## Provider Router

**File:** `hermes_cli/models.py` + `provider_runtime.py`

Unified `call_llm()` / `async_call_llm()` for all LLM consumers in the system:

- Main agent loop
- Context compression (auxiliary model)
- Vision analysis (auxiliary model)
- Web content extraction (auxiliary model)
- Session search summarization (auxiliary model)
- Skills Hub assistance (auxiliary model)
- MCP tool invocation (auxiliary model)

**Resolution order:**
1. Per-call override (e.g., cron job model override)
2. Config file `model:` field
3. Environment variables
4. Fallback to default

---

## Messaging Gateway Architecture

```
Messaging Platforms
  [Telegram] [Discord] [Slack] [WhatsApp] [Signal] [Email] [HomeAsst]
        ↓           ↓           ↓
Platform Adapter (BasePlatformAdapter)
  - connect() / disconnect()
  - handle_message()
  - send(text / image / audio)
        ↓
Message Event Dispatcher
  - Extract SessionSource (platform, chat_id, user_id, thread_id)
  - Build session key
  - Evaluate reset policy
  - Inject session context
        ↓
Session Management (hermes_state.py)
  - SQLite state per platform session
  - File-based logs
  - Optional Honcho sync
        ↓
AIAgent (shared core)
        ↓
Response Router
  - Chunk long messages per platform limits
  - Route to delivery target
  - Mirror to gateway session
        ↓
Platform send methods
```

**Session key format:** `{platform}:{chat_id}` or `{platform}:{chat_id}:{thread_id}`

---

## Memory Architecture

Three tiers:

| Tier | Storage | Scope | Tools |
|------|---------|-------|-------|
| Inference memory | Context window | Current session | (none — automatic) |
| Procedural skills | `~/.hermes/skills/*/SKILL.md` | All sessions | `skills_list`, `skill_view`, `skill_manage` |
| Session history | SQLite FTS5 | All sessions | `session_search` |
| User model | Honcho API | Cross-device | `honcho_profile`, `honcho_search`, `honcho_context`, `honcho_conclude` |

---

## ACP Protocol

**File:** `acp_adapter/` (Agent Communication Protocol)

Runs as a stdio subprocess. Editors (VS Code, Zed, JetBrains) communicate via JSON-RPC.

**Session lifecycle:** `initialize → authenticate → new_session → [prompt]* → cancel? → disconnect`

**Streaming events:** `session_update` events carry thinking blocks, tool progress, step completions, and message content in real time.

See [ACP Integration](acp.md) for full details.

---

## Cron Scheduler

**File:** `cron/scheduler.py`

- **Tick interval:** 60 seconds
- **Locking:** File lock prevents concurrent ticks
- **Job persistence:** `~/.hermes/cron/jobs.json`
- **Output:** `~/.hermes/cron/output/{job_id}/{timestamp}.md`
- **Delivery:** origin | local | telegram | discord | slack | email

See [Memory & Scheduling](memory.md) for full details.

---

## RL / Trajectory Generation

**Files:** `batch_runner.py`, `trajectory_compressor.py`, `rl_cli.py`

Hermes includes a research-grade pipeline for generating supervised fine-tuning (SFT) and RL training data:

1. **`batch_runner.py`** — Run agent on seed tasks, record tool call trajectories
2. **`trajectory_compressor.py`** — Optimize transcripts (remove redundancy, summarize verbose sections, preserve action semantics)
3. **Atropos environments** (`environments/`) — RL evaluation environments for GRPO/PPO training
4. **`rl_cli.py`** — `rl_list_environments`, `rl_start_training`, `rl_check_status`, etc.

This pipeline powers NousResearch's agentic RL training for Hermes model family improvements.

---

## Self-Evolution System

A companion repository (`NousResearch/hermes-agent-self-evolution`) uses DSPy + GEPA (Generalized Evolutionary Prompt Adaptation) to autonomously optimize:

- **Phase 1 (current):** Skill document quality improvement
- **Planned:** Tool description optimization, system prompt optimization, code optimization

The self-evolution loop runs periodically, proposes improvements to skills/prompts, validates them against held-out test sets, and opens PRs for human review.
