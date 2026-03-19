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

Health check endpoint. Returns `{"status": "ok"}`.

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

The API server includes CORS headers on all responses (`Access-Control-Allow-Origin: *`), so browser-based frontends can connect directly without a proxy.

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `API_SERVER_ENABLED` | `false` | Enable the API server |
| `API_SERVER_PORT` | `8642` | HTTP server port |
| `API_SERVER_HOST` | `127.0.0.1` | Bind address (localhost only by default) |
| `API_SERVER_KEY` | _(none)_ | Bearer token for authentication |

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

## Limitations

- **Response storage is in-memory** — stored responses for `previous_response_id` are lost on gateway restart. Maximum 100 stored responses (LRU eviction).
- **No file upload** — vision/document analysis via uploaded files is not yet supported through the API.
- **Model field is cosmetic** — the `model` field in requests is accepted but the actual LLM model used is configured server-side in `config.yaml`.
- **Config via env only** — `config.yaml` support for API server settings is not yet available.
- **Truncation** — Responses API supports `"truncation": "auto"` which limits conversation history to the last 100 messages when set.
- **Store control** — Set `"store": false` in Responses API to skip saving the response (useful for one-off queries where chaining is not needed).
- **`conversation` and `previous_response_id` are mutually exclusive** — using both in the same request returns a 400 error.
