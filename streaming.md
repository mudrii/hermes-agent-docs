# Unified Streaming

Hermes v0.3.0 (v2026.3.17) introduced a unified streaming infrastructure that delivers LLM responses token-by-token across all platforms. Before this release, all responses arrived as a single block after the model finished generating. With streaming, text appears progressively as it is produced.

Streaming is implemented via PR [#1538](https://github.com/NousResearch/hermes-agent/pull/1538).

---

## What Unified Streaming Is

Unified streaming means the same streaming mechanism in `AIAgent` (`run_agent.py`) works identically regardless of whether the consumer is the CLI, a messaging gateway platform, or an API server. The agent does not know or care what the consumer does with each token — it emits tokens through a callback, and each platform handles display in the appropriate way.

Before v0.3.0:

- The agent waited for the LLM to complete the full response
- The entire response was delivered to the CLI or messaging platform at once
- Users saw a spinner, then a complete response block
- No incremental feedback during generation

After v0.3.0:

- The agent opens a streaming connection to the LLM API
- Each text token is delivered via `stream_callback(text_delta: str)` as it arrives
- The CLI prints tokens to the terminal progressively
- Gateway platforms (Telegram, Discord, Slack) send an initial message then edit it with accumulated content at regular intervals
- A `None` signal (`stream_callback(None)`) marks end of stream so consumers can finalize display

---

## Architecture

The streaming bridge is a thread-safe `queue.Queue()` connecting the agent thread to the consumer.

```
                          stream_callback(delta)
                                │
  ┌─────────────┐    ┌──────────▼──────────────┐
  │  LLM API    │    │      queue.Queue()       │
  │  (stream)   │───►│  thread-safe bridge      │
  │             │    │  agent thread ↔ consumer │
  └─────────────┘    └──────────┬───────────────┘
                                │
                 ┌──────────────┼──────────────┐
                 │              │              │
           ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
           │    CLI     │ │  Gateway  │ │ API Server│
           │ print to   │ │ edit msg  │ │ SSE event │
           │ terminal   │ │ on Tg/Dc  │ │ to client │
           └───────────┘ └───────────┘ └───────────┘
```

The `AIAgent` class in `run_agent.py` accepts an optional `stream_callback: callable = None` parameter. When `None`, all existing code paths are unchanged — the non-streaming path is the fallback for any failure.

---

## How Token Delivery Works

### API Modes

Hermes supports three API modes. Streaming is wired into each:

**Chat Completions API** (`chat_completions` mode)

The `_run_streaming_chat_completion()` method in `run_agent.py`:

1. Adds `stream=True` and `stream_options={"include_usage": True}` to the API call
2. Iterates chunks from the stream
3. For text content (`delta.content`): appends to accumulated buffer and calls `stream_callback(delta.content)`
4. For tool call deltas: accumulates silently into a dict indexed by `tc_delta.index`
5. At stream end, constructs a fake response object using `SimpleNamespace` that is compatible with the non-streaming code path
6. On any exception, falls back to `self.client.chat.completions.create(**api_kwargs)` (non-streaming)

**Codex Responses API** (`codex_responses` mode)

The existing `_run_codex_stream()` method already iterated the stream. In v0.3.0 it was extended to emit text deltas: when `event.type == 'response.output_text.delta'`, `stream_callback(event.delta)` is called.

**Anthropic Messages API** (`anthropic_messages` mode)

Native Anthropic streaming is handled through the `AnthropicAuxiliaryClient` in `agent/auxiliary_client.py`, using the Anthropic SDK's streaming interface.

### End-of-Stream Signal

After `_interruptible_api_call()` returns, the agent sends `stream_callback(None)`. This `None` token is the end-of-stream signal. Consumers use it to:

- Remove the typing cursor from the terminal display
- Remove the cursor character (`▌`) from the last gateway message edit
- Close SSE connections in the API server

### Tool Calls During Streaming

When the model returns tool calls instead of text, no text tokens are emitted — `stream_callback` is never called with text content in that turn. Tool progress messages continue to appear through the existing tool display mechanism. After tools execute, if the model produces a final text response, streaming picks up again.

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

When streaming is active and tokens were delivered via progressive edits, the normal final-response `send()` call is suppressed to prevent duplicate messages. The result dict carries a `_streamed_msg_id` marker that tells the base gateway adapter to skip its normal send path.

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

```yaml
streaming:
  enabled: false          # Master switch (default: off)
  cli: true               # Per-platform override
  telegram: true
  discord: true
  slack: true
  api_server: true        # API server always streams when client requests it
  edit_interval: 1.5      # Seconds between message edits (default: 1.5)
  min_tokens: 20          # Tokens before first display (default: 20)
```

### Environment variable

```bash
HERMES_STREAMING_ENABLED=true
```

### Precedence order

1. API server: the client's `stream` field overrides everything — a `stream: true` request always streams regardless of config
2. Per-platform override (e.g., `streaming.telegram: true`)
3. Master `streaming.enabled` flag
4. Default: off

### Graceful degradation

If the LLM provider does not support streaming, or the streaming connection fails for any reason, the agent falls back silently to the non-streaming path. The user sees the full response as a single block, as in v0.2.0 and earlier.

---

## API Server Streaming

The gateway API server (`gateway/platforms/api_server.py`) supports real Server-Sent Events (SSE) streaming for `stream: true` requests on `/v1/chat/completions`.

When a client sends `stream: true`:

1. A `queue.Queue()` is created and a `_api_stream_callback` function enqueues each token
2. The agent runs as a background task via `asyncio.create_task`
3. A real SSE writer reads the queue and emits `chat.completion.chunk` events as tokens arrive
4. When `None` is received from the queue (end of stream), the SSE writer sends `data: [DONE]` and closes

For `/v1/responses` with `stream: true`, the events follow the Responses API format (`response.output_text.delta`, `response.completed`).

---

## Edge Cases

**Interrupt during streaming**: if the user sends a message while the agent is streaming, the HTTP connection is closed via `_interruptible_api_call`. Accumulated tokens are displayed as-is (cursor removed), and the interrupt message is processed normally.

**Context compression during streaming**: compression happens between API calls, not during a streaming turn, so it does not affect in-flight streaming tokens.

**Multi-model fallback**: if the primary model fails and the agent falls back to a different model, the streaming state resets. The fallback call may or may not support streaming; the graceful fallback in `_run_streaming_chat_completion` handles both cases.

---

## Related Source Files

| File | Role |
|------|------|
| `run_agent.py` | `AIAgent.stream_callback`, `_run_streaming_chat_completion()`, `_run_codex_stream()`, `_interruptible_api_call()` |
| `gateway/run.py` | Streaming config reader, queue/callback setup, `stream_preview()` task, skip-final-send logic |
| `gateway/platforms/base.py` | Checks `_streamed_msg_id` in response handler to suppress duplicate send |
| `cli.py` | CLI streaming callback setup, token display with prompt_toolkit integration |
| `gateway/platforms/api_server.py` | Real SSE writer, streaming callback wiring for `/v1/chat/completions` |
| `hermes_cli/config.py` | `streaming` config section defaults |

---

## Related Docs

- [Architecture](./architecture.md) — agent loop, API modes, concurrent tool execution
- [Gateway](./gateway.md) — messaging platform setup and configuration
- [CLI Reference](./cli-reference.md) — CLI commands and flags
