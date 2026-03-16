# ACP Integration

Hermes Agent supports the **Agent Communication Protocol (ACP)**, enabling integration with editors (VS Code, Zed, JetBrains) and other ACP-compatible clients.

---

## What is ACP?

ACP (Agent Communication Protocol) is a standardized JSON-RPC protocol for running AI agents as editor-native services. It runs over stdio, allowing editors to spawn Hermes as a subprocess and communicate in real time.

**Entry point:** `hermes-acp`

---

## Editor Setup

### VS Code

Install the ACP extension, then configure it to use Hermes:

```json
// .vscode/settings.json
{
  "acp.agents": [
    {
      "name": "Hermes",
      "command": "hermes-acp",
      "args": []
    }
  ]
}
```

### Zed

```json
// ~/.config/zed/settings.json
{
  "acp": {
    "agents": [
      {
        "name": "Hermes",
        "binary": "hermes-acp"
      }
    ]
  }
}
```

### JetBrains

Configure via the ACP plugin settings: set the binary path to `hermes-acp`.

---

## ACP Session Lifecycle

```
Editor                         ACP Server (hermes-acp)          AIAgent
  │                                   │                            │
  ├── initialize() ─────────────────> │                            │
  │ <── InitializeResponse ────────── │                            │
  │                                   │                            │
  ├── authenticate(method_id) ──────> │                            │
  │ <── AuthenticateResponse ──────── │                            │
  │                                   │                            │
  ├── new_session(cwd) ─────────────> │                            │
  │                                   ├── create_session() ──────> │
  │ <── NewSessionResponse ─────────  │                            │
  │                                   │                            │
  ├── prompt(session_id, text) ─────> │                            │
  │                                   ├── run_conversation() ────> │
  │ <── session_update (thinking) ─── │ <── thinking_callback ──── │
  │ <── session_update (tool_start) ─ │ <── tool_progress ──────── │
  │ <── session_update (tool_end) ─── │ <── step_callback ──────── │
  │ <── session_update (message) ──── │ <── message_callback ────── │
  │ <── PromptResponse ─────────────  │                            │
  │                                   │                            │
  ├── cancel(session_id) ───────────> │                            │
  │                                   ├── interrupt() ──────────── │
  └── disconnect() ─────────────────> │                            │
```

---

## ACP Methods

### `initialize()`

Returns agent metadata and supported authentication methods.

**Response:**
```json
{
  "agent": {
    "name": "hermes",
    "version": "0.2.0",
    "description": "The self-improving AI agent by Nous Research"
  },
  "auth_methods": ["nous", "openrouter", "anthropic"]
}
```

### `authenticate(method_id)`

Validates credentials for the selected provider.

**Response:**
```json
{
  "success": true,
  "provider": "nous"
}
```

### `new_session(cwd, mcp_servers?)`

Creates a new Hermes session with the given working directory.

**Parameters:**
- `cwd` — Working directory for file operations
- `mcp_servers` — Optional MCP servers to connect

**Response:**
```json
{
  "session_id": "sess_abc123"
}
```

### `load_session(cwd, session_id)`

Loads an existing session and updates the working directory.

### `resume_session(cwd, session_id?)`

Resumes a session or auto-creates if not found.

### `fork_session(cwd, session_id)`

Duplicates session state for branching conversations.

### `list_sessions(cursor?, cwd?)`

Paginated session listing. Returns session metadata with titles and timestamps.

### `cancel(session_id)`

Interrupts a running agent turn.

### `prompt(session_id, prompt, include_context?)`

Sends a user prompt and returns the full response with streaming events.

**Content block types supported:**
- `TextContentBlock` — plain text
- `ImageContentBlock` — image data (base64)
- `AudioContentBlock` — audio data
- `ResourceContentBlock` — external URLs
- `EmbeddedResourceContentBlock` — embedded file content

---

## Streaming Events

The ACP server emits `session_update` events during agent execution:

| Event Type | Description |
|------------|-------------|
| `thinking` | Extended thinking block (Claude extended thinking) |
| `tool_start` | Tool invocation beginning |
| `tool_end` | Tool invocation complete with result |
| `step` | Agent turn completed |
| `message` | Text content generated |

---

## ACP Toolset

When running via ACP, Hermes uses the `hermes-acp` toolset, which excludes:

- Audio/TTS tools (no speaker in editor context)
- `clarify` tool (no interactive prompts in editor)

All other tools are available: web, terminal, file operations, browser, vision, skills, memory, etc.

---

## Authentication in ACP

The ACP adapter detects the configured runtime provider automatically:

1. Checks for `OPENROUTER_API_KEY` → uses OpenRouter
2. Checks for `GLM_API_KEY` → uses z.ai
3. Checks for `KIMI_API_KEY` → uses Kimi
4. Checks for `MINIMAX_API_KEY` → uses MiniMax
5. Falls back to Nous Portal if none found

The `authenticate()` method validates the detected provider and returns its name.

---

## Parallel Sessions

The ACP server uses a `ThreadPoolExecutor` with 4 workers, allowing up to 4 simultaneous sessions. Each session runs an independent AIAgent with its own conversation state and tool execution.

---

## MCP Servers via ACP

Pass MCP server configurations when creating a session:

```json
{
  "method": "new_session",
  "params": {
    "cwd": "/home/user/project",
    "mcp_servers": [
      {
        "name": "filesystem",
        "transport": "stdio",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
      }
    ]
  }
}
```

MCP tools appear as `mcp_<server>_<tool>` in the agent's tool list.
