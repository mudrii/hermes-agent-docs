# Architecture

This page is the authoritative map of Hermes Agent internals, updated through v0.5.0 (v2026.3.28). The project is organized around a shared agent core that serves multiple platform frontends, a centralized provider router, a SQLite session store, and a set of loosely-coupled optional subsystems.

---

## High-Level Component Diagram

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
|  |  load_soul_md() -> SOUL.md identity                              | |
|  |  build_context_files_prompt() -> AGENTS.md, .cursorrules,        | |
|  |                                  .hermes.md / HERMES.md          | |
|  |  build_skills_system_prompt() -> skills index                    | |
|  |  PLATFORM_HINTS -> platform-specific guidance                    | |
|  |  MEMORY_GUIDANCE, SESSION_SEARCH_GUIDANCE, SKILLS_GUIDANCE       | |
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
|  New in v0.5.0: reasoning, reasoning_details, codex_reasoning_items  |
|  Default: ~/.hermes/state.db (override via HERMES_HOME)              |
+------------------------------+----------------------------------------+
                               |
          +--------------------+------------------------+
          |                    |                        |
          v                    v                        v
+--- Messaging --------+ +--- ACP ----------------+ +--- Cron -----------+
|  gateway/            | |  acp_adapter/           | |  cron/             |
|  8 platform          | |  stdio/JSON-RPC         | |  scheduler + jobs  |
|  adapters            | |  VS Code, Zed,          | |  SQLite-backed     |
|  Session routing     | |  JetBrains              | |  60s tick interval |
|  Delivery            | +------------------------+ +--------------------+
+----------------------+

+--- Optional Subsystems -----------------------------------------------+
|  honcho_integration/ -> cross-session user modeling                   |
|  environments/       -> RL benchmark framework                        |
|  tools/              -> 40+ tool implementations                      |
|  skills/             -> 70+ bundled skill documents                   |
+----------------------------------------------------------------------+
```

---

## Agent Loop: How run_agent.py Orchestrates Everything

The core orchestration engine is `AIAgent` in `run_agent.py`. It is the single class used by all entry points -- CLI, gateway, ACP, cron, and batch runner.

### Core Responsibilities

`AIAgent` is responsible for:

- Assembling the effective prompt and tool schemas
- Selecting the correct provider and API mode
- Making interruptible model calls (streaming or non-streaming)
- Executing tool calls (sequentially or concurrently)
- Maintaining session history in memory and in the `SessionDB`
- Handling compression, retries, and fallback models
- Firing platform-specific callbacks during execution

### AIAgent Constructor Parameters (Selected)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `base_url` | str | `OPENROUTER_BASE_URL` | Base URL for the model API |
| `api_key` | str | None | API key for authentication |
| `provider` | str | `"openrouter"` | Provider identifier |
| `api_mode` | str | auto-detected | `"chat_completions"`, `"codex_responses"`, or `"anthropic_messages"` |
| `model` | str | `"anthropic/claude-opus-4.6"` | Model name (OpenRouter format) |
| `max_iterations` | int | 90 | Maximum tool-calling iterations (shared with subagents via `IterationBudget`) |
| `tool_delay` | float | 1.0 | Delay between tool calls in seconds |
| `stream_delta_callback` | callable | None | Callback fired with each text token during streaming |
| `tool_progress_callback` | callable | None | Callback when a tool starts or completes |
| `thinking_callback` | callable | None | Callback when an extended thinking block is received |
| `reasoning_callback` | callable | None | Callback when a reasoning step completes |
| `clarify_callback` | callable | None | Callback when user clarification is requested |
| `step_callback` | callable | None | Callback at iteration milestones |
| `platform` | str | None | `"cli"`, `"telegram"`, `"discord"`, etc. |
| `session_db` | SessionDB | None | SQLite session store |
| `iteration_budget` | IterationBudget | None | Shared budget across parent + subagents |
| `fallback_model` | dict | None | Fallback model configuration |
| `checkpoints_enabled` | bool | False | Enable filesystem checkpoints |
| `pass_session_id` | bool | False | Include session ID in system prompt |

### Turn Lifecycle

Each call to `run_conversation()` follows this path:

```
run_conversation(user_message, stream_callback=None, ...)
  -> generate effective task_id
  -> reset retry counters and iteration budget
  -> store stream_callback as self._stream_callback
  -> append current user message to conversation history
  -> load or build cached system prompt (assembled once, reused across turns)
  -> maybe preflight-compress (rough token estimate before API call)
  -> build api_messages from conversation history
  -> inject ephemeral prompt layers (not persisted to cached system prompt)
  -> apply prompt caching markers if appropriate (Anthropic / Claude-via-OpenRouter)
  -> make interruptible API call:
       if _has_stream_consumers() -> _interruptible_streaming_api_call()
       else -> _interruptible_api_call()
  -> parse response:
      if tool calls -> execute them (sequential or concurrent)
                    -> append tool results to history
                    -> loop back to API call
      if final text -> persist to SessionDB
                    -> maybe_auto_title() in background thread
                    -> cleanup
                    -> return response to caller
```

### API Modes

Hermes supports three API execution modes:

| Mode | Used For |
|------|---------|
| `chat_completions` | OpenAI-compatible chat endpoints, OpenRouter, most custom endpoints |
| `codex_responses` | OpenAI Codex / Responses API path |
| `anthropic_messages` | Native Anthropic Messages API |

The mode is resolved from explicit arguments, provider selection, and base URL heuristics. Detection logic in `AIAgent.__init__`:

- `api_mode="codex_responses"` when provider is `openai-codex` or base URL contains `chatgpt.com/backend-api/codex`
- `api_mode="anthropic_messages"` when provider is `anthropic`, base URL contains `api.anthropic.com`, or base URL ends in `/anthropic`
- `api_mode="chat_completions"` for everything else

### Interruptible API Calls

Hermes wraps API requests so they can be interrupted from the CLI or gateway. This handles:

- Long LLM calls when the user sends a new message mid-flight
- Gateway cancellation semantics when a session resets
- Background system cancellation during shutdown

### Tool Execution Modes

Two strategies are used:

- **Sequential execution** -- for single tool calls or interactive tools (e.g. `clarify`)
- **Concurrent execution** -- for multiple non-interactive tools via `ThreadPoolExecutor` (max `_MAX_TOOL_WORKERS = 8` workers)

Concurrent execution is decided by `_should_parallelize_tool_batch()`, which checks:

- Batch has more than 1 tool call
- No tool is in `_NEVER_PARALLEL_TOOLS` (currently: `{"clarify"}`)
- Every tool is either in `_PARALLEL_SAFE_TOOLS` (read-only tools like `read_file`, `search_files`, `session_search`, `web_search`, etc.) or is a `_PATH_SCOPED_TOOLS` tool (`read_file`, `write_file`, `patch`) targeting non-overlapping paths
- All tool call arguments parse as valid JSON dicts

Message and result ordering is preserved when reinserting tool responses into conversation history.

### Callback Surfaces

`AIAgent` supports platform-specific callbacks:

| Callback | Parameter | Purpose |
|---------|-----------|---------|
| `tool_progress_callback` | `__init__` | Fires when a tool starts or completes |
| `thinking_callback` | `__init__` | Fires when an extended thinking block is received |
| `reasoning_callback` | `__init__` | Fires when a reasoning step completes |
| `clarify_callback` | `__init__` | Fires when user clarification is requested |
| `step_callback` | `__init__` | Fires at iteration milestones |
| `stream_delta_callback` | `__init__` | Fires with each text token during streaming (persistent) |
| `stream_callback` | `run_conversation()` | Per-call stream callback (stored as `_stream_callback`; used for TTS) |

Both `stream_delta_callback` and `_stream_callback` are fired by `_fire_stream_delta()` for each text token. Either or both can be set independently.

### Budget and Fallback Behavior

Hermes tracks a shared iteration budget across parent agents and all subagents it spawns via the `IterationBudget` class. The budget is thread-safe (`threading.Lock`), supports `consume()` (returns False when exhausted), and `refund()` (for `execute_code` turns that should not count).

Budget pressure hints are injected into tool results at two thresholds:
- **70% consumed** (`_budget_caution_threshold`) -- nudge to start wrapping up
- **90% consumed** (`_budget_warning_threshold`) -- urgent, respond now

Fallback model support allows the agent to switch providers or models when the primary route fails.

---

## Context Compression: How It Works

**Primary file:** `agent/context_compressor.py` -- `ContextCompressor` class

### When Compression Is Triggered

`ContextCompressor` tracks actual token counts from API responses. Compression fires when:

- `should_compress(prompt_tokens)` returns `True` -- actual token count exceeds `threshold_percent` of the model's context length (default: 50%)
- `should_compress_preflight(messages)` returns `True` -- rough estimate before an API call (avoids 413 errors)

The threshold is tuned to 50% in v0.3.0 for more proactive compression.

### Constructor Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `model` | required | Model name for context length lookup |
| `threshold_percent` | `0.50` | Fraction of context window that triggers compression |
| `protect_first_n` | `3` | Number of head messages to preserve |
| `protect_last_n` | `4` | Number of tail messages to preserve |
| `summary_target_tokens` | `2500` | Target token count for the summary |
| `summary_model_override` | None | Force a specific model for summarization |

### Compression Algorithm

```
1. Check: does message count exceed protect_first_n + protect_last_n + 1?
2. Set compress_start = protect_first_n (default: 3)
3. Set compress_end = n_messages - protect_last_n (default: 4)
4. Align compress_start forward past any orphan tool results
5. Align compress_end backward to avoid splitting tool_call/result groups
6. Extract turns_to_summarize = messages[compress_start:compress_end]
7. Call _generate_summary() via call_llm(task="compression")
8. Build compressed message list:
   - Preserve messages[0:compress_start] (head)
   - Insert summary message with SUMMARY_PREFIX
   - Preserve messages[compress_end:] (tail)
9. Run _sanitize_tool_pairs() to fix orphaned tool_call/tool_result pairs
10. Increment compression_count
```

The `SUMMARY_PREFIX` marker is:

```
[CONTEXT COMPACTION] Earlier turns in this conversation were compacted to save
context space. The summary below describes work that was already completed, and
the current session state may still reflect that work (for example, files may
already be changed). Use the summary and the current state to continue from
where things left off, and avoid repeating work:
```

On the first compression, a note is also appended to the system message (message index 0): `"[Note: Some earlier conversation turns have been compacted into a handoff summary to preserve context space...]"`

### Summary Generation

`_generate_summary()` truncates each turn to 2000 characters (1000 head + 500 tail), appends tool call names, and sends the formatted content to `call_llm(task="compression", temperature=0.3, max_tokens=summary_target_tokens * 2)`. If the auxiliary model fails, it falls back to the user's main model. If all models fail, the middle turns are dropped without a summary.

### Summary Role Selection

The summary message role is chosen to avoid consecutive same-role messages with both the head neighbor and the tail neighbor. If neither `"user"` nor `"assistant"` avoids a collision, the summary is merged into the first tail message instead of being inserted as a standalone message.

### Tool-Pair Sanitization

After compression, `_sanitize_tool_pairs()` handles two failure modes that would cause API rejections:

1. **Orphaned tool results** -- a tool result references a call_id whose assistant message was compressed away. These results are removed.
2. **Calls without results** -- an assistant message has `tool_calls` whose result messages were compressed away. Stub result messages (`"[Result from earlier conversation -- see context summary above]"`) are inserted.

### Session Lineage After Compression

Compression can split the session into a new `session_id` while preserving ancestry via `parent_session_id` in the `SessionDB`. This allows the agent to continue with a smaller active context while retaining a searchable ancestry chain for session history and the `session_search` tool.

### Trajectory Compressor (Separate Tool)

`trajectory_compressor.py` is a separate `TrajectoryCompressor` class for post-processing completed agent trajectories for training datasets. It is distinct from the runtime `ContextCompressor`. Key differences:

| Feature | `ContextCompressor` | `TrajectoryCompressor` |
|---------|--------------------|-----------------------|
| Purpose | Runtime context management | Offline training data preparation |
| Input | Live conversation messages | JSONL trajectory files |
| Output | Compressed messages list | Compressed JSONL files |
| Token counting | Rough estimate or API response | HuggingFace tokenizer (`moonshotai/Kimi-K2-Thinking`) |
| Concurrency | Synchronous | Async with `asyncio.gather`, semaphore |
| Default model | Configured auxiliary model | `google/gemini-3-flash-preview` via OpenRouter |
| Target tokens | Context limit threshold | `target_max_tokens` (default: 15,250) |
| Summary target | 2,500 tokens | 750 tokens |
| Protected turns | First 3 + last 4 | First system/human/gpt/tool + last 4 |

`TrajectoryCompressor` emits rich metrics (`TrajectoryMetrics`) including compression ratios and token savings.

---

## Prompt Assembly: System Prompt Layers

**Primary file:** `agent/prompt_builder.py`

Hermes deliberately separates the **cached system prompt** (stable across turns) from **ephemeral API-call-time additions** (not persisted). This is critical for prompt caching effectiveness and memory correctness.

### Cached System Prompt Layers (in order)

1. **Agent identity** -- `SOUL.md` loaded from `HERMES_HOME` via `load_soul_md()` when available; falls back to `DEFAULT_AGENT_IDENTITY` in `prompt_builder.py`
2. **Tool-aware behavior guidance** -- `MEMORY_GUIDANCE`, `SESSION_SEARCH_GUIDANCE`, `SKILLS_GUIDANCE` constants
3. **Honcho static block** -- when Honcho integration is active
4. **Optional system message** -- user-provided system message override
5. **Frozen MEMORY snapshot** -- memory tool contents at session start
6. **Frozen USER profile snapshot** -- user profile at session start
7. **Skills index** -- built by `build_skills_system_prompt()`, scanning `~/.hermes/skills/` for `SKILL.md` files grouped by category; filtered by platform compatibility and conditional activation rules
8. **Context files** -- discovered by `build_context_files_prompt()`: `AGENTS.md` (hierarchical, recursive), `.cursorrules`, `.cursor/rules/*.mdc`, `.hermes.md`/`HERMES.md` (walks to git root); `SOUL.md` is excluded here when already loaded as identity in step 1
9. **Timestamp and optional session ID** -- from `hermes_time.now()` (timezone-aware, configured via `HERMES_TIMEZONE` env var or `timezone` key in `config.yaml`)
10. **Platform hint** -- from `PLATFORM_HINTS` dict, keyed by platform name

When `skip_context_files` is set (for subagent delegation), `SOUL.md` is not loaded and the hardcoded `DEFAULT_AGENT_IDENTITY` is used instead.

### API-Call-Time-Only Layers (not persisted)

These are intentionally excluded from the cached system prompt:

- `ephemeral_system_prompt` -- injected per-call
- Prefill messages
- Gateway-derived session context overlays
- Honcho recall injected into the current-turn user message via `_inject_honcho_turn_context()` (kept out of the cached prefix to preserve cache stability)

### Context File Security Scanning

`build_context_files_prompt()` and `load_soul_md()` run `_scan_context_content()` on every context file before injection. The scanner checks for:

- Invisible Unicode characters (zero-width `U+200B`-`U+200F`, `U+FEFF`, directional overrides `U+202A`-`U+202E`, `U+2060`-`U+2069`)
- Prompt injection patterns (`"ignore previous instructions"`, `"system prompt override"`, `"act as if you have no restrictions"`, `"disregard your rules"`, etc.)
- Data exfiltration commands (`curl ... $KEY`, `cat .env`, `cat credentials`)
- Hidden HTML elements (`<div style="display:none"`)

Files that match threat patterns are blocked and replaced with a `[BLOCKED: ...]` warning message.

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
- Platform compatibility (`skill_matches_platform()`)
- Disabled skills list
- Conditional activation: `fallback_for_toolsets`, `requires_toolsets`, `fallback_for_tools`, `requires_tools` from skill frontmatter

---

## Session Storage: hermes_state.py

**File:** `hermes_state.py` -- `SessionDB` class

### Database Location

```
~/.hermes/state.db
```

Override via the `HERMES_HOME` environment variable. The file is created automatically.

### Schema (v6)

The `SessionDB` maintains schema version 6 (`SCHEMA_VERSION = 6`). Migrations run automatically on startup, stepping through each version sequentially. Schema v6 was introduced in v0.5.0 ([#2974](https://github.com/NousResearch/hermes-agent/pull/2974)) to persist reasoning data across gateway session turns.

**`sessions` table:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PRIMARY KEY | Session UUID |
| `source` | TEXT NOT NULL | Platform origin: `cli`, `telegram`, `discord`, `slack`, etc. |
| `user_id` | TEXT | Platform user identifier |
| `model` | TEXT | Active model slug |
| `model_config` | TEXT | JSON serialized model configuration |
| `system_prompt` | TEXT | Snapshot of the assembled system prompt |
| `parent_session_id` | TEXT | Foreign key to parent session after compression split |
| `started_at` | REAL NOT NULL | Unix timestamp of session creation |
| `ended_at` | REAL | Unix timestamp when session ended |
| `end_reason` | TEXT | Why the session ended |
| `message_count` | INTEGER DEFAULT 0 | Total message count |
| `tool_call_count` | INTEGER DEFAULT 0 | Total tool call count |
| `input_tokens` | INTEGER DEFAULT 0 | Cumulative input tokens |
| `output_tokens` | INTEGER DEFAULT 0 | Cumulative output tokens |
| `cache_read_tokens` | INTEGER DEFAULT 0 | Cumulative cache-read tokens (v5) |
| `cache_write_tokens` | INTEGER DEFAULT 0 | Cumulative cache-write tokens (v5) |
| `reasoning_tokens` | INTEGER DEFAULT 0 | Cumulative reasoning tokens (v5) |
| `billing_provider` | TEXT | Provider used for billing (v5) |
| `billing_base_url` | TEXT | Endpoint used for billing (v5) |
| `billing_mode` | TEXT | Billing mode (v5) |
| `estimated_cost_usd` | REAL | Estimated cost in USD (v5) |
| `actual_cost_usd` | REAL | Actual cost in USD from provider (v5) |
| `cost_status` | TEXT | `actual`, `estimated`, `included`, `unknown` (v5) |
| `cost_source` | TEXT | Source of pricing data (v5) |
| `pricing_version` | TEXT | Pricing snapshot version identifier (v5) |
| `title` | TEXT | Human-readable session title (unique when non-NULL, max 100 chars) |

**`messages` table:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Row ID |
| `session_id` | TEXT NOT NULL | Foreign key to sessions |
| `role` | TEXT NOT NULL | `user`, `assistant`, `tool` |
| `content` | TEXT | Message content |
| `tool_call_id` | TEXT | Tool call identifier for tool results |
| `tool_calls` | TEXT | JSON serialized tool call list |
| `tool_name` | TEXT | Tool name for tool result messages |
| `timestamp` | REAL NOT NULL | Unix timestamp |
| `token_count` | INTEGER | Per-message token estimate |
| `finish_reason` | TEXT | Model finish reason (added in schema v2) |
| `reasoning` | TEXT | Raw reasoning text extracted from `<think>` blocks or native reasoning (schema v6) |
| `reasoning_details` | TEXT | JSON serialized structured reasoning detail objects (schema v6) |
| `codex_reasoning_items` | TEXT | JSON serialized Codex reasoning items for Responses API sessions (schema v6) |

**`messages_fts` virtual table:** FTS5 full-text search over message content, with triggers to auto-index on insert, update, and delete.

### Indexes

```sql
idx_sessions_source        ON sessions(source)
idx_sessions_parent        ON sessions(parent_session_id)
idx_sessions_started       ON sessions(started_at DESC)
idx_sessions_title_unique  ON sessions(title) WHERE title IS NOT NULL
idx_messages_session       ON messages(session_id, timestamp)
```

### Schema Migrations

| Version | Changes |
|---------|---------|
| v2 | Added `finish_reason` column to `messages` |
| v3 | Added `title` column to `sessions` |
| v4 | Added unique index on `title` (NULLs allowed) |
| v5 | Added `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`, billing columns (`billing_provider`, `billing_base_url`, `billing_mode`), and cost columns (`estimated_cost_usd`, `actual_cost_usd`, `cost_status`, `cost_source`, `pricing_version`) |
| v6 | Added `reasoning`, `reasoning_details`, `codex_reasoning_items` columns to `messages` to persist reasoning across gateway turns (v0.5.0, [#2974](https://github.com/NousResearch/hermes-agent/pull/2974)) |

### Thread Safety

`SessionDB` uses WAL (Write-Ahead Logging) mode and a `threading.Lock()` for all write operations. The connection is created with `check_same_thread=False` and `timeout=10.0`. This supports the common gateway pattern of multiple reader threads and a single writer.

### Session Titles and Lineage

- `MAX_TITLE_LENGTH = 100` characters
- `sanitize_title()` strips control characters, invisible Unicode, collapses whitespace, enforces max length
- `set_session_title()` validates uniqueness across sessions
- `resolve_session_by_title()` follows numbered lineage variants (e.g., `"my session"` -> `"my session #2"`) to find the most recent continuation
- `get_next_title_in_lineage()` generates the next title in a lineage (strips existing `#N` suffix, finds highest, increments)
- `resolve_session_id()` supports both exact ID and unique prefix matching
- Auto-titling via `agent/title_generator.py`: after the first user-assistant exchange, `maybe_auto_title()` fires a background thread that calls `call_llm(task="compression", max_tokens=30, temperature=0.3)` to generate a 3-7 word title

### FTS5 Search

`search_messages()` performs full-text search with FTS5 MATCH syntax. It sanitizes user query input via `_sanitize_fts5_query()` which:

- Preserves properly paired double-quoted phrases
- Strips unmatched FTS5-special characters (`+`, `{}`, `()`, `"`, `^`)
- Collapses repeated `*` and removes leading `*`
- Removes dangling boolean operators (`AND`, `OR`, `NOT`) at start/end
- Wraps unquoted hyphenated terms in double quotes (e.g., `chat-send` -> `"chat-send"`)

Results include a snippet (40 tokens, delimited by `>>>` and `<<<`), surrounding context (1 message before and after), and session metadata. Full content is removed from results to save tokens.

### Gateway vs CLI Persistence

- CLI uses `SessionDB` directly for resume, history, and search via `hermes sessions`
- Gateway maintains active-session mappings per platform and also writes to `SessionDB`
- Some legacy JSON/JSONL artifacts exist for compatibility, but SQLite is the canonical historical store

---

## Prompt Caching: How It Works

**File:** `agent/prompt_caching.py`

### Strategy: system_and_3

Anthropic prompt caching uses up to 4 `cache_control` breakpoints (the Anthropic maximum):

1. System prompt (stable across all turns)
2-4. Last 3 non-system messages (rolling window)

This reduces input token costs by approximately 75% on multi-turn conversations by caching the conversation prefix.

### When It Activates

`AIAgent.__init__` sets `self._use_prompt_caching = True` when:

- The base URL contains `openrouter` and the model name contains `claude`, OR
- `self.api_mode == "anthropic_messages"` (native Anthropic provider)

Default cache TTL is `"5m"` (5 minutes, 1.25x write cost). A `"1h"` TTL option is also supported.

### Implementation

`apply_anthropic_cache_control(api_messages, cache_ttl="5m")` is a pure function that:

1. Deep-copies the messages list
2. Sets `cache_control: {"type": "ephemeral"}` (or `{"type": "ephemeral", "ttl": "1h"}` for 1h TTL) on the system message
3. Sets the same marker on the last 3 non-system messages
4. For string content, wraps it in `[{"type": "text", "text": ..., "cache_control": ...}]`
5. For list content, adds `cache_control` to the last item
6. For tool messages, adds `cache_control` at the message level

---

## Trajectory Format: What Gets Logged

**Primary files:** `agent/trajectory.py`, `run_agent.py`, `batch_runner.py`, `trajectory_compressor.py`

### Purpose

Trajectory outputs are used for:

- SFT (supervised fine-tuning) data generation
- Debugging agent behavior
- Benchmark and evaluation artifact capture
- Post-processing and compression pipelines for training

### File Format

Trajectories are saved as JSONL files. Each line is a JSON object:

```json
{
  "conversations": [...],
  "timestamp": "2026-03-17T12:34:56.789012",
  "model": "anthropic/claude-opus-4.6",
  "completed": true
}
```

Successful trajectories go to `trajectory_samples.jsonl`, failed ones to `failed_trajectories.jsonl`.

### Normalization Strategy

Hermes converts live conversation structure into a training-friendly format:

- Reasoning is represented in explicit markup (think blocks converted via `convert_scratchpad_to_think()` which replaces `<REASONING_SCRATCHPAD>` tags with `<think>` tags)
- `has_incomplete_scratchpad()` detects opening tags without closing tags for cleanup
- Tool calls are converted into structured regions for dataset compatibility
- Successful and failed trajectories are separated

### Persistence Boundaries

Trajectory files do not blindly mirror all runtime prompt state. The `ephemeral_system_prompt` is intentionally excluded from persisted trajectory content so datasets are cleaner and less environment-specific.

---

## Auxiliary Client: What It Does

**File:** `agent/auxiliary_client.py`

The auxiliary client is a shared routing layer that handles all side-task LLM calls -- everything except the main agent loop. It provides a single resolution chain so every consumer picks up the best available backend without duplicating fallback logic.

### Consumers of the Auxiliary Client

| Consumer | Task Name | Purpose |
|---------|-----------|---------|
| `ContextCompressor._generate_summary()` | `compression` | Summarize middle turns |
| `title_generator.py` | `compression` | Generate session titles |
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
3. Custom endpoint (resolved via `resolve_runtime_provider(requested="custom")`)
4. Codex OAuth (Responses API via `chatgpt.com/backend-api/codex` with `gpt-5.2-codex`)
5. Native Anthropic (via `resolve_anthropic_token()`)
6. Direct API-key providers from `PROVIDER_REGISTRY`: z.ai/GLM, Kimi/Moonshot, MiniMax, MiniMax-CN, and others
7. None (raises `RuntimeError`)

### Provider Resolution Order (Vision Tasks, Auto Mode)

1. Selected main provider (if it is a supported vision backend)
2. OpenRouter
3. Nous Portal
4. Codex OAuth (`gpt-5.2-codex` supports vision via Responses API)
5. Native Anthropic
6. Custom endpoint (local vision models: Qwen-VL, LLaVA, Pixtral, etc.)
7. None

### Default Auxiliary Models Per Provider

| Provider | Default Auxiliary Model |
|----------|----------------------|
| OpenRouter | `google/gemini-3-flash-preview` |
| Nous Portal | `gemini-3-flash` |
| Anthropic | `claude-haiku-4-5-20251001` |
| z.ai | `glm-4.5-flash` |
| Kimi/Moonshot | `kimi-k2-turbo-preview` |
| MiniMax | `MiniMax-M2.7-highspeed` |
| AI Gateway | `google/gemini-3-flash` |
| Codex | `gpt-5.2-codex` |

### Per-Task Override Configuration

Each auxiliary task supports per-task provider and model overrides via:

- Environment variables: `AUXILIARY_{TASK}_PROVIDER`, `AUXILIARY_{TASK}_MODEL`, `AUXILIARY_{TASK}_BASE_URL`, `AUXILIARY_{TASK}_API_KEY`
- Config file: `auxiliary.{task}.provider`, `auxiliary.{task}.model`, `auxiliary.{task}.base_url`
- Legacy: `compression.summary_provider`, `compression.summary_model`, `compression.summary_base_url`

Resolution priority: explicit args > env vars > config file > auto-detection.

### Codex Adapter

The `_CodexCompletionsAdapter` class provides a `chat.completions.create()`-compatible interface that internally translates to the Codex Responses API. This makes Codex transparent to all auxiliary consumers. The adapter converts content formats (`text`/`image_url` -> `input_text`/`input_image`), handles tools, and streams the response to reconstruct a chat-completions-compatible result. `CodexAuxiliaryClient` wraps this for sync use; `AsyncCodexAuxiliaryClient` wraps it for async consumers.

### Anthropic Adapter

`AnthropicAuxiliaryClient` wraps the native Anthropic client in a `chat.completions.create()`-compatible interface via `_AnthropicCompletionsAdapter`. It uses `build_anthropic_kwargs()` and `normalize_anthropic_response()` from `agent/anthropic_adapter.py`. Auth supports regular API keys (`sk-ant-api*`), OAuth setup-tokens (`sk-ant-oat*`), and Claude Code credentials (`~/.claude.json` or `~/.claude/.credentials.json`).

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
- Message contains backticks or code fences
- Message contains a URL (`https://` or `www.`)
- Message contains any word from `_COMPLEX_KEYWORDS`: `debug`, `debugging`, `implement`, `implementation`, `refactor`, `patch`, `traceback`, `stacktrace`, `exception`, `error`, `analyze`, `analysis`, `investigate`, `architecture`, `design`, `compare`, `benchmark`, `optimize`, `optimise`, `review`, `terminal`, `shell`, `tool`, `tools`, `pytest`, `test`, `tests`, `plan`, `planning`, `delegate`, `subagent`, `cron`, `docker`, `kubernetes`

### Route Resolution

When cheap routing activates, `resolve_turn_route()` calls `resolve_runtime_provider()` with the cheap provider's configuration and returns a route dict with `model`, `runtime`, `label`, and `signature` fields. The label field is set to `"smart route -> {model} ({provider})"` for display purposes.

---

## Timezone-Aware Clock

**File:** `hermes_time.py`

Provides `now()` which returns a timezone-aware `datetime`. Resolution order:

1. `HERMES_TIMEZONE` environment variable (highest priority)
2. `timezone` key in `~/.hermes/config.yaml`
3. Server's local time via `datetime.now().astimezone()`

Uses `zoneinfo.ZoneInfo` for IANA timezone names. Invalid timezone values log a warning and fall back safely. The timezone is resolved once and cached; call `reset_cache()` after config changes.

---

## Usage Pricing and Insights

**Files:** `agent/usage_pricing.py`, `agent/insights.py`

### Pricing

`CanonicalUsage` is a frozen dataclass holding: `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`, `request_count`. Cost status is tracked via `CostStatus` (`"actual"`, `"estimated"`, `"included"`, `"unknown"`) and `CostSource` (`"provider_cost_api"`, `"provider_generation_api"`, `"provider_models_api"`, `"official_docs_snapshot"`, etc.).

### Insights Engine

`InsightsEngine` analyzes historical session data from the SQLite state database to produce usage insights: token consumption, cost estimates, tool usage patterns, activity trends, model/platform breakdowns, and session metrics.

Usage: `InsightsEngine(db).generate(days=30)` returns a report dict; `format_terminal(report)` renders it for the CLI.

---

## Architectural Changes in v0.4.0 and v0.5.0

### Dependency Removals

**mini-swe-agent removed (v0.5.0, [#2804](https://github.com/NousResearch/hermes-agent/pull/2804)):** The `mini-swe-agent` submodule and its accompanying Docker and Modal backends were inlined directly into `tools/environments/docker.py` and `tools/environments/modal.py`. The submodule install step (`uv pip install -e "./mini-swe-agent"`) is no longer required.

**swe-rex replaced with native Modal SDK (v0.5.0, [#3538](https://github.com/NousResearch/hermes-agent/pull/3538)):** The Modal terminal backend previously delegated container operations through the `swe-rex` library, which required tunnel management and additional process coordination. The backend now calls the Modal SDK directly: `Sandbox.create.aio` for container creation and `exec.aio` for command execution. This removes the tunnel layer entirely and simplifies the `modal` extra in `pyproject.toml` to just `modal>=1.0.0,<2`.

### Plugin Lifecycle Hooks (v0.5.0, [#3542](https://github.com/NousResearch/hermes-agent/pull/3542))

The plugin hook system that was wired up in v0.4.0 ([#2333](https://github.com/NousResearch/hermes-agent/pull/2333)) at the TUI layer now fires through the full agent loop and both CLI and gateway entry paths. Four hooks are active:

| Hook | Fires when |
|------|-----------|
| `pre_llm_call` | Immediately before each API call to the LLM |
| `post_llm_call` | Immediately after the LLM response is received |
| `on_session_start` | When a new session is started (CLI start, gateway turn start) |
| `on_session_end` | When a session ends (clean exit, reset, or timeout) |

Plugins register hooks via `hermes plugins install`. Plugin toolset visibility in `hermes tools` and standalone processes was also fixed in v0.5.0 ([#3457](https://github.com/NousResearch/hermes-agent/pull/3457)): toolsets registered by plugins are now included in the toolset discovery pass that runs at startup.

### Always-Streaming Approach (v0.5.0, [#3120](https://github.com/NousResearch/hermes-agent/pull/3120))

The agent loop now always prefers streaming API calls. Previously, API calls could be issued non-streaming when no stream consumers were attached, which caused subagents to hang waiting for large responses. The new policy: if the provider supports streaming, use it; fall back to non-streaming only if the stream itself fails ([#3020](https://github.com/NousResearch/hermes-agent/pull/3020)). This eliminated a class of hung-subagent bugs in gateway mode.

### Canonical Config Path: `get_hermes_home()` (v0.5.0, [#3062](https://github.com/NousResearch/hermes-agent/pull/3062))

`get_hermes_home()` in `hermes_constants.py` is now the single source of truth for the Hermes home directory. All previously scattered inline calls to `Path.home() / ".hermes"` or `os.environ.get("HERMES_HOME", ...)` were consolidated to import and call this function. The function reads the `HERMES_HOME` environment variable and falls back to `~/.hermes`:

```python
def get_hermes_home() -> Path:
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))
```

A companion `get_hermes_dir(new_subpath, old_name)` handles backward-compatible subdirectory resolution: new installs use the consolidated layout (e.g., `cache/images`), existing installs that already have the legacy path (e.g., `image_cache`) continue to use it without migration.

### Streaming Schema v6 and Reasoning Persistence (v0.5.0, [#2974](https://github.com/NousResearch/hermes-agent/pull/2974))

Prior to v0.5.0, reasoning content extracted from `<think>` blocks or native reasoning fields was only available in the current turn's streaming callbacks and was discarded before the message was persisted. Schema v6 adds three columns to the `messages` table (`reasoning`, `reasoning_details`, `codex_reasoning_items`) so that reasoning is stored alongside assistant messages. The gateway's session restore path reads these columns back, allowing multi-turn gateway sessions to retain and replay reasoning context across connection resets.

### Plugin Toolset Discovery Ordering (v0.5.0, [#3457](https://github.com/NousResearch/hermes-agent/pull/3457))

`model_tools.py` now runs a plugin discovery pass before building the final toolset list. Previously, toolsets contributed by installed plugins were invisible to `hermes tools` (the tool selection TUI) and to standalone processes that did not go through the full CLI startup path. The fix moves plugin toolset registration to an earlier phase so all toolsets -- built-in and plugin-provided -- are present when the toolset selector and any external process queries the registry.

---

## Design Themes

Several cross-cutting design themes appear throughout the codebase:

- **Prompt stability matters** -- the system prompt is assembled once per session and the stable prefix must remain stable for provider-side prompt caching to be effective
- **Tool execution must be observable and interruptible** -- every tool call fires callbacks; long LLM calls can be cancelled
- **Session persistence must survive long-running use** -- every message and token count is written to SQLite immediately; WAL mode allows concurrent readers
- **Platform frontends should share one agent core** -- `AIAgent` is identical whether called from CLI, gateway, ACP, or cron
- **Optional subsystems should remain loosely coupled** -- Honcho, MCP, and ACP are injected via callbacks and configuration, not hardwired into the agent loop
- **Graceful degradation everywhere** -- `_SafeWriter` wraps stdout/stderr to prevent crashes from broken pipes; streaming falls back to non-streaming on error; auxiliary models fall back through the provider chain

---

## Source Files Reference

| File | Description |
|------|-------------|
| `run_agent.py` | `AIAgent` -- core orchestration loop (entry point for all platforms) |
| `cli.py` | Interactive terminal UI (prompt_toolkit TUI) |
| `model_tools.py` | Tool discovery, schema building, and dispatch |
| `toolsets.py` | Tool groupings and platform presets |
| `hermes_state.py` | `SessionDB` -- SQLite session and message store |
| `hermes_constants.py` | Shared API endpoint constants (`OPENROUTER_BASE_URL`, `AI_GATEWAY_BASE_URL`, `NOUS_API_BASE_URL`) |
| `hermes_time.py` | `now()` -- timezone-aware clock |
| `batch_runner.py` | Batch trajectory generation for RL/SFT |
| `trajectory_compressor.py` | `TrajectoryCompressor` -- offline training data compression |
| `agent/context_compressor.py` | `ContextCompressor` -- runtime context management |
| `agent/prompt_builder.py` | System prompt assembly functions |
| `agent/prompt_caching.py` | `apply_anthropic_cache_control()` -- Anthropic cache_control markers |
| `agent/auxiliary_client.py` | `call_llm()`, `async_call_llm()`, `resolve_provider_client()` -- centralized LLM router |
| `agent/model_metadata.py` | Context length resolution, token estimation, pricing extraction |
| `agent/smart_model_routing.py` | `choose_cheap_model_route()` -- optional cheap model routing |
| `agent/usage_pricing.py` | `estimate_usage_cost()`, `normalize_usage()`, `CanonicalUsage`, `BillingRoute`, `PricingEntry` |
| `agent/insights.py` | `InsightsEngine` -- session analytics and cost estimation |
| `agent/anthropic_adapter.py` | Native Anthropic API translation, OAuth, credential discovery |
| `agent/trajectory.py` | `save_trajectory()`, `convert_scratchpad_to_think()` |
| `agent/title_generator.py` | `maybe_auto_title()` -- auto-generate session titles |
| `agent/redact.py` | `redact_sensitive_text()`, `RedactingFormatter` -- secret masking |
| `agent/display.py` | `KawaiiSpinner`, tool preview/emoji helpers |
| `gateway/` | Platform adapters (Telegram, Discord, Slack, WhatsApp, Signal, Email, Home Assistant) |
| `cron/` | Scheduled job storage and scheduler |
| `honcho_integration/` | Honcho memory integration |
| `acp_adapter/` | ACP editor integration server |
| `environments/` | RL/benchmark environment framework |
| `skills/` | Bundled skill documents |
| `optional-skills/` | Official optional skills |
| `tools/` | Tool implementations and terminal environments |
