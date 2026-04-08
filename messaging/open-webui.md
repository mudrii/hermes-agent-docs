# Open WebUI

[Open WebUI](https://github.com/open-webui/open-webui) is a self-hosted chat interface for AI. With Hermes Agent's built-in API server, you can use Open WebUI as a polished web frontend for your agent — complete with conversation management, user accounts, and a modern chat interface.

This document covers the released Open WebUI integration through v0.8.0 (v2026.4.8). No API-server-specific changes were made in v0.8.0.

---

## Architecture

```
┌──────────────────┐    POST /v1/chat/completions    ┌──────────────────────┐
│   Open WebUI     │ ──────────────────────────────► │  hermes-agent        │
│   (browser UI)   │    SSE streaming response       │  gateway API server  │
│   port 3000      │ ◄────────────────────────────── │  port 8642           │
└──────────────────┘                                  └──────────────────────┘
```

Open WebUI connects to Hermes Agent's API server just like it would connect to OpenAI. Your agent handles the requests with its full toolset — terminal, file operations, web search, memory, skills — and returns the final response.

The API server is also the integration point for any other OpenAI-compatible client (curl, Python scripts, VS Code extensions, etc.).

The API server supports streaming tool-progress updates in SSE responses and optional session continuity via `X-Hermes-Session-Id` for chat-completions clients (added in v0.7.0).

---

## Step 1: Enable the API Server

Add to `~/.hermes/.env`:

```bash
API_SERVER_ENABLED=true

# Optional: set a key for auth (recommended if accessible beyond localhost)
# API_SERVER_KEY=your-secret-key
```

---

## Step 2: Start Hermes Gateway

```bash
hermes gateway
```

You should see:

```
[API Server] API server listening on http://127.0.0.1:8642
```

Verify the server is running:

```bash
curl http://localhost:8642/health
# Returns: {"status": "ok"}

curl http://localhost:8642/v1/models
# Returns model list including hermes-agent
```

---

## Step 3: Start Open WebUI

Using Docker (with Podman substitute `docker` with `podman`):

```bash
docker run -d -p 3000:8080 \
  -e OPENAI_API_BASE_URL=http://host.docker.internal:8642/v1 \
  -e OPENAI_API_KEY=not-needed \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

If you set an `API_SERVER_KEY`, use it instead of `not-needed`:

```bash
-e OPENAI_API_KEY=your-secret-key
```

---

## Step 4: Open the UI

Go to **http://localhost:3000**. Create your admin account (the first user becomes admin). You should see **hermes-agent** in the model dropdown. Start chatting.

---

## Docker Compose Setup

For a more permanent setup, create a `docker-compose.yml`:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    volumes:
      - open-webui:/app/backend/data
    environment:
      - OPENAI_API_BASE_URL=http://host.docker.internal:8642/v1
      - OPENAI_API_KEY=not-needed
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always

volumes:
  open-webui:
```

Then:

```bash
docker compose up -d
```

---

## Configuring via the Admin UI

If you prefer to configure the connection through the UI after launch:

1. Log in to Open WebUI at **http://localhost:3000**
2. Click your **profile avatar** → **Admin Settings**
3. Go to **Connections**
4. Under **OpenAI API**, click the **wrench icon** (Manage)
5. Click **+ Add New Connection**
6. Enter:
   - **URL**: `http://host.docker.internal:8642/v1`
   - **API Key**: your key or any non-empty value (e.g., `not-needed`)
7. Click the checkmark to verify the connection
8. **Save**

Environment variables only take effect on Open WebUI's **first launch**. After that, connection settings are stored in its internal database. To change them later, use the Admin UI or delete the Docker volume and start fresh.

---

## API Type: Chat Completions vs Responses

Open WebUI supports two API modes when connecting to a backend:

| Mode | Endpoint | When to use |
|------|----------|-------------|
| **Chat Completions** (default) | `/v1/chat/completions` | Recommended. Works out of the box. |
| **Responses** (experimental) | `/v1/responses` | For server-side conversation state via `previous_response_id`. |

### Chat Completions (Recommended)

This is the default and requires no extra configuration. Open WebUI sends standard OpenAI-format requests and Hermes Agent responds accordingly. Each request includes the full conversation history.

### Responses API

To use the Responses API mode:

1. Go to **Admin Settings** → **Connections** → **OpenAI** → **Manage**
2. Edit your hermes-agent connection
3. Change **API Type** from "Chat Completions" to **"Responses (Experimental)"**
4. Save

With the Responses API, Open WebUI sends requests in the Responses format (`input` array + `instructions`), and Hermes Agent can preserve full tool call history across turns via `previous_response_id`.

Note: Open WebUI currently manages conversation history client-side even in Responses mode — it sends the full message history in each request rather than using `previous_response_id`. The Responses API mode is mainly useful for future compatibility as frontends evolve. For other compatible clients, Hermes also supports `X-Hermes-Session-Id` on `/v1/chat/completions` when you want server-side session continuity without switching to `/v1/responses`.

---

## How It Works

When you send a message in Open WebUI:

1. Open WebUI sends a `POST /v1/chat/completions` request with your message and conversation history
2. Hermes Agent creates an AIAgent instance with its full toolset
3. The agent processes your request — it may call tools (terminal, file operations, web search, etc.)
4. Tool calls happen invisibly server-side
5. The agent's final text response is returned to Open WebUI
6. Open WebUI displays the response in its chat interface

Your agent has access to all the same tools and capabilities as when using the CLI or Telegram — the only difference is the frontend.

---

## Environment Variables Reference

### Hermes Agent (API server)

| Variable | Default | Description |
|----------|---------|-------------|
| `API_SERVER_ENABLED` | `false` | Enable the API server |
| `API_SERVER_PORT` | `8642` | HTTP server port |
| `API_SERVER_HOST` | `127.0.0.1` | Bind address |
| `API_SERVER_KEY` | (none) | Bearer token for auth. No key = allow all requests. |

### Open WebUI (Docker environment variables)

| Variable | Description |
|----------|-------------|
| `OPENAI_API_BASE_URL` | Hermes Agent's API URL — must include `/v1` suffix |
| `OPENAI_API_KEY` | Must be non-empty. Match your `API_SERVER_KEY` if set. |

---

## Linux Docker Without Docker Desktop

On Linux without Docker Desktop, `host.docker.internal` does not resolve by default:

```bash
# Option 1: Add host mapping
docker run --add-host=host.docker.internal:host-gateway ...

# Option 2: Use host networking
docker run --network=host -e OPENAI_API_BASE_URL=http://localhost:8642/v1 ...

# Option 3: Use Docker bridge IP
docker run -e OPENAI_API_BASE_URL=http://172.17.0.1:8642/v1 ...
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| No models appear in the dropdown | Check the URL has `/v1` suffix. Verify the gateway is running: `curl http://localhost:8642/health`. Check model listing: `curl http://localhost:8642/v1/models`. |
| Connection test passes but no models load | Almost always the missing `/v1` suffix. The connection test is a basic connectivity check — it does not verify model listing. |
| Response takes a long time | Hermes Agent may be executing multiple tool calls before producing its final response. Streaming mode includes tool-progress updates while the agent works. |
| "Invalid API key" errors | Make sure your `OPENAI_API_KEY` in Open WebUI matches the `API_SERVER_KEY` in Hermes Agent. If no key is configured on the Hermes side, any non-empty value works. |
| Docker networking issues | From inside a Docker container, `localhost` refers to the container, not your host. Use `host.docker.internal` or `--network=host`. |
