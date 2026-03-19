# Architecture

This page is the authoritative map of Hermes Agent internals for v0.3.0 (v2026.3.17). The project is organized around a shared agent core that serves multiple platform frontends, a centralized provider router, a SQLite session store, and a set of loosely-coupled optional subsystems.

---

## High-Level Component Diagram

```
┌─── Entry Points ─────────────────────────────────────────────────────────┐
│  hermes          Interactive terminal CLI (cli.py)                       │
│  hermes-agent    Programmatic runner (run_agent.py main())               │
│  hermes-acp      ACP editor integration server (acp_adapter/entry.py)   │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
                               ▼
┌─── Core Agent Loop ─────────────────────────────────────────────────────┐
│  AIAgent  (run_agent.py)                                                │
│                                                                         │
│  ┌── Prompt Assembly ──────────────────────────────────────────────┐    │
│  │  agent/prompt_builder.py                                        │    │
│  │  load_soul_md() → SOUL.md identity                             │    │
│  │  build_context_files_prompt() → AGENTS.md, .cursorrules        │    │
│  │  build_skills_system_prompt() → skills index                   │    │
│  │  PLATFORM_HINTS → platform-specific guidance                   │    │
│  │  MEMORY_GUIDANCE, SESSION_SEARCH_GUIDANCE, SKILLS_GUIDANCE     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌── LLM Inference ────────────────────────────────────────────────┐    │
│  │  Three API modes: chat_completions / codex_responses /          │    │
│  │                   anthropic_messages                            │    │
│  │  agent/auxiliary_client.py → call_llm() / async_call_llm()     │    │
│  │  agent/anthropic_adapter.py → native Anthropic support         │    │
│  │  agent/smart_model_routing.py → cheap vs strong routing        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌── Tool Execution ───────────────────────────────────────────────┐    │
│  │  model_tools.py → get_tool_definitions(), handle_function_call()│    │
│  │  Sequential (single or interactive tools)                       │    │
│  │  Concurrent via ThreadPoolExecutor (multiple non-interactive)   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌── Context Management ───────────────────────────────────────────┐    │
│  │  agent/context_compressor.py → ContextCompressor                │    │
│  │  agent/prompt_caching.py → Anthropic cache_control markers      │    │
│  │  agent/model_metadata.py → context length resolution            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
                               ▼
┌─── Session Storage ──────────────────────────────────────────────────────┐
│  hermes_state.py → SessionDB                                            │
│  SQLite WAL mode, schema v5, FTS5 full-text search                      │
│  Default: ~/.hermes/state.db (override via HERMES_HOME)                 │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
          ┌────────────────────┼───────────────────────┐
          │                   │                       │
          ▼                   ▼                       ▼
┌─── Messaging ──────┐ ┌─── ACP ─────────────┐ ┌─── Cron ───────────┐
│  gateway/          │ │  acp_adapter/        │ │  cron/             │
│  8 platform        │ │  stdio/JSON-RPC      │ │  scheduler + jobs  │
│  adapters          │ │  VS Code, Zed,       │ │  SQLite-backed     │
│  Session routing   │ │  JetBrains           │ │  60s tick interval │
│  Delivery          │ └──────────────────────┘ └────────────────────┘
└────────────────────┘

┌─── Optional Subsystems ──────────────────────────────────────────────────┐
│  honcho_integration/ → cross-session user modeling                      │
│  environments/       → RL benchmark framework                           │
│  tools/              → 40+ tool implementations                         │
│  skills/             → 70+ bundled skill documents                      │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Agent Loop: How run_agent.py Orchestrates Everything

The core orchestration engine is `AIAgent` in `run_agent.py`. It is the single class used by all entry points — CLI, gateway, ACP, cron, and batch runner.

### Core Responsibilities

`AIAgent` is responsible for:

- Assembling the effective prompt and tool schemas
- Selecting the correct provider and API mode
- Making interruptible model calls
- Executing tool calls (sequentially or concurrently)
- Maintaining session history in memory and in the `SessionDB`
- Handling compression, retries, and fallback models
- Firing platform-specific callbacks during execution

### Turn Lifecycle

Each call to `run_conversation()` follows this path:

```
run_conversation()
  → generate effective task_id
  → append current user message to conversation history
  → load or build cached system prompt (assembled once, reused across turns)
  → maybe preflight-compress (rough token estimate before API call)
  → build api_messages from conversation history
  → inject ephemeral prompt layers (not persisted to cached system prompt)
  → apply prompt caching markers if appropriate (Anthropic / Claude-via-OpenRouter)
  → make interruptible API call (can be cancelled from CLI or gateway)
  → parse response:
      if tool calls → execute them (sequential or concurrent)
                    → append tool results to history
                    → loop back to API call
      if final text → persist to SessionDB
                    → cleanup
                    → return response to caller
```

### API Modes

Hermes currently supports three API execution modes:

| Mode | Used For |
|------|---------|
| `chat_completions` | OpenAI-compatible chat endpoints, OpenRouter, most custom endpoints |
| `codex_responses` | OpenAI Codex / Responses API path |
| `anthropic_messages` | Native Anthropic Messages API |

The mode is resolved from explicit arguments, provider selection, and base URL heuristics.

### Interruptible API Calls

Hermes wraps API requests so they can be interrupted from the CLI or gateway. This handles:

- Long LLM calls when the user sends a new message mid-flight
- Gateway cancellation semantics when a session resets
- Background system cancellation during shutdown

### Tool Execution Modes

Two strategies are used:

- **Sequential execution** — for single tool calls or interactive tools (such as `clarify`)
- **Concurrent execution** — for multiple non-interactive tools via `ThreadPoolExecutor`

Concurrent execution preserves message and result ordering when reinserting tool responses into conversation history.

### Callback Surfaces

`AIAgent` supports platform-specific callbacks:

| Callback | Purpose |
|---------|---------|
| `tool_progress_callback` | Fires when a tool starts or completes |
| `thinking_callback` | Fires when an extended thinking block is received |
| `reasoning_callback` | Fires when a reasoning step completes |
| `clarify_callback` | Fires when user clarification is requested |
| `step_callback` | Fires at iteration milestones |
| `message_callback` | Fires when final message content is available |

These are how the CLI, gateway, and ACP integrations stream intermediate progress and handle interactive approval and clarification flows.

### Budget and Fallback Behavior

Hermes tracks a shared iteration budget across parent agents and all subagents it spawns. Near the end of the available iteration window, it injects budget pressure hints into tool results. Fallback model support allows the agent to switch providers or models when the primary route fails in supported failure paths.

---

## Context Compression: How It Works

**Primary file:** `agent/context_compressor.py` — `ContextCompressor` class

### When Compression Is Triggered

`ContextCompressor` tracks actual token counts from API responses. Compression fires when:

- `should_compress(prompt_tokens)` returns `True` — actual token count exceeds `threshold_percent` of the model's context length (default: 50%)
- `should_compress_preflight(messages)` returns `True` — rough estimate before an API call (avoids 413 errors)

The threshold is tuned to 50% in v0.3.0 for more proactive compression.

### Compression Algorithm

```
1. Check: does message count exceed protect_first_n + protect_last_n + 1?
2. Set compress_start = protect_first_n (default: 3)
3. Set compress_end = n_messages - protect_last_n (default: 4)
4. Align compress_start forward past any orphan tool results
5. Align compress_end backward to avoid splitting tool_call/result groups
6. Extract turns_to_summarize = messages[compress_start:compress_end]
7. Call _generate_summary() via auxiliary model (task="compression")
8. Build compressed message list:
   - Preserve messages[0:compress_start] (head)
   - Insert summary message with SUMMARY_PREFIX
   - Preserve messages[compress_end:] (tail)
9. Run _sanitize_tool_pairs() to fix orphaned tool_call/tool_result pairs
10. Increment compression_count
```

The `SUMMARY_PREFIX` marker is `"[CONTEXT COMPACTION] Earlier turns in this conversation were compacted to save context space..."` and is prepended to every summary.

### Tool-Pair Sanitization

After compression, `_sanitize_tool_pairs()` handles two failure modes that would cause API rejections:

1. **Orphaned tool results** — a tool result references a call_id whose assistant message was compressed away. These results are removed.
2. **Calls without results** — an assistant message has `tool_calls` whose result messages were compressed away. Stub result messages (`"[Result from earlier conversation — see context summary above]"`) are inserted.

### Pre-Compression Memory Flush

Before compression, Hermes can give the model one last chance to persist memory so facts are not lost when middle turns are summarized away.

### Session Lineage After Compression

Compression can split the session into a new `session_id` while preserving ancestry via `parent_session_id` in the `SessionDB`. This allows the agent to continue with a smaller active context while retaining a searchable ancestry chain for session history and the `session_search` tool.

### Re-Injected State After Compression

After compression, Hermes may re-inject compact operational state such as:

- Todo snapshot
- Prior-read-files summary

### Trajectory Compressor (Separate Tool)

`trajectory_compressor.py` is a separate `TrajectoryCompressor` class for post-processing completed agent trajectories for training datasets. It is distinct from the runtime `ContextCompressor`. Key differences:

| Feature | `ContextCompressor` | `TrajectoryCompressor` |
|---------|--------------------|-----------------------|
| Purpose | Runtime context management | Offline training data preparation |
| Input | Live conversation messages | JSONL trajectory files |
| Output | Compressed messages list | Compressed JSONL files |
| Token counting | Rough estimate or API response | HuggingFace tokenizer |
| Concurrency | Synchronous | Async with `asyncio.gather`, semaphore |
| Default model | Configured auxiliary model | `google/gemini-3-flash-preview` via OpenRouter |
| Target tokens | Context limit threshold | `target_max_tokens` (default: 15,250) |

`TrajectoryCompressor` protects the first system, human, gpt, and tool turns plus the last N turns, and emits rich metrics (`TrajectoryMetrics`, `AggregateMetrics`) including compression ratios and token savings.

---

## Prompt Assembly: System Prompt Layers

**Primary file:** `agent/prompt_builder.py`

Hermes deliberately separates the **cached system prompt** (stable across turns) from **ephemeral API-call-time additions** (not persisted). This is critical for prompt caching effectiveness and memory correctness.

### Cached System Prompt Layers (in order)

1. **Agent identity** — `SOUL.md` loaded from `HERMES_HOME` via `load_soul_md()` when available; falls back to `DEFAULT_AGENT_IDENTITY` in `prompt_builder.py`
2. **Tool-aware behavior guidance** — `MEMORY_GUIDANCE`, `SESSION_SEARCH_GUIDANCE`, `SKILLS_GUIDANCE` constants
3. **Honcho static block** — when Honcho integration is active
4. **Optional system message** — user-provided system message override
5. **Frozen MEMORY snapshot** — memory tool contents at session start
6. **Frozen USER profile snapshot** — user profile at session start
7. **Skills index** — built by `build_skills_system_prompt()`, scanning `~/.hermes/skills/` for `SKILL.md` files grouped by category; filtered by platform compatibility and conditional activation rules
8. **Context files** — discovered by `build_context_files_prompt()`: `AGENTS.md` (hierarchical, recursive), `.cursorrules`, `.cursor/rules/*.mdc`, `.hermes.md`/`HERMES.md` (walks to git root); `SOUL.md` is excluded here when already loaded as identity in step 1
9. **Timestamp and optional session ID** — from `hermes_time.now()` (timezone-aware, configured via `HERMES_TIMEZONE` or `config.yaml`)
10. **Platform hint** — from `PLATFORM_HINTS` dict, keyed by platform name

When `skip_context_files` is set (for subagent delegation), `SOUL.md` is not loaded and the hardcoded `DEFAULT_AGENT_IDENTITY` is used instead.

### API-Call-Time-Only Layers (not persisted)

These are intentionally excluded from the cached system prompt:

- `ephemeral_system_prompt` — injected per-call
- Prefill messages
- Gateway-derived session context overlays
- Honcho recall injected into the current-turn user message (kept out of the cached prefix to preserve cache stability)

### Context File Security Scanning

`build_context_files_prompt()` and `load_soul_md()` run `_scan_context_content()` on every context file before injection. The scanner checks for:

- Invisible Unicode characters (zero-width, directional overrides)
- Prompt injection patterns (`"ignore previous instructions"`, `"system prompt override"`, `"act as if you have no restrictions"`, etc.)
- Data exfiltration commands (`curl ... $KEY`, `cat .env`)

Files that match threat patterns are blocked and replaced with a warning message.

### Long File Truncation

Files exceeding `CONTEXT_FILE_MAX_CHARS` (20,000 characters) are truncated using head/tail strategy: 70% from the head and 20% from the tail, with a marker in the middle.

### Skills Index Format

`build_skills_system_prompt()` produces a compact index in the format:

```
## Skills (mandatory)
<available_skills>
  category: description
    - skill-name: description
    - skill-name
  category:
    - skill-name: description
</available_skills>
```

Skills are filtered by:
- Platform compatibility (`skill_matches_platform()` from `tools/skills_tool.py`)
- Disabled skills list
- Conditional activation: `fallback_for_toolsets`, `requires_toolsets`, `fallback_for_tools`, `requires_tools` from skill frontmatter

---

## Session Storage: hermes_state.py

**File:** `hermes_state.py` — `SessionDB` class

### Database Location

```
~/.hermes/state.db
```

Override via the `HERMES_HOME` environment variable. The file is created automatically.

### Schema (v5)

The `SessionDB` maintains schema version 5. Migrations run automatically on startup.

**`sessions` table:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PRIMARY KEY | Session UUID |
| `source` | TEXT | Platform origin: `cli`, `telegram`, `discord`, `slack`, etc. |
| `user_id` | TEXT | Platform user identifier |
| `model` | TEXT | Active model slug |
| `model_config` | TEXT | JSON serialized model configuration |
| `system_prompt` | TEXT | Snapshot of the assembled system prompt |
| `parent_session_id` | TEXT | Foreign key to parent session after compression split |
| `started_at` | REAL | Unix timestamp of session creation |
| `ended_at` | REAL | Unix timestamp when session ended |
| `end_reason` | TEXT | Why the session ended |
| `message_count` | INTEGER | Total message count |
| `tool_call_count` | INTEGER | Total tool call count |
| `input_tokens` | INTEGER | Cumulative input tokens |
| `output_tokens` | INTEGER | Cumulative output tokens |
| `cache_read_tokens` | INTEGER | Cumulative cache-read tokens (v5) |
| `cache_write_tokens` | INTEGER | Cumulative cache-write tokens (v5) |
| `reasoning_tokens` | INTEGER | Cumulative reasoning tokens (v5) |
| `billing_provider` | TEXT | Provider used for billing (v5) |
| `billing_base_url` | TEXT | Endpoint used for billing (v5) |
| `billing_mode` | TEXT | Billing mode: `official_docs_snapshot`, `subscription_included`, etc. (v5) |
| `estimated_cost_usd` | REAL | Estimated cost in USD (v5) |
| `actual_cost_usd` | REAL | Actual cost in USD from provider (v5) |
| `cost_status` | TEXT | `actual`, `estimated`, `included`, `unknown` (v5) |
| `cost_source` | TEXT | Source of pricing data (v5) |
| `pricing_version` | TEXT | Pricing snapshot version identifier (v5) |
| `title` | TEXT | Human-readable session title (unique, optional) |

**`messages` table:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Row ID |
| `session_id` | TEXT | Foreign key to sessions |
| `role` | TEXT | `user`, `assistant`, `tool` |
| `content` | TEXT | Message content |
| `tool_call_id` | TEXT | Tool call identifier for tool results |
| `tool_calls` | TEXT | JSON serialized tool call list |
| `tool_name` | TEXT | Tool name for tool result messages |
| `timestamp` | REAL | Unix timestamp |
| `token_count` | INTEGER | Per-message token estimate |
| `finish_reason` | TEXT | Model finish reason |

**`messages_fts` virtual table:** FTS5 full-text search over message content, with triggers to auto-index on insert, update, and delete.

### Indexes

```sql
idx_sessions_source     ON sessions(source)
idx_sessions_parent     ON sessions(parent_session_id)
idx_sessions_started    ON sessions(started_at DESC)
idx_sessions_title_unique ON sessions(title) WHERE title IS NOT NULL
idx_messages_session    ON messages(session_id, timestamp)
```

### Thread Safety

`SessionDB` uses WAL (Write-Ahead Logging) mode and a `threading.Lock()` for all write operations. This supports the common gateway pattern of multiple reader threads and a single writer.

### Session Lineage

When `ContextCompressor` splits a session, it continues in a new `session_id` while setting `parent_session_id` to the previous session. The `resolve_session_by_title()` method follows numbered lineage variants (e.g., `"my session"` → `"my session #2"`) to find the most recent continuation of a named session.

### FTS5 Search

`search_messages()` performs full-text search with FTS5 MATCH syntax. It sanitizes user query input via `_sanitize_fts5_query()` (preserves quoted phrases, strips special characters, wraps hyphenated terms), then returns matching messages with a snippet, surrounding context (1 message before and after), and session metadata.

### Gateway vs CLI Persistence

- CLI uses `SessionDB` directly for resume, history, and search via `hermes sessions`
- Gateway maintains active-session mappings per platform and also writes to `SessionDB`
- Some legacy JSON/JSONL artifacts exist for compatibility, but SQLite is the canonical historical store

---

## Trajectory Format: What Gets Logged

**Primary files:** `agent/trajectory.py`, `run_agent.py`, `batch_runner.py`, `trajectory_compressor.py`

### Purpose

Trajectory outputs are used for:

- SFT (supervised fine-tuning) data generation
- Debugging agent behavior
- Benchmark and evaluation artifact capture
- Post-processing and compression pipelines for training

### Normalization Strategy

Hermes converts live conversation structure into a training-friendly format:

- Reasoning is represented in explicit markup (think blocks converted via `convert_scratchpad_to_think()`)
- Tool calls are converted into structured XML-like regions for dataset compatibility
- Tool outputs are grouped appropriately
- Successful and failed trajectories are separated

### Persistence Boundaries

Trajectory files do not blindly mirror all runtime prompt state. Some prompt-time-only layers are intentionally excluded from persisted trajectory content so datasets are cleaner and less environment-specific.

### Batch Runner Metadata

`batch_runner.py` emits richer metadata than single-session trajectory saving:

- Model and provider metadata
- Toolset information
- Partial and failure markers
- Tool statistics

### TrajectoryMetrics (trajectory_compressor.py)

When `TrajectoryCompressor` processes a JSONL file, each entry receives optional `compression_metrics`:

```json
{
  "original_tokens": 18500,
  "compressed_tokens": 14200,
  "tokens_saved": 4300,
  "compression_ratio": 0.7676,
  "original_turns": 32,
  "compressed_turns": 28,
  "turns_removed": 4,
  "compression_region": {
    "start_idx": 4,
    "end_idx": 8,
    "turns_count": 4
  },
  "was_compressed": true,
  "still_over_limit": false,
  "skipped_under_target": false,
  "summarization_api_calls": 1,
  "summarization_errors": 0
}
```

---

## Auxiliary Client: What It Does

**File:** `agent/auxiliary_client.py`

The auxiliary client is a shared routing layer that handles all side-task LLM calls — everything except the main agent loop. It provides a single resolution chain so every consumer picks up the best available backend without duplicating fallback logic.

### Consumers of the Auxiliary Client

| Consumer | Task Name | Purpose |
|---------|-----------|---------|
| `ContextCompressor._generate_summary()` | `compression` | Summarize middle turns |
| `tools/session_search_tool.py` | `session_search` | Summarize session search results |
| `tools/web_tools.py` | `web_extract` | Extract content from web pages |
| `tools/vision_tools.py` | `vision` | Analyze images |
| `tools/browser_tool.py` | `vision` | Analyze browser screenshots |
| `tools/skills_tool.py` | `skills_hub` | Skills Hub assistance |
| `tools/memory_tool.py` | `flush_memories` | Flush memories before compression |
| `trajectory_compressor.py` | Provider-direct | Summarize trajectory regions |

### Centralized API: call_llm() and async_call_llm()

The primary public API is:

```python
call_llm(
    task: str = None,        # "compression", "vision", "web_extract", etc.
    provider: str = None,    # Explicit provider override
    model: str = None,       # Explicit model override
    base_url: str = None,    # Direct endpoint override
    api_key: str = None,     # Direct API key override
    messages: list,          # Chat messages
    temperature: float = None,
    max_tokens: int = None,
    tools: list = None,
    timeout: float = 30.0,
    extra_body: dict = None,
) -> Any  # Response with .choices[0].message.content
```

`async_call_llm()` is the async counterpart with identical signature.

### Provider Resolution Order (Text Tasks, Auto Mode)

1. OpenRouter (`OPENROUTER_API_KEY`)
2. Nous Portal (`~/.hermes/auth.json` active provider)
3. Custom endpoint (`OPENAI_BASE_URL` + `OPENAI_API_KEY`)
4. Codex OAuth (Responses API via `chatgpt.com` with `gpt-5.2-codex`)
5. Native Anthropic
6. Direct API-key providers: z.ai/GLM, Kimi/Moonshot, MiniMax, MiniMax-CN
7. None (raises `RuntimeError`)

### Provider Resolution Order (Vision Tasks, Auto Mode)

1. Selected main provider (if it is a supported vision backend)
2. OpenRouter
3. Nous Portal
4. Codex OAuth (gpt-5.2-codex supports vision via Responses API)
5. Native Anthropic
6. Custom endpoint (local vision models: Qwen-VL, LLaVA, Pixtral, etc.)
7. None

### Per-Task Override Configuration

Each auxiliary task supports per-task provider and model overrides via:

- Environment variables: `AUXILIARY_{TASK}_PROVIDER`, `AUXILIARY_{TASK}_MODEL`, `AUXILIARY_{TASK}_BASE_URL`
- Config file: `auxiliary.{task}.provider`, `auxiliary.{task}.model`, `auxiliary.{task}.base_url`
- Legacy: `compression.summary_provider`, `compression.summary_model`

### Codex Adapter

The `_CodexCompletionsAdapter` class provides a `chat.completions.create()`-compatible interface that internally translates to the Codex Responses API. This makes Codex transparent to all auxiliary consumers. The adapter converts content formats (`text`/`image_url` → `input_text`/`input_image`), handles tools, and streams the response to reconstruct a chat-completions-compatible result.

### Anthropic Adapter

`AnthropicAuxiliaryClient` wraps the native Anthropic client in a `chat.completions.create()`-compatible interface via `_AnthropicCompletionsAdapter`. It uses `build_anthropic_kwargs()` and `normalize_anthropic_response()` from `agent/anthropic_adapter.py`.

### Client Caching

Resolved clients are cached in `_client_cache` keyed by `(provider, async_mode, base_url, api_key)`. The cache uses a `threading.Lock()` for thread-safe initialization.

---

## Smart Model Routing

**File:** `agent/smart_model_routing.py`

### What It Does

Smart model routing is an optional feature that can route simple conversational turns to a cheaper, faster model instead of the primary model configured by the user.

### When It Activates

`choose_cheap_model_route()` activates only when:

- `routing_config.enabled` is `True` in the user's configuration
- `cheap_model.provider` and `cheap_model.model` are specified in `routing_config`
- The user message passes all simplicity filters (see below)

### Decision Logic

`choose_cheap_model_route()` is conservative by design. It returns `None` (keep the primary model) if any of the following are true:

- Message exceeds `max_simple_chars` (default: 160 characters)
- Message exceeds `max_simple_words` (default: 28 words)
- Message contains more than 1 newline
- Message contains backticks or code fences (` ``` `)
- Message contains a URL (`https://` or `www.`)
- Message contains any word from `_COMPLEX_KEYWORDS`:
  `debug`, `implement`, `refactor`, `patch`, `traceback`, `stacktrace`, `exception`, `error`, `analyze`, `analysis`, `investigate`, `architecture`, `design`, `compare`, `benchmark`, `optimize`, `review`, `terminal`, `shell`, `tool`, `tools`, `pytest`, `test`, `tests`, `plan`, `planning`, `delegate`, `subagent`, `cron`, `docker`, `kubernetes`

### Route Resolution

When cheap routing activates, `resolve_turn_route()` calls `resolve_runtime_provider()` with the cheap provider's configuration and returns a route dict with `model`, `runtime`, `label`, and `signature` fields. The label field is set to `"smart route → {model} ({provider})"` for display purposes.

---

## Design Themes

Several cross-cutting design themes appear throughout the codebase (from the official architecture documentation):

- **Prompt stability matters** — the system prompt is assembled once per session and the stable prefix must remain stable for provider-side prompt caching to be effective
- **Tool execution must be observable and interruptible** — every tool call fires callbacks; long LLM calls can be cancelled
- **Session persistence must survive long-running use** — every message and token count is written to SQLite immediately; WAL mode allows concurrent readers
- **Platform frontends should share one agent core** — `AIAgent` is identical whether called from CLI, gateway, ACP, or cron
- **Optional subsystems should remain loosely coupled** — Honcho, MCP, and ACP are injected via callbacks and configuration, not hardwired into the agent loop

---

## Source Files Reference

| File | Description |
|------|-------------|
| `run_agent.py` | `AIAgent` — core orchestration loop (entry point for all platforms) |
| `cli.py` | Interactive terminal UI (prompt_toolkit TUI) |
| `model_tools.py` | Tool discovery, schema building, and dispatch |
| `toolsets.py` | Tool groupings and platform presets |
| `hermes_state.py` | `SessionDB` — SQLite session and message store |
| `hermes_constants.py` | Shared API endpoint constants |
| `hermes_time.py` | `now()` — timezone-aware clock |
| `batch_runner.py` | Batch trajectory generation for RL/SFT |
| `trajectory_compressor.py` | `TrajectoryCompressor` — offline training data compression |
| `agent/context_compressor.py` | `ContextCompressor` — runtime context management |
| `agent/prompt_builder.py` | System prompt assembly functions |
| `agent/auxiliary_client.py` | `call_llm()`, `resolve_provider_client()` — centralized LLM router |
| `agent/model_metadata.py` | Context length resolution, token estimation, pricing |
| `agent/smart_model_routing.py` | `choose_cheap_model_route()` — optional cheap model routing |
| `agent/usage_pricing.py` | `estimate_usage_cost()`, `normalize_usage()`, `CanonicalUsage` |
| `agent/prompt_caching.py` | Anthropic cache_control marker application |
| `agent/anthropic_adapter.py` | Native Anthropic API translation |
| `agent/trajectory.py` | Trajectory normalization and saving |
| `gateway/` | Platform adapters (Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant) |
| `cron/` | Scheduled job storage and scheduler |
| `honcho_integration/` | Honcho memory integration |
| `acp_adapter/` | ACP editor integration server |
| `environments/` | RL/benchmark environment framework |
| `skills/` | Bundled skill documents |
| `optional-skills/` | Official optional skills |
| `tools/` | Tool implementations and terminal environments |
