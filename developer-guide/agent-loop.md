# Agent Loop Internals

The core orchestration engine is `AIAgent` in `run_agent.py`. It is the single class used by all entry points -- CLI, gateway, ACP, cron, and batch runner.

## Core Responsibilities

`AIAgent` is responsible for:

- Assembling the effective prompt and tool schemas
- Selecting the correct provider and API mode
- Making interruptible model calls (streaming or non-streaming)
- Executing tool calls (sequentially or concurrently)
- Maintaining session history in memory and in the `SessionDB`
- Handling compression, retries, and fallback models
- Firing platform-specific callbacks during execution

## AIAgent Constructor Parameters (Selected)

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

## Turn Lifecycle

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

## API Modes

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

## Interruptible API Calls

Hermes wraps API requests so they can be interrupted from the CLI or gateway. This handles:

- Long LLM calls when the user sends a new message mid-flight
- Gateway cancellation semantics when a session resets
- Background system cancellation during shutdown

## Tool Execution Modes

Two strategies are used:

- **Sequential execution** -- for single tool calls or interactive tools (e.g. `clarify`)
- **Concurrent execution** -- for multiple non-interactive tools via `ThreadPoolExecutor` (max `_MAX_TOOL_WORKERS = 8` workers)

Concurrent execution is decided by `_should_parallelize_tool_batch()`, which checks:

- Batch has more than 1 tool call
- No tool is in `_NEVER_PARALLEL_TOOLS` (currently: `{"clarify"}`)
- Every tool is either in `_PARALLEL_SAFE_TOOLS` (read-only tools like `read_file`, `search_files`, `session_search`, `web_search`, etc.) or is a `_PATH_SCOPED_TOOLS` tool (`read_file`, `write_file`, `patch`) targeting non-overlapping paths
- All tool call arguments parse as valid JSON dicts

Message and result ordering is preserved when reinserting tool responses into conversation history.

## Callback Surfaces

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

## Budget and Fallback Behavior

Hermes tracks a shared iteration budget across parent agents and all subagents it spawns via the `IterationBudget` class. The budget is thread-safe (`threading.Lock`), supports `consume()` (returns False when exhausted), and `refund()` (for `execute_code` turns that should not count).

Budget pressure hints are injected into tool results at two thresholds:
- **70% consumed** (`_budget_caution_threshold`) -- nudge to start wrapping up
- **90% consumed** (`_budget_warning_threshold`) -- urgent, respond now

Fallback model support allows the agent to switch providers or models when the primary route fails.

## API Retry: Jittered Backoff (v0.8.0)

**New in v0.8.0** (PR [#6048](https://github.com/NousResearch/hermes-agent/pull/6048)): API retries use jittered exponential backoff instead of fixed delays. The implementation lives in `agent/retry_utils.py`.

```python
jittered_backoff(attempt, base_delay=2.0, max_delay=60.0)
```

- Formula: `min(base * 2^(attempt-1), max_delay) + uniform_jitter`
- Jitter range: `[0, 0.5 * computed_delay]` by default
- The jitter seed mixes `time.time_ns()` with a monotonic counter so concurrent sessions (multiple gateway turns) produce decorrelated delays, preventing thundering-herd retry spikes against a rate-limited provider
- Rate-limit (429) retries use `base_delay=2.0, max_delay=60.0`; extended backoff paths use `base_delay=5.0, max_delay=120.0`

## Smart Thinking Block Signature Management (v0.8.0)

**New in v0.8.0** (PR [#6112](https://github.com/NousResearch/hermes-agent/pull/6112)): Anthropic signs thinking blocks against the full turn content. Any upstream mutation -- context compression, session truncation, orphan stripping, message merging -- invalidates the signature and causes HTTP 400 errors.

The Anthropic adapter (`agent/anthropic_adapter.py`) applies a three-rule strategy when building API messages:

1. **Strip thinking/redacted_thinking from all assistant messages except the last one.** Only the current tool-use chain needs live thinking signatures. All prior turns have their thinking blocks removed.
2. **Downgrade unsigned thinking blocks to plain text.** If a thinking block has no `signature` field, it cannot be validated by Anthropic and will be rejected -- it is converted to a `{"type": "text", "text": ...}` block instead to preserve the reasoning content.
3. **Strip `cache_control` from all thinking/redacted_thinking blocks** in every message -- cache markers interfere with signature validation.

## Accepting Reasoning-Only Responses (v0.8.0)

**New in v0.8.0** (PR [#5278](https://github.com/NousResearch/hermes-agent/pull/5278)): When a model returns structured reasoning (via API fields) but no visible text content, the agent no longer retries indefinitely. Instead:

1. **Thinking-only prefill continuation**: if the response has `reasoning`, `reasoning_content`, or `reasoning_details` fields but no text, the agent appends the assistant message as-is (marked with `_thinking_prefill = True`) and continues to the next API call. The model sees its own reasoning on the next turn and typically produces the text portion. This is retried up to 2 times (`_thinking_prefill_retries < 2`).

2. **Fallback to `"(empty)"`**: if prefill retries are exhausted or there is no structured reasoning at all, the assistant message content is set to the string `"(empty)"` and the turn ends. This prevents an infinite retry loop that burns through the iteration budget.

## System Prompt Caching Strategy

The system prompt is built once on the first `run_conversation()` call and reused for all subsequent turns in the session. This is critical for Anthropic prefix cache hits -- a stable system prompt prefix means the provider can serve cached tokens instead of reprocessing.

Key behaviors:

- **First turn**: `_build_system_prompt()` assembles all 10 layers (identity through platform hints) and stores the result in `self._cached_system_prompt`
- **Subsequent turns**: the cached prompt is reused without reassembly
- **After context compression**: the cache is invalidated and the prompt is rebuilt, because compression may alter the system message (e.g. appending the compaction note)
- **Continuing gateway sessions**: when a gateway session is resumed, the stored system prompt is loaded from `SessionDB` (via the `system_prompt` column) rather than rebuilt from scratch. This preserves the exact prompt prefix that Anthropic has already cached, avoiding cache misses from non-deterministic elements like timestamps or changed context files

## Honcho Prefetch Mechanics

When Honcho integration is active, context is injected at two different points depending on the turn:

- **First turn**: Honcho context is baked into the cached system prompt as a static block (layer 3). This content remains stable for the entire session, preserving cache effectiveness
- **Later turns**: turn-specific context from Honcho is attached at API call time only via `_inject_honcho_turn_context()`. This content is injected into the current-turn user message and is NOT persisted into the cached system prompt

This split ensures that turn N can consume the Honcho prefetch results from turn N-1 without destabilizing the cached system prompt prefix. The static block captures the user's cross-session profile, while the per-turn injection captures recall results specific to the current conversation flow.

## Iteration Budget Semantics

The default `max_iterations` is 90. The `IterationBudget` class manages how many tool-calling iterations the agent can perform.

Key semantics:

- **Independent subagent budgets**: subagents spawned via `delegate_task` receive their own independent `IterationBudget`. They do NOT count against the parent agent's remaining budget
- **Thread safety**: `IterationBudget` uses a `threading.Lock` internally. `consume()` returns `False` when the budget is exhausted. `refund()` gives back iterations (used for `execute_code` turns that should not count)
- **Budget pressure warnings**: hints are injected into tool results at two thresholds:
  - **70% consumed** (`_budget_caution_threshold`): "You have used most of your iteration budget. Start wrapping up."
  - **90% consumed** (`_budget_warning_threshold`): "You are almost out of iterations. Respond to the user now."
- **Budget exhaustion**: when `consume()` returns `False`, the agent loop stops making tool calls and returns whatever response it has

## Compression and Persistence

Before and during long runs, Hermes may:

- Flush memory before context loss
- Compress middle conversation turns via `ContextCompressor`
- Split the session lineage into a new session ID after compression
- Preserve recent context and structural tool-call/result consistency

## Source Files

| File | Description |
|------|-------------|
| `run_agent.py` | `AIAgent` -- core orchestration loop |
| `agent/prompt_builder.py` | System prompt assembly |
| `agent/context_compressor.py` | Runtime context compression |
| `agent/prompt_caching.py` | Anthropic cache_control markers |
| `model_tools.py` | Tool discovery, schema building, and dispatch |
