
# API Server

The API server exposes hermes-agent as an OpenAI-compatible HTTP endpoint. Any frontend that speaks the OpenAI format — Open WebUI, LobeChat, LibreChat, NextChat, ChatBox, and hundreds more — can connect to hermes-agent and use it as a backend.

Your agent handles requests with its full toolset (terminal, file operations, web search, memory, skills) and returns the final response. When streaming, tool progress indicators appear inline so frontends can show what the agent is doing.

## Quick Start

### 1. Enable the API server

Add to `~/.hermes/.env`:

```bash
API_SERVER_ENABLED=true
API_SERVER_KEY=change-me-local-dev
# Optional: only if a browser must call Hermes directly
# API_SERVER_CORS_ORIGINS=http://localhost:3000
```

### 2. Start the gateway

```bash
hermes gateway
```

You'll see:

```
[API Server] API server listening on http://127.0.0.1:8642
```

### 3. Connect a frontend

Point any OpenAI-compatible client at `http://localhost:8642/v1`:

```bash
# Test with curl
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer change-me-local-dev" \
  -H "Content-Type: application/json" \
  -d '{"model": "hermes-agent", "messages": [{"role": "user", "content": "Hello!"}]}'
```

Or connect Open WebUI, LobeChat, or any other frontend — see the [Open WebUI integration guide](/docs/user-guide/messaging/open-webui) for step-by-step instructions.

## Endpoints

### POST /v1/chat/completions

Standard OpenAI Chat Completions format. Stateless — the full conversation is included in each request via the `messages` array.

**Request:**
```json
{
  "model": "hermes-agent",
  "messages": [
    {"role": "system", "content": "You are a Python expert."},
    {"role": "user", "content": "Write a fibonacci function"}
  ],
  "stream": false
}
```

**Response:**
```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1710000000,
  "model": "hermes-agent",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "Here's a fibonacci function..."},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 50, "completion_tokens": 200, "total_tokens": 250}
}
```

**Streaming** (`"stream": true`): Returns Server-Sent Events (SSE) with token-by-token response chunks. When streaming is enabled in config, tokens are emitted live as the LLM generates them. When disabled, the full response is sent as a single SSE chunk.

**Tool progress in streams**: When the agent calls tools during a streaming request, brief progress indicators are injected into the content stream as the tools start executing (e.g. `` `💻 pwd` ``, `` `🔍 Python docs` ``). These appear as inline markdown before the agent's response text, giving frontends like Open WebUI real-time visibility into tool execution.

In addition to the inline markdown, streaming responses now emit structured **`event: hermes.tool.progress`** SSE frames alongside the regular `data:` chunks. Each frame carries a JSON payload describing the tool name, phase (`start` / `end`), arguments preview, and a duration on completion — frontends that understand the event can render rich tool-call UI (collapsible cards, spinners, transcripts) without parsing the inline markdown. Clients that only handle standard `data:` chunks ignore these events automatically.

**Inline image inputs** — `chat/completions` accepts the OpenAI multimodal content schema, so messages can include `image_url` parts (data URLs or remote URLs) alongside text:

```json
{
  "model": "hermes-agent",
  "messages": [
    {"role": "user", "content": [
      {"type": "text", "text": "What's in this screenshot?"},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,iVBORw0K..."}}
    ]}
  ]
}
```

Images are routed through the agent's vision pipeline; both inline base64 data URLs and `https://` URLs are supported.

### POST /v1/responses

OpenAI Responses API format. Supports server-side conversation state via `previous_response_id` — the server stores full conversation history (including tool calls and results) so multi-turn context is preserved without the client managing it.

**Request:**
```json
{
  "model": "hermes-agent",
  "input": "What files are in my project?",
  "instructions": "You are a helpful coding assistant.",
  "store": true
}
```

**Response:**
```json
{
  "id": "resp_abc123",
  "object": "response",
  "status": "completed",
  "model": "hermes-agent",
  "output": [
    {"type": "function_call", "name": "terminal", "arguments": "{\"command\": \"ls\"}", "call_id": "call_1"},
    {"type": "function_call_output", "call_id": "call_1", "output": "README.md src/ tests/"},
    {"type": "message", "role": "assistant", "content": [{"type": "output_text", "text": "Your project has..."}]}
  ],
  "usage": {"input_tokens": 50, "output_tokens": 200, "total_tokens": 250}
}
```

#### Multi-turn with previous_response_id

Chain responses to maintain full context (including tool calls) across turns:

```json
{
  "input": "Now show me the README",
  "previous_response_id": "resp_abc123"
}
```

The server reconstructs the full conversation from the stored response chain — all previous tool calls and results are preserved.

The `output` array uses the OpenAI item schema with three item types:

- **`function_call`** — the agent invoked a tool, with `name`, `arguments` (JSON string), and `call_id`.
- **`function_call_output`** — the matching tool result, keyed by `call_id`.
- **`message`** — the assistant's natural-language reply (one or more `output_text` parts).

Because tool calls and outputs are stored as first-class items in the chain, follow-up requests that reference `previous_response_id` (or use a named `conversation`) get the **full** history — including which tools ran and what they returned — without the client having to re-send anything. This is the key difference from `chat/completions`, which is stateless and requires the client to replay the entire `messages` array on every turn.

#### Named conversations

Use the `conversation` parameter instead of tracking response IDs:

```json
{"input": "Hello", "conversation": "my-project"}
{"input": "What's in src/?", "conversation": "my-project"}
{"input": "Run the tests", "conversation": "my-project"}
```

The server automatically chains to the latest response in that conversation. Like the `/title` command for gateway sessions.

### GET /v1/responses/\{id\}

Retrieve a previously stored response by ID.

### DELETE /v1/responses/\{id\}

Delete a stored response.

### GET /v1/models

Lists the agent as an available model. The advertised model name defaults to the [profile](/docs/user-guide/profiles) name (or `hermes-agent` for the default profile). Required by most frontends for model discovery.

### GET /health

Health check. Returns `{"status": "ok"}`. Also available at **GET /v1/health** for OpenAI-compatible clients that expect the `/v1/` prefix.

### GET /health/detailed

Extended health check. Returns the gateway's connected platforms, current LLM provider/model, profile name, uptime, and the status of each background subsystem (scheduler, memory, webhook server, etc.). Useful for monitoring and dashboards:

```json
{
  "status": "ok",
  "profile": "default",
  "model": "hermes-agent",
  "platforms": {"telegram": "connected", "matrix": "connected"},
  "subsystems": {"scheduler": "running", "memory": "ok", "webhooks": "listening"},
  "uptime_seconds": 12345
}
```

### Runs API

The Runs API exposes long-running agent invocations as first-class objects, similar to OpenAI's Assistants/Runs surface. Useful when a single request will dispatch tools, stream partial output, and may take minutes to complete — instead of holding a streaming HTTP connection open, a client can submit the work, poll for status, and pull the final result and event stream when it's done.

| Endpoint | Description |
|----------|-------------|
| `POST /v1/runs` | Submit a new agent run. Body mirrors `POST /v1/responses` (input, instructions, conversation, etc.) plus optional metadata. Returns a run object with `id` and `status: "queued"`. |
| `GET /v1/runs/{id}` | Fetch the current state of a run. Status transitions: `queued` → `in_progress` → `completed` / `failed` / `cancelled`. The completed object includes the final `output` items and usage. |
| `GET /v1/runs/{id}/events` | SSE stream of run events — token deltas, tool-progress frames, and the terminal `run.completed` / `run.failed` event. Safe to reconnect mid-run; events are replayed from the beginning. |
| `POST /v1/runs/{id}/cancel` | Cancel an in-progress run. The agent stops at the next safe point and the run transitions to `cancelled`. |

Runs are persisted alongside stored responses, so they survive gateway restarts.

### Jobs API

The Jobs API is the scheduling counterpart to Runs — it lets a client register an agent invocation that should fire on a cron expression or one-shot delay, and inspect/cancel it later. Internally it's the same machinery that backs the gateway's `/cron` slash command.

| Endpoint | Description |
|----------|-------------|
| `POST /v1/jobs` | Create a scheduled job. Body: `{schedule, input, conversation?, deliver?}`. `schedule` is either a 5-field cron string (`"0 9 * * 1"`) or an ISO-8601 timestamp for a one-shot. Returns a job object with `id` and `next_run_at`. |
| `GET /v1/jobs` | List all jobs for the current key, with their cron expressions, last/next run timestamps, and last status. |
| `GET /v1/jobs/{id}` | Fetch a single job, including its full history of past run IDs. |
| `DELETE /v1/jobs/{id}` | Cancel and remove a scheduled job. |

When a job fires, it dispatches a Run; the resulting run ID is recorded on the job's history list and (if `deliver` was set) the final response is also pushed to the configured platform/channel.

## System Prompt Handling

When a frontend sends a `system` message (Chat Completions) or `instructions` field (Responses API), hermes-agent **layers it on top** of its core system prompt. Your agent keeps all its tools, memory, and skills — the frontend's system prompt adds extra instructions.

This means you can customize behavior per-frontend without losing capabilities:
- Open WebUI system prompt: "You are a Python expert. Always include type hints."
- The agent still has terminal, file tools, web search, memory, etc.

## Authentication

Bearer token auth via the `Authorization` header:

```
Authorization: Bearer ***
```

Configure the key via `API_SERVER_KEY` env var. If you need a browser to call Hermes directly, also set `API_SERVER_CORS_ORIGINS` to an explicit allowlist.

:::warning Security
The API server gives full access to hermes-agent's toolset, **including terminal commands**. When binding to a non-loopback address like `0.0.0.0`, `API_SERVER_KEY` is **required**. Also keep `API_SERVER_CORS_ORIGINS` narrow to control browser access.

The default bind address (`127.0.0.1`) is for local-only use. Browser access is disabled by default; enable it only for explicit trusted origins.
:::

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `API_SERVER_ENABLED` | `false` | Enable the API server |
| `API_SERVER_PORT` | `8642` | HTTP server port |
| `API_SERVER_HOST` | `127.0.0.1` | Bind address (localhost only by default) |
| `API_SERVER_KEY` | _(none)_ | Bearer token for auth |
| `API_SERVER_CORS_ORIGINS` | _(none)_ | Comma-separated allowed browser origins |
| `API_SERVER_MODEL_NAME` | _(profile name)_ | Model name on `/v1/models`. Defaults to profile name, or `hermes-agent` for default profile. |

### config.yaml

```yaml
# Not yet supported — use environment variables.
# config.yaml support coming in a future release.
```

## Security Headers

All responses include security headers:
- `X-Content-Type-Options: nosniff` — prevents MIME type sniffing
- `Referrer-Policy: no-referrer` — prevents referrer leakage

## CORS

The API server does **not** enable browser CORS by default.

For direct browser access, set an explicit allowlist:

```bash
API_SERVER_CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

When CORS is enabled:
- **Preflight responses** include `Access-Control-Max-Age: 600` (10 minute cache)
- **SSE streaming responses** include CORS headers so browser EventSource clients work correctly
- **`Idempotency-Key`** is an allowed request header — clients can send it for deduplication (responses are cached by key for 5 minutes)

Most documented frontends such as Open WebUI connect server-to-server and do not need CORS at all.

## Compatible Frontends

Any frontend that supports the OpenAI API format works. Tested/documented integrations:

| Frontend | Stars | Connection |
|----------|-------|------------|
| [Open WebUI](/docs/user-guide/messaging/open-webui) | 126k | Full guide available |
| LobeChat | 73k | Custom provider endpoint |
| LibreChat | 34k | Custom endpoint in librechat.yaml |
| AnythingLLM | 56k | Generic OpenAI provider |
| NextChat | 87k | BASE_URL env var |
| ChatBox | 39k | API Host setting |
| Jan | 26k | Remote model config |
| HF Chat-UI | 8k | OPENAI_BASE_URL |
| big-AGI | 7k | Custom endpoint |
| OpenAI Python SDK | — | `OpenAI(base_url="http://localhost:8642/v1")` |
| curl | — | Direct HTTP requests |

## Multi-User Setup with Profiles

To give multiple users their own isolated Hermes instance (separate config, memory, skills), use [profiles](/docs/user-guide/profiles):

```bash
# Create a profile per user
hermes profile create alice
hermes profile create bob

# Configure each profile's API server on a different port
hermes -p alice config set API_SERVER_ENABLED true
hermes -p alice config set API_SERVER_PORT 8643
hermes -p alice config set API_SERVER_KEY alice-secret

hermes -p bob config set API_SERVER_ENABLED true
hermes -p bob config set API_SERVER_PORT 8644
hermes -p bob config set API_SERVER_KEY bob-secret

# Start each profile's gateway
hermes -p alice gateway &
hermes -p bob gateway &
```

Each profile's API server automatically advertises the profile name as the model ID:

- `http://localhost:8643/v1/models` → model `alice`
- `http://localhost:8644/v1/models` → model `bob`

In Open WebUI, add each as a separate connection. The model dropdown shows `alice` and `bob` as distinct models, each backed by a fully isolated Hermes instance. See the [Open WebUI guide](/docs/user-guide/messaging/open-webui#multi-user-setup-with-profiles) for details.

## Limitations

- **Response storage** — stored responses (for `previous_response_id`) are persisted in SQLite and survive gateway restarts. Max 100 stored responses (LRU eviction).
- **Model field is cosmetic** — the `model` field in requests is accepted but the actual LLM model used is configured server-side in config.yaml.
