# Context Compression and Caching

Hermes Agent uses a dual compression system and Anthropic prompt caching to manage context window usage efficiently across long conversations.

Source files: `agent/context_compressor.py`, `agent/prompt_caching.py`, `gateway/run.py`, `run_agent.py`

## Dual Compression System

Hermes has two separate compression layers that operate independently:

```
                     +----------------------------+
  Incoming message   |  Gateway Session Hygiene   |  Fires at 85% of context
  =================> |  (pre-agent, rough est.)   |  Safety net for large sessions
                     +-------------+--------------+
                                   |
                                   v
                     +----------------------------+
                     |  Agent ContextCompressor   |  Fires at 50% of context (default)
                     |  (in-loop, real tokens)    |  Normal context management
                     +----------------------------+
```

### Gateway Session Hygiene (85% threshold)

Located in `gateway/run.py`. This is a safety net that runs before the agent processes a message. It prevents API failures when sessions grow too large between turns (e.g. overnight accumulation in Telegram/Discord).

- **Threshold**: Fixed at 85% of model context length
- **Token source**: Prefers actual API-reported tokens from last turn; falls back to rough character-based estimate
- **Fires**: Only when `len(history) >= 4` and compression is enabled

The gateway hygiene threshold is intentionally higher than the agent's compressor. Setting it at 50% (same as the agent) caused premature compression on every turn in long gateway sessions.

### Agent ContextCompressor (50% threshold, configurable)

Located in `agent/context_compressor.py`. This is the primary compression system that runs inside the agent's tool loop with access to accurate, API-reported token counts.

## Configuration

All compression settings are read from `config.yaml` under the `compression` key:

```yaml
compression:
  enabled: true              # Enable/disable compression (default: true)
  threshold: 0.50            # Fraction of context window (default: 0.50 = 50%)
  target_ratio: 0.20         # How much of threshold to keep as tail (default: 0.20)
  protect_last_n: 20         # Minimum protected tail messages (default: 20)
  summary_model: null        # Override model for summaries (default: uses auxiliary)
```

### Constructor Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `model` | required | Model name for context length lookup |
| `threshold_percent` | `0.50` | Fraction of context window that triggers compression |
| `protect_first_n` | `10` | Number of head messages to preserve |
| `protect_last_n` | `20` | Minimum number of recent tail messages to preserve |
| `summary_target_ratio` | `0.20` | Fraction of the threshold budget preserved as the recent tail target |
| `summary_model_override` | None | Force a specific model for summarization |

### Computed Values (for a 200K context model at defaults)

```
context_length       = 200,000
threshold_tokens     = 200,000 x 0.50 = 100,000
tail_token_budget    = 100,000 x 0.20 = 20,000
max_summary_tokens   = min(200,000 x 0.05, 12,000) = 10,000
```

## Compression Algorithm

The `ContextCompressor.compress()` method follows a 4-phase algorithm:

### Phase 1: Prune Old Tool Results (cheap, no LLM call)

Old tool results (>200 chars) outside the protected tail are replaced with:

```
[Old tool output cleared to save context space]
```

This is a cheap pre-pass that saves significant tokens from verbose tool outputs (file contents, terminal output, search results).

### Phase 2: Determine Boundaries

```
+-------------------------------------------------------------+
|  Message list                                               |
|                                                             |
|  [0..9]  <- protect_first_n (system + first exchanges)      |
|  [3..N]  <- middle turns -> SUMMARIZED                      |
|  [N..end] <- tail (by token budget OR protect_last_n)       |
+-------------------------------------------------------------+
```

Tail protection is token-budget based: walks backward from the end, accumulating tokens until the budget is exhausted. Falls back to the fixed `protect_last_n` count if the budget would protect fewer messages.

Boundaries are aligned to avoid splitting tool_call/tool_result groups. The `_align_boundary_backward()` method walks past consecutive tool results to find the parent assistant message, keeping groups intact.

### Phase 3: Generate Structured Summary

The middle turns are summarized using the auxiliary LLM with a structured template:

```
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
[User preferences, coding style, constraints, important decisions]

## Progress
### Done
[Completed work -- specific file paths, commands run, results]
### In Progress
[Work currently underway]
### Blocked
[Any blockers or issues encountered]

## Key Decisions
[Important technical decisions and why]

## Relevant Files
[Files read, modified, or created -- with brief note on each]

## Next Steps
[What needs to happen next]

## Critical Context
[Specific values, error messages, configuration details]
```

Summary budget scales with the amount of content being compressed:
- Formula: `content_tokens x 0.20` (the `_SUMMARY_RATIO` constant)
- Minimum: 2,000 tokens
- Maximum: `min(context_length x 0.05, 12,000)` tokens

### Phase 4: Assemble Compressed Messages

The compressed message list is:
1. Head messages (with a note appended to system prompt on first compression)
2. Summary message (role chosen to avoid consecutive same-role violations)
3. Tail messages (unmodified)

Orphaned tool_call/tool_result pairs are cleaned up by `_sanitize_tool_pairs()`:
- Tool results referencing removed calls are removed
- Tool calls whose results were removed get stub results injected

### Iterative Re-compression

On subsequent compressions, the previous summary is passed to the LLM with instructions to update it rather than summarize from scratch. This preserves information across multiple compactions -- items move from "In Progress" to "Done", new progress is added, and obsolete information is removed.

The `_previous_summary` field on the compressor instance stores the last summary text for this purpose.

### Multi-Pass Compression

For very large sessions, a single compression pass may not reduce the context enough. The compressor may iterate multiple times: after the first compression, if the resulting message list still exceeds the threshold, `compress()` is called again. Each pass summarizes the new middle region (which now includes the previous summary message) and produces a smaller message list. This continues until the context fits within the threshold or no further compression is possible (i.e., the message count is at or below `protect_first_n + protect_last_n + 1`).

## Compression Activation Condition

Compression activates when `len(messages) > protect_first_n + protect_last_n + 1`. With current defaults (`protect_first_n = 10`, `protect_last_n = 20`), this means compression only fires when there are more than 31 messages. Below that threshold, there is no middle region to summarize.

## Summary Generation

`_generate_summary()` truncates each turn to 2000 characters (1000 head + 500 tail), appends tool call names, and sends the formatted content to `call_llm(task="compression", max_tokens=summary_target_tokens * 2)`. If the auxiliary model fails, it falls back to the user's main model. If all models fail, the middle turns are dropped without a summary.

### Summary Failure Cooldown

When summary generation fails (no provider available or an exception is raised), the compressor sets a cooldown of 10 minutes (`_SUMMARY_FAILURE_COOLDOWN_SECONDS = 600`). During the cooldown period, `_generate_summary()` returns `None` immediately without attempting another API call, and the compressor falls back to dropping middle turns without a summary. This prevents repeated failed summary attempts from adding latency to every compression pass after a provider outage.

### Summary Role Selection

The summary message role is chosen to avoid consecutive same-role messages with both the head neighbor and the tail neighbor. If neither `"user"` nor `"assistant"` avoids a collision, the summary is merged into the first tail message instead of being inserted as a standalone message.

### Tool-Pair Sanitization

After compression, `_sanitize_tool_pairs()` handles two failure modes that would cause API rejections:

1. **Orphaned tool results** -- a tool result references a call_id whose assistant message was compressed away. These results are removed.
2. **Calls without results** -- an assistant message has `tool_calls` whose result messages were compressed away. Stub result messages (`"[Result from earlier conversation -- see context summary above]"`) are inserted.

## Session Lineage After Compression

Compression can split the session into a new `session_id` while preserving ancestry via `parent_session_id` in the `SessionDB`. This allows the agent to continue with a smaller active context while retaining a searchable ancestry chain for session history and the `session_search` tool.

## Prompt Caching (Anthropic)

Source: `agent/prompt_caching.py`

Reduces input token costs by approximately 75% on multi-turn conversations by caching the conversation prefix. Uses Anthropic's `cache_control` breakpoints.

### Strategy: system_and_3

Anthropic allows a maximum of 4 `cache_control` breakpoints per request. Hermes uses the "system_and_3" strategy:

```
Breakpoint 1: System prompt           (stable across all turns)
Breakpoint 2: 3rd-to-last non-system message
Breakpoint 3: 2nd-to-last non-system message   (rolling window)
Breakpoint 4: Last non-system message
```

### How It Works

`apply_anthropic_cache_control()` deep-copies the messages and injects `cache_control` markers:

```python
# Cache marker format
marker = {"type": "ephemeral"}
# Or for 1-hour TTL:
marker = {"type": "ephemeral", "ttl": "1h"}
```

The marker is applied differently based on content type:

| Content Type | Where Marker Goes |
|-------------|-------------------|
| String content | Converted to `[{"type": "text", "text": ..., "cache_control": ...}]` |
| List content | Added to the last element's dict |
| None/empty | Added as `msg["cache_control"]` |
| Tool messages | Added as `msg["cache_control"]` (native Anthropic only) |

### When It Activates

`AIAgent.__init__` sets `self._use_prompt_caching = True` when:

- The base URL contains `openrouter` and the model name contains `claude`, OR
- `self.api_mode == "anthropic_messages"` (native Anthropic provider)

Default cache TTL is `"5m"` (5 minutes, 1.25x write cost). A `"1h"` TTL option is also supported.

### Cache-Aware Design Patterns

1. **Stable system prompt**: The system prompt is breakpoint 1 and cached across all turns. Avoid mutating it mid-conversation.
2. **Message ordering matters**: Cache hits require prefix matching. Adding or removing messages in the middle invalidates the cache for everything after.
3. **Compression cache interaction**: After compression, the cache is invalidated for the compressed region but the system prompt cache survives. The rolling 3-message window re-establishes caching within 1-2 turns.
4. **TTL selection**: Default is `5m` (5 minutes). Use `1h` for long-running sessions where the user takes breaks between turns.

## Trajectory Compressor (Separate Tool)

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
| Summary target | 2,500 tokens | 750 tokens |
| Protected turns | First 10 + last 20 | First system/human/gpt/tool + last 4 |
