# API Server

The API server exposes Hermes Agent as an OpenAI-compatible HTTP endpoint. Any frontend that speaks the OpenAI API format — Open WebUI, LobeChat, LibreChat, NextChat, ChatBox, and hundreds more — can connect to Hermes and use it as a backend.

Your agent handles requests with its full toolset (terminal, file operations, web search, memory, skills) and returns the final response. Tool calls execute invisibly server-side.

## Quick Start

### 1. Enable the API server

Add to `~/.hermes/.env`:

```bash
API_SERVER_ENABLED=true
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
  -H "Content-Type: application/json" \
  -d '{"model": "hermes-agent", "messages": [{"role": "user", "content": "Hello!"}]}'
```

## Endpoints

### POST /v1/chat/completions

Standard OpenAI Chat Completions format. By default this is stateless — the full conversation is included in each request via the `messages` array. Clients can opt into server-side continuity by sending and reusing an `X-Hermes-Session-Id` header.

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

**Streaming** (`"stream": true`): Returns Server-Sent Events (SSE) with token-by-token response chunks. The SSE stream includes Hermes tool-progress updates so Open WebUI-style frontends can see activity while the agent is working. When streaming is disabled in config, the full response is sent as a single SSE chunk.

#### Session continuity with `X-Hermes-Session-Id`

To continue a chat-completions session without the client owning the full continuity state, send a stable session header:

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "X-Hermes-Session-Id: my-project-session" \
  -d '{"model":"hermes-agent","messages":[{"role":"user","content":"List the repo files"}]}'
```

Hermes returns the active session header on the response as well, so clients can persist or reuse it on later requests.

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

The server reconstructs the full conversation from the stored response chain — all previous tool calls and results are preserved. Response chains persist in the shared SessionDB-backed response store, so continuity survives normal process restarts.

#### Named conversations

Use the `conversation` parameter instead of tracking response IDs:

```json
{"input": "Hello", "conversation": "my-project"}
{"input": "What's in src/?", "conversation": "my-project"}
{"input": "Run the tests", "conversation": "my-project"}
```

The server automatically chains to the latest response in that named conversation.

### GET /v1/responses/{id}

Retrieve a previously stored response by ID.

### DELETE /v1/responses/{id}

Delete a stored response.

### GET /v1/models

Lists `hermes-agent` as an available model. Required by most frontends for model discovery.

### GET /health

Health check endpoint. Returns `{"status": "ok", "platform": "hermes-agent"}`.

## Cron Jobs REST API

The API server exposes a REST API for managing scheduled jobs. All endpoints require the same Bearer token auth as the OpenAI endpoints (if `API_SERVER_KEY` is configured).

### GET /api/jobs

List all cron jobs.

Query parameters:
- `include_disabled=true` — include disabled jobs (omitted by default)

**Response:** `{"jobs": [...]}`

### POST /api/jobs

Create a new cron job.

**Request:**

```json
{
  "name": "Morning report",
  "schedule": "0 9 * * *",
  "prompt": "Check Hacker News for AI news and summarize",
  "deliver": "local",
  "skills": ["blogwatcher"],
  "repeat": 5
}
```

Fields:
- `name` (required) — max 200 characters
- `schedule` (required) — cron expression, interval (`every 2h`), or one-shot delay (`30m`)
- `prompt` — task instruction, max 5 000 characters
- `deliver` — delivery target (`local`, `origin`, `telegram`, etc.)
- `skills` — list of skill names to load
- `repeat` — positive integer; limits execution count

**Response:** `{"job": {...}}`

### GET /api/jobs/{job_id}

Get a single job by ID. Returns `{"job": {...}}` or 404.

### PATCH /api/jobs/{job_id}

Update a job. Accepted fields: `name`, `schedule`, `prompt`, `deliver`, `skills`, `skill`, `repeat`, `enabled`. All other keys are silently ignored.

**Response:** `{"job": {...}}`

### DELETE /api/jobs/{job_id}

Delete a job. Returns `{"ok": true}` or 404.

### POST /api/jobs/{job_id}/pause

Pause a running job. Returns `{"job": {...}}`.

### POST /api/jobs/{job_id}/resume

Resume a paused job. Returns `{"job": {...}}`.

### POST /api/jobs/{job_id}/run

Trigger immediate execution on the next scheduler tick. Returns `{"job": {...}}`.

## System Prompt Handling

When a frontend sends a `system` message (Chat Completions) or `instructions` field (Responses API), Hermes Agent **layers it on top** of its core system prompt. Your agent keeps all its tools, memory, and skills — the frontend's system prompt adds extra instructions.

This means you can customize behavior per-frontend without losing capabilities:

```json
{"role": "system", "content": "You are a Python expert. Always include type hints."}
```

The agent still has terminal, file tools, web search, memory, and skills — the system message adds extra constraints.

## Authentication

Bearer token authentication via the `Authorization` header:

```
Authorization: Bearer your-token-here
```

Configure the key via the `API_SERVER_KEY` environment variable. If no key is set, all requests are allowed (safe for local-only use since the default bind address is `127.0.0.1`).

**Security warning:** The API server gives full access to Hermes Agent's toolset, including terminal commands. If you change the bind address to `0.0.0.0` (network-accessible), always set `API_SERVER_KEY` — without it, anyone on your network can execute arbitrary commands on your machine.

## CORS

CORS is controlled by the `API_SERVER_CORS_ORIGINS` setting. By default no CORS headers are sent, which means non-browser clients (curl, Python SDK, mobile apps) work without restriction, while browser-based frontends require an explicit origin allowlist.

Configure allowed origins as a comma-separated list:

```bash
API_SERVER_CORS_ORIGINS=http://localhost:3000,https://my-chat.example.com
```

Use `*` to allow any origin (equivalent to the old open-CORS behaviour):

```bash
API_SERVER_CORS_ORIGINS=*
```

Requests from an unlisted browser origin receive a `403` response. The `Idempotency-Key` header is included in CORS preflight responses (`Access-Control-Allow-Headers`), so clients that send idempotency keys work correctly from the browser (PR #3530).

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `API_SERVER_ENABLED` | `false` | Enable the API server |
| `API_SERVER_PORT` | `8642` | HTTP server port |
| `API_SERVER_HOST` | `127.0.0.1` | Bind address (localhost only by default) |
| `API_SERVER_KEY` | _(none)_ | Bearer token for authentication |
| `API_SERVER_CORS_ORIGINS` | _(none)_ | Comma-separated allowed browser origins, or `*` for any |

All configuration is via environment variables in `~/.hermes/.env`. `config.yaml` support for these settings is not yet available.

## Compatible Frontends

Any frontend that supports the OpenAI API format works. Tested and documented integrations:

| Frontend | Stars | Connection |
|----------|-------|------------|
| Open WebUI | 126k | Full guide available in the official Hermes docs |
| LobeChat | 73k | Custom provider endpoint |
| LibreChat | 34k | Custom endpoint in `librechat.yaml` |
| AnythingLLM | 56k | Generic OpenAI provider |
| NextChat | 87k | `BASE_URL` environment variable |
| ChatBox | 39k | API Host setting |
| Jan | 26k | Remote model config |
| HF Chat-UI | 8k | `OPENAI_BASE_URL` |
| big-AGI | 7k | Custom endpoint |
| OpenAI Python SDK | — | `OpenAI(base_url="http://localhost:8642/v1")` |

### OpenAI Python SDK example

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8642/v1",
    api_key="your-token-or-anything-if-no-key-set"
)

response = client.chat.completions.create(
    model="hermes-agent",
    messages=[{"role": "user", "content": "List files in the current directory"}]
)
print(response.choices[0].message.content)
```

## Security Features

The API server is hardened with several built-in protections (PR #2450, #2451, #2456, #2472):

- **Request body limit** — POST bodies larger than 1 MB are rejected with a `413` response (OpenAI-format error envelope).
- **Field whitelist on job updates** — `PATCH /api/jobs/{id}` only accepts `name`, `schedule`, `prompt`, `deliver`, `skills`, `skill`, `repeat`, and `enabled`. All other keys are silently ignored.
- **Job ID format validation** — Job IDs must match `[a-f0-9]{12}` before any cron operation executes.
- **SQLite-backed response persistence** — `POST /v1/responses` stores responses in `~/.hermes/response_store.db`. Responses survive gateway restarts (up to the 100-entry LRU cap).
- **CORS origin protection** — Browser requests from unlisted origins are rejected with a `403`. Non-browser clients (no `Origin` header) are always allowed.
- **Input length limits on cron** — `name` is capped at 200 characters; `prompt` is capped at 5 000 characters.

## Idempotency

Both `POST /v1/chat/completions` and `POST /v1/responses` support the `Idempotency-Key` header (PR #2903, #3530). Send the same key with the same request body and the cached response is returned without re-running the agent. The cache is in-memory with a 5-minute TTL and a 1 000-entry LRU cap.

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: req_abc123" \
  -d '{"model": "hermes-agent", "messages": [{"role": "user", "content": "Hello"}]}'
```

## What's New

### v0.8.0

- **Conversation history preserved on `/v1/runs` multi-message input** — When `/v1/runs` receives an OpenAI-style array of messages, all messages except the last user turn are extracted as `conversation_history`. Previously only the last message was kept, silently discarding earlier turns.
- **Session boundary hooks** — Plugins can now subscribe to `on_session_finalize` (fires on exit/before `/new`) and `on_session_reset` (fires after `/new`) via `ctx.register_hook()`. These hooks are also covered by the gateway test suite ([#6129](https://github.com/NousResearch/hermes-agent/pull/6129)).
- **API server streaming fix + `conversation_history` in request body** — Streaming reliability improved; `conversation_history` can now be passed explicitly in the request body ([#5977](https://github.com/NousResearch/hermes-agent/pull/5977)).

### v0.5.0 (PR #3427, #3530, #3537, #3304)

- **Cancel orphaned agent + true interrupt on SSE disconnect** (PR #3427) — When a streaming client disconnects, the agent's `interrupt()` method is called to stop in-progress LLM API calls, and the asyncio task is cancelled. Previously, disconnects left orphaned agent threads running to completion.
- **Allow `Idempotency-Key` in CORS headers** (PR #3530) — Browser frontends that send idempotency keys now work correctly without CORS preflight errors.
- **Auto-repair `jobs.json` with invalid control characters** (PR #3537) — If `jobs.json` contains bare control characters (e.g., from a crash mid-write), the file is automatically repaired on load and rewritten with proper JSON escaping.
- **Explicit `hermes-api-server` toolset** (PR #3304) — The API server platform now has its own named toolset in `platform_toolsets.api_server` configuration.

### v0.4.0 (PR #1756, #2450, #2456, #2451, #2472)

- OpenAI-compatible API server introduced as a gateway platform adapter
- `POST /v1/chat/completions` and `POST /v1/responses` endpoints
- `/api/jobs` REST API for cron job management
- Request body size limits, field whitelists, and CORS origin protection
- SQLite-backed response persistence for `previous_response_id` chaining
- Idempotency-Key support on both endpoints (PR #2903)

## Limitations

- **Response storage cap** — Maximum 100 stored responses (SQLite-backed LRU). Oldest entries are evicted when the cap is reached.
- **No file upload** — vision/document analysis via uploaded files is not yet supported through the API.
- **Model field is cosmetic** — the `model` field in requests is accepted but the actual LLM model used is configured server-side in `config.yaml`.
- **Config via env only** — `config.yaml` support for API server settings is not yet available.
- **Truncation** — Responses API supports `"truncation": "auto"` which limits conversation history to the last 100 messages when set.
- **Store control** — Set `"store": false` in Responses API to skip saving the response (useful for one-off queries where chaining is not needed).
- **`conversation` and `previous_response_id` are mutually exclusive** — using both in the same request returns a 400 error.
