
# Agent Loop Internals

The core orchestration engine is `run_agent.py`'s `AIAgent` class — roughly 10,700 lines that handle everything from prompt assembly to tool dispatch to provider failover.

## Core Responsibilities

`AIAgent` is responsible for:

- Assembling the effective system prompt and tool schemas via `prompt_builder.py`
- Selecting the correct provider/API mode (chat_completions, codex_responses, anthropic_messages)
- Making interruptible model calls with cancellation support
- Executing tool calls (sequentially or concurrently via thread pool)
- Maintaining conversation history in OpenAI message format
- Handling compression, retries, and fallback model switching
- Tracking iteration budgets across parent and child agents
- Flushing persistent memory before context is lost

## Two Entry Points

```python
# Simple interface — returns final response string
response = agent.chat("Fix the bug in main.py")

# Full interface — returns dict with messages, metadata, usage stats
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    system_message=None,           # auto-built if omitted
    conversation_history=None,      # auto-loaded from session if omitted
    task_id="task_abc123"
)
```

`chat()` is a thin wrapper around `run_conversation()` that extracts the `final_response` field from the result dict.

## Transport Dispatch

In v0.11.0, format conversion and response normalization were extracted from `run_agent.py` into the [Transport Layer](./transport-layer.md). The agent loop no longer carries inline `if api_mode == ...` branches for each protocol; instead it looks up a `ProviderTransport` from `agent/transports/` keyed by the resolved `api_mode`:

```python
from agent.transports import get_transport

transport = get_transport(api_mode)
kwargs = transport.build_kwargs(model, messages, tools, **params)
raw = client.create(**kwargs)            # streaming or non-streaming
nr = transport.normalize_response(raw)   # → NormalizedResponse
```

The four shipped `api_mode`s and their transport classes:

| `api_mode` | Used for | Transport |
|------------|----------|-----------|
| `chat_completions` | OpenAI-compatible endpoints (OpenRouter, Nous Portal, NVIDIA NIM, Step Plan, ai-gateway, xAI chat, Qwen, etc.) | `ChatCompletionsTransport` |
| `codex_responses` | OpenAI Codex / Responses API (incl. GPT-5.5 over Codex OAuth, xAI Grok Responses) | `ResponsesApiTransport` |
| `anthropic_messages` | Native Anthropic Messages API | `AnthropicTransport` |
| `bedrock_converse` | AWS Bedrock Converse API (v0.11.0) | `BedrockTransport` |

The transport owns format conversion, kwargs assembly, and normalization. Streaming, retries, prompt caching, credential refresh, and client construction stay on `AIAgent`.

**Mode resolution order:**
1. Explicit `api_mode` constructor arg (highest priority)
2. Provider-specific detection (e.g., `anthropic` provider → `anthropic_messages`, `bedrock` provider → `bedrock_converse`)
3. Base URL heuristics (e.g., `api.anthropic.com` → `anthropic_messages`)
4. Default: `chat_completions`

## Turn Lifecycle

Each iteration of the agent loop follows this sequence:

```text
run_conversation()
  1. Generate task_id if not provided
  2. Append user message to conversation history
  3. Build or reuse cached system prompt (prompt_builder.py)
  4. Check if preflight compression is needed (>50% context)
  5. Build API messages from conversation history
     - chat_completions: OpenAI format as-is
     - codex_responses: convert to Responses API input items
     - anthropic_messages: convert via anthropic_adapter.py
  6. Inject ephemeral prompt layers (budget warnings, context pressure)
  7. Apply prompt caching markers if on Anthropic
  8. Make interruptible API call (_api_call_with_interrupt)
  9. Parse response:
     - If tool_calls: execute them, append results, loop back to step 5
     - If text response: persist session, flush memory if needed, return
```

### Mid-Run Steering (v0.11.0)

Between turns, a user can call `/steer <prompt>` (CLI or any gateway platform) to inject a note that the running agent sees after its next tool call, without interrupting the current turn or breaking prompt cache. The pending steer message is queued by the CLI / gateway adapter, then merged into the next user-role message the loop sees in step 5 of the turn lifecycle. Because the injection lands between turns, prompt-cache markers stay aligned and no streaming response is discarded.

### Auto-Continue Interrupted Work (v0.11.0)

When the gateway restarts (or the CLI is relaunched against a session that was mid-turn), the agent loop detects the interrupted state — a session whose last persisted message is an assistant tool-call without matching tool results — and automatically re-issues a "continue" turn so work in progress is resumed instead of dropped. This is paired with **activity heartbeats**: the agent emits periodic activity pings during long-running tool execution so platform-side gateways do not falsely classify the session as inactive and tear down the connection.

### Orchestrator Role and `max_spawn_depth` (v0.11.0)

`delegate_task` now distinguishes two subagent roles:

- **Worker** (default, leaf) — executes a focused task; cannot itself call `delegate_task`.
- **Orchestrator** — can spawn workers (and, transitively, more orchestrators) up to a configurable depth.

Three knobs control the interaction with the agent loop:

```yaml
delegation:
  orchestrator_enabled: true        # global on/off; default true
  max_spawn_depth: 1                # 1-3; default 1 (flat — only the top-level agent delegates)
  max_concurrent_children: 3        # default 3 parallel siblings
```

At depth 1 (the default), only the top-level agent retains `delegate_task` in its tool list — child workers cannot delegate. At depth 2-3, intermediate orchestrators retain the tool until the leaf level. Concurrent siblings share filesystem state through a cross-agent file-coordination layer so they do not clobber each other's edits.

### Message Format

All messages use OpenAI-compatible format internally:

```python
{"role": "system", "content": "..."}
{"role": "user", "content": "..."}
{"role": "assistant", "content": "...", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "...", "content": "..."}
```

Reasoning content (from models that support extended thinking) is stored in `assistant_msg["reasoning"]` and optionally displayed via the `reasoning_callback`.

### Message Alternation Rules

The agent loop enforces strict message role alternation:

- After the system message: `User → Assistant → User → Assistant → ...`
- During tool calling: `Assistant (with tool_calls) → Tool → Tool → ... → Assistant`
- **Never** two assistant messages in a row
- **Never** two user messages in a row
- **Only** `tool` role can have consecutive entries (parallel tool results)

Providers validate these sequences and will reject malformed histories.

## Interruptible API Calls

API requests are wrapped in `_api_call_with_interrupt()` which runs the actual HTTP call in a background thread while monitoring an interrupt event:

```text
┌──────────────────────┐     ┌──────────────┐
│  Main thread         │     │  API thread   │
│  wait on:            │────▶│  HTTP POST    │
│  - response ready    │     │  to provider  │
│  - interrupt event   │     └──────────────┘
│  - timeout           │
└──────────────────────┘
```

When interrupted (user sends new message, `/stop` command, or signal):
- The API thread is abandoned (response discarded)
- The agent can process the new input or shut down cleanly
- No partial response is injected into conversation history

## Tool Execution

### Sequential vs Concurrent

When the model returns tool calls:

- **Single tool call** → executed directly in the main thread
- **Multiple tool calls** → executed concurrently via `ThreadPoolExecutor`
  - Exception: tools marked as interactive (e.g., `clarify`) force sequential execution
  - Results are reinserted in the original tool call order regardless of completion order

### Execution Flow

```text
for each tool_call in response.tool_calls:
    1. Resolve handler from tools/registry.py
    2. Fire pre_tool_call plugin hook
    3. Check if dangerous command (tools/approval.py)
       - If dangerous: invoke approval_callback, wait for user
    4. Execute handler with args + task_id
    5. Fire post_tool_call plugin hook
    6. Append {"role": "tool", "content": result} to history
```

### Agent-Level Tools

Some tools are intercepted by `run_agent.py` *before* reaching `handle_function_call()`:

| Tool | Why intercepted |
|------|--------------------|
| `todo` | Reads/writes agent-local task state |
| `memory` | Writes to persistent memory files with character limits |
| `session_search` | Queries session history via the agent's session DB |
| `delegate_task` | Spawns subagent(s) with isolated context |

These tools modify agent state directly and return synthetic tool results without going through the registry.

## Callback Surfaces

`AIAgent` supports platform-specific callbacks that enable real-time progress in the CLI, gateway, and ACP integrations:

| Callback | When fired | Used by |
|----------|-----------|---------|
| `tool_progress_callback` | Before/after each tool execution | CLI spinner, gateway progress messages |
| `thinking_callback` | When model starts/stops thinking | CLI "thinking..." indicator |
| `reasoning_callback` | When model returns reasoning content | CLI reasoning display, gateway reasoning blocks |
| `clarify_callback` | When `clarify` tool is called | CLI input prompt, gateway interactive message |
| `step_callback` | After each complete agent turn | Gateway step tracking, ACP progress |
| `stream_delta_callback` | Each streaming token (when enabled) | CLI streaming display |
| `tool_gen_callback` | When tool call is parsed from stream | CLI tool preview in spinner |
| `status_callback` | State changes (thinking, executing, etc.) | ACP status updates |

## Budget and Fallback Behavior

### Iteration Budget

The agent tracks iterations via `IterationBudget`:

- Default: 90 iterations (configurable via `agent.max_turns`)
- Each agent gets its own budget. Subagents get independent budgets capped at `delegation.max_iterations` (default 50) — total iterations across parent + subagents can exceed the parent's cap
- At 100%, the agent stops and returns a summary of work done

### Fallback Model

When the primary model fails (429 rate limit, 5xx server error, 401/403 auth error):

1. Check `fallback_providers` list in config
2. Try each fallback in order
3. On success, continue the conversation with the new provider
4. On 401/403, attempt credential refresh before failing over

The fallback system also covers auxiliary tasks independently — vision, compression, web extraction, and session search each have their own fallback chain configurable via the `auxiliary.*` config section.

### Auxiliary Fallback to Main on 503/404 (v0.11.0)

Auxiliary tasks (compression, title generation, summarization) auto-fall back to the **main** model when the configured auxiliary provider returns a permanent 503 or 404 — for example, when an auxiliary model has been deprovisioned or the auxiliary endpoint is unreachable. The fallback applies per-task and is logged once per session so the user can see why the main model is being used for what would normally be an aux call. Transient 5xx errors still go through the normal retry loop before this fallback fires.

## Compression and Persistence

### When Compression Triggers

- **Preflight** (before API call): If conversation exceeds 50% of model's context window
- **Gateway auto-compression**: If conversation exceeds 85% (more aggressive, runs between turns)

### What Happens During Compression

1. Memory is flushed to disk first (preventing data loss)
2. Middle conversation turns are summarized into a compact summary
3. The last N messages are preserved intact (`compression.protect_last_n`, default: 20)
4. Tool call/result message pairs are kept together (never split)
5. A new session lineage ID is generated (compression creates a "child" session)

### Session Persistence

After each turn:
- Messages are saved to the session store (SQLite via `hermes_state.py`)
- Memory changes are flushed to `MEMORY.md` / `USER.md`
- The session can be resumed later via `/resume` or `hermes chat --resume`

## Key Source Files

| File | Purpose |
|------|---------|
| `run_agent.py` | AIAgent class — the complete agent loop (~10,700 lines) |
| `agent/prompt_builder.py` | System prompt assembly from memory, skills, context files, personality |
| `agent/context_engine.py` | ContextEngine ABC — pluggable context management |
| `agent/context_compressor.py` | Default engine — lossy summarization algorithm |
| `agent/prompt_caching.py` | Anthropic prompt caching markers and cache metrics |
| `agent/auxiliary_client.py` | Auxiliary LLM client for side tasks (vision, summarization) |
| `model_tools.py` | Tool schema collection, `handle_function_call()` dispatch |

## Related Docs

- [Transport Layer](./transport-layer.md)
- [Provider Runtime Resolution](./provider-runtime.md)
- [Prompt Assembly](./prompt-assembly.md)
- [Context Compression & Prompt Caching](./context-compression-and-caching.md)
- [Tools Runtime](./tools-runtime.md)
- [Architecture Overview](./architecture.md)
