# Unified Streaming

Hermes v0.3.0 (v2026.3.17) introduced a unified streaming infrastructure that delivers LLM responses token-by-token across all platforms. Before this release, all responses arrived as a single block after the model finished generating. With streaming, text appears progressively as it is produced.

Streaming is implemented via PR [#1538](https://github.com/NousResearch/hermes-agent/pull/1538).

**v0.4.0 changes (PR #2340, #2161, #2118, #2159):** CLI streaming mode added. Spinner and tool progress display now work correctly during streaming turns. Reasoning/thinking blocks display alongside streamed text when `show_reasoning` is enabled. Context pressure warnings appear in both CLI and gateway.

**v0.5.0 changes (PR #3120, #3020, #2974):** Always-prefer-streaming mode prevents hung subagents. Non-streaming fallback is restored after stream failures. Reasoning fields are persisted across gateway turns with schema v6 columns.

---

## What Unified Streaming Is

Unified streaming means the same streaming mechanism in `AIAgent` (`run_agent.py`) works identically regardless of whether the consumer is the CLI, a messaging gateway platform, or an API server. The agent does not know or care what the consumer does with each token -- it emits tokens through callbacks, and each platform handles display in the appropriate way.

Before v0.3.0:

- The agent waited for the LLM to complete the full response
- The entire response was delivered to the CLI or messaging platform at once
- Users saw a spinner, then a complete response block
- No incremental feedback during generation

After v0.3.0:

- The agent opens a streaming connection to the LLM API
- Each text token is delivered via callbacks as it arrives
- The CLI prints tokens to the terminal progressively
- Gateway platforms (Telegram, Discord, Slack) send an initial message then edit it with accumulated content at regular intervals
- A `None` signal marks end of stream so consumers can finalize display

---

## Architecture

### Callback Registration

`AIAgent` has two independent stream callback paths:

1. **`stream_delta_callback`** -- set in `AIAgent.__init__()`. This is a persistent callback that fires for every text token across all turns. Used by the CLI for progressive display.

2. **`stream_callback`** -- set per-call via `run_conversation(stream_callback=...)`. Stored internally as `self._stream_callback`. Used by the TTS pipeline to start audio generation before the full response.

Both callbacks are fired by `_fire_stream_delta(text)` for each text token. The method `_has_stream_consumers()` returns `True` if either callback is registered.

### Thread-Safe Bridge

The streaming bridge between the agent thread and the consumer uses a thread-safe `queue.Queue()`:

```
                          _fire_stream_delta(delta)
                                |
  +---------------+    +-------v------------------+
  |  LLM API      |    |      queue.Queue()        |
  |  (stream)     |--->|  thread-safe bridge       |
  |               |    |  agent thread <-> consumer|
  +---------------+    +-------+-------------------+
                               |
                 +-------------+-------------+
                 |             |             |
           +-----v-----+ +----v------+ +----v------+
           |    CLI     | |  Gateway  | | API Server|
           | print to   | | edit msg  | | SSE event |
           | terminal   | | on Tg/Dc  | | to client |
           +-----------+ +-----------+ +-----------+
```

---

## How Token Delivery Works

### The Streaming Entry Point

When `_has_stream_consumers()` returns `True`, the agent loop calls `_interruptible_streaming_api_call()` instead of `_interruptible_api_call()`. This method handles all three API modes.

### API Modes

**Chat Completions API** (`chat_completions` mode)

Inside `_interruptible_streaming_api_call()`, the `_call_chat_completions()` inner function:

1. Adds `stream=True` and `stream_options={"include_usage": True}` to the API call
2. Iterates chunks from the stream
3. For text content (`delta.content`): appends to accumulated buffer and calls `_fire_stream_delta(delta.content)`
4. For reasoning content: calls `_fire_reasoning_delta()`
5. For tool call deltas: accumulates silently into a dict indexed by `tc_delta.index`
6. At stream end, constructs a `SimpleNamespace` response object compatible with the non-streaming code path
7. On any exception, falls back to `_interruptible_api_call()` (non-streaming)

**Codex Responses API** (`codex_responses` mode)

For Codex, `_interruptible_streaming_api_call()` delegates to `_interruptible_api_call()`, which internally calls `_run_codex_stream()`. The `_codex_on_first_delta` callback is temporarily stored on the instance so the Codex streaming path can fire it.

**Anthropic Messages API** (`anthropic_messages` mode)

Native Anthropic streaming is handled within `_interruptible_streaming_api_call()` using the Anthropic SDK's `client.messages.stream()` context manager. Text deltas (`content_block_delta` events of type `text_delta`) are delivered via `_fire_stream_delta()`. Thinking deltas fire `_fire_reasoning_delta()`.

### End-of-Stream Signal

After the streaming API call returns, the agent sends `_fire_stream_delta(None)` (a `None` token). This is the end-of-stream signal. Consumers use it to:

- Remove the typing cursor from the terminal display
- Remove the cursor character from the last gateway message edit
- Close SSE connections in the API server

### Tool Calls During Streaming

When the model returns tool calls instead of text, no text tokens are emitted -- `_fire_stream_delta` is never called with text content in that turn. Tool progress messages continue to appear through the existing `tool_progress_callback` mechanism. After tools execute, if the model produces a final text response, streaming picks up again.

---

## Platform Support

| Platform | Streaming support | Delivery method |
|----------|------------------|-----------------|
| CLI | Full | Tokens printed to terminal progressively |
| Telegram | Full | Message sent, then edited every ~1.5s |
| Discord | Full | Message sent, then edited every ~1.2s |
| Slack | Full | Message sent, then updated every ~1.5s |
| WhatsApp | No edit API | Falls back to non-streaming automatically |
| Signal | No edit API | Falls back to non-streaming automatically |
| Email | Not applicable | Falls back to non-streaming |
| Home Assistant | No edit API | Falls back to non-streaming automatically |

### Gateway Rate Limits

Platform message-edit APIs have rate limits. The streaming preview task respects these by throttling edits:

| Platform | Rate limit | Default edit interval |
|----------|------------|----------------------|
| Telegram | ~20 edits/min | 1.5s |
| Discord | 5 edits per 5s per message | 1.2s |
| Slack | ~50 API calls/min | 1.5s |

A `min_tokens` threshold (default: 20) prevents displaying a partial message before enough context exists for the preview to be meaningful.

If an edit receives a 429 rate-limit response, that edit cycle is skipped and the next interval attempt continues normally.

### Duplicate Message Prevention

When streaming is active and tokens were delivered via progressive edits, the normal final-response `send()` call is suppressed to prevent duplicate messages.

At the platform boundary this is implemented by a runtime `already_sent` marker:

- `GatewayStreamConsumer.already_sent` in `gateway/stream_consumer.py` is set to `True` after first stream send/edit succeeds.
- `gateway/run.py` copies that into the agent result (`response["already_sent"] = True`).
- `gateway/run.py` reads `already_sent` to skip the adapter's standard final-send path.

The final edit uses the post-processed response (with `<think>...</think>` blocks stripped and trailing whitespace cleaned) rather than the raw accumulated stream text.

---

## CLI Streaming

In the interactive CLI (`cli.py`), streaming tokens are delivered through `prompt_toolkit`'s `patch_stdout` context to avoid display corruption. Tokens are batched every ~50ms and printed as a group, balancing display responsiveness with prompt_toolkit compatibility.

When streaming is active in the CLI:

1. The response box top border prints on first token
2. Tokens are written to stdout as they arrive
3. On end-of-stream (`None` signal), the response box bottom border is printed
4. The existing spinner is suppressed during streaming

---

## Configuration

### config.yaml

**CLI streaming** (controls display in the interactive terminal):

```yaml
display:
  streaming: false        # Stream tokens to terminal. Default: false (v0.3.0), added in v0.4.0.
  show_reasoning: false   # Show reasoning/thinking blocks during streaming. Default: false.
```

**Gateway streaming** (controls progressive message editing on Telegram, Discord, Slack):

```yaml
streaming:
  enabled: false          # Master switch for gateway streaming (default: off)
  edit_interval: 0.3      # Seconds between message edits
  buffer_threshold: 40    # Characters before forcing an edit flush
  cursor: " ▉"            # Cursor shown during active streaming
```

> **v0.4.0 note (PR #2340):** CLI streaming was added as a display mode. The gateway `streaming.enabled` key remains `false` by default in `cli-config.yaml.example`. The `display.streaming` key is the CLI-specific toggle.

### Environment variable

```bash
HERMES_STREAMING_ENABLED=true
```

### Precedence order

1. API server: the client's `stream` field overrides everything -- a `stream: true` request always streams regardless of config
2. Per-platform override (e.g., `streaming.telegram: true`)
3. Master `streaming.enabled` flag
4. Default: off

### Graceful degradation

If the LLM provider does not support streaming, or the streaming connection fails for any reason, the agent falls back silently to the non-streaming path. The user sees the full response as a single block, as in v0.2.0 and earlier. This fallback is handled within `_interruptible_streaming_api_call()` which catches exceptions and retries via `_interruptible_api_call()`.

**v0.5.0 (PR #3020):** Safe non-streaming fallback is explicitly restored after stream failures, fixing a regression introduced by the always-prefer-streaming change (PR #3120).

### Always-prefer-streaming (v0.5.0 — PR #3120)

When subagents are spawned via `delegate_task`, Hermes always prefers a streaming API call over a blocking call. This prevents subagents from hanging when the parent's event loop is occupied. If streaming fails, the non-streaming fallback activates transparently.

---

## Spinner and Tool Progress During Streaming (v0.4.0 — PR #2161)

Before v0.4.0, the spinner and tool-progress display were suppressed when streaming was active. In v0.4.0, both display concurrently with streamed text:

- The spinner shows during model think/generation phases between token bursts.
- Tool call previews (the "preparing terminal…" lines) appear when the model emits tool arguments.
- The response box renders as tokens arrive; the bottom border appears on the end-of-stream signal.

The `display.streaming` key must be `true` in `config.yaml` to activate CLI streaming.

---

## Reasoning Blocks During Streaming (v0.4.0 — PR #2118)

When `display.show_reasoning: true` is set, reasoning/thinking tokens from models that produce them (Claude with extended thinking, GLM, Gemini) are displayed above the final response in a collapsible reasoning box. The box renders incrementally as reasoning tokens arrive, then the text response follows.

---

## Context Pressure Warnings (v0.4.0 — PR #2159)

When the conversation context approaches the model's limit during a streaming turn, Hermes injects a warning into the UI. This appears in both CLI and gateway:

- CLI: displayed as a dim status line between streaming turns.
- Gateway: delivered as a notice message to the chat.

The warning threshold is controlled by `compression.threshold` in `config.yaml`.

---

## Persisting Reasoning Across Gateway Turns (v0.5.0 — PR #2974)

For gateway sessions (Telegram, Discord, etc.), the session database schema v6 adds three columns to persist reasoning data across turns:

| Column | Contents |
|--------|----------|
| `reasoning` | Raw reasoning text extracted from `<think>...</think>` blocks or native thinking content |
| `reasoning_details` | Structured details (signature, model-specific fields) |
| `codex_reasoning_items` | Codex-specific reasoning summary items |

This allows the agent to carry forward reasoning context on model turns that follow tool calls, preserving coherence across multi-turn tool-use sequences in long gateway conversations.

---

## API Server Streaming

The gateway API server (`gateway/platforms/api_server.py`) supports real Server-Sent Events (SSE) streaming for `stream: true` requests on `/v1/chat/completions`.

When a client sends `stream: true`:

1. A `queue.Queue()` is created and a callback function enqueues each token
2. The agent runs as a background task via `asyncio.create_task`
3. A real SSE writer reads the queue and emits `chat.completion.chunk` events as tokens arrive
4. When `None` is received from the queue (end of stream), the SSE writer sends `data: [DONE]` and closes

For `/v1/responses`, Hermes currently returns a single JSON response (no streaming mode exposed from this endpoint in the released 2026.3.17 code path). Streaming SSE is implemented for:

1. `POST /v1/chat/completions` with `stream: true`.
2. CLI, messaging gateway platforms, and TTS paths via callback-driven stream consumers.

---

## Edge Cases

**Interrupt during streaming**: if the user sends a message while the agent is streaming, the HTTP connection is closed via the interrupt mechanism. Accumulated tokens are displayed as-is (cursor removed), and the interrupt message is processed normally.

**Context compression during streaming**: compression happens between API calls, not during a streaming turn, so it does not affect in-flight streaming tokens.

**Multi-model fallback**: if the primary model fails and the agent falls back to a different model, the streaming state resets. The fallback call may or may not support streaming; the graceful fallback in `_interruptible_streaming_api_call` handles both cases.

---

## Related Source Files

| File | Role |
|------|------|
| `run_agent.py` | `_fire_stream_delta()`, `_has_stream_consumers()`, `_interruptible_streaming_api_call()`, `stream_delta_callback` (init param), `stream_callback` (run_conversation param) |
| `gateway/run.py` | Streaming config reader, queue/callback setup, `stream_preview()` task, `already_sent` skip-final-send logic |
| `gateway/stream_consumer.py` | Maintains queue + progressive edits and sets `already_sent` when streaming produces visible output |
| `gateway/platforms/api_server.py` | `/v1/chat/completions` SSE writer, content-chunk emission, `[DONE]` close event |
| `cli.py` | CLI streaming callback setup, token display with prompt_toolkit integration |
| `gateway/platforms/api_server.py` | Real SSE writer, streaming callback wiring for `/v1/chat/completions` |
| `hermes_cli/config.py` | `streaming` config section defaults |

---

## Related Docs

- [Architecture](./architecture.md) -- agent loop, API modes, concurrent tool execution
- [Changelog](./changelog.md) -- v0.3.0 release notes
