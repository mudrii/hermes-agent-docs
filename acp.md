# ACP Editor Integration

Hermes Agent can run as an ACP (Agent Communication Protocol) server, letting ACP-compatible editors communicate with Hermes over stdio and render chat messages, tool activity, file diffs, terminal commands, approval prompts, and streamed response chunks directly in the editor UI.

ACP is the right choice when you want Hermes to behave like an editor-native coding agent rather than a standalone CLI tool or messaging bot.

Released v0.7.0 also lets ACP clients contribute their own MCP servers. Hermes can ingest those editor-provided MCP endpoints and expose them as additional tools inside the ACP-backed session.

---

## What is ACP?

ACP (Agent Communication Protocol) is a JSON-RPC protocol over stdio that allows editors and IDEs to communicate with AI agents in a structured way. The protocol defines:

- Session lifecycle management (new, load, resume, fork, cancel)
- Prompt execution with streaming response chunks
- Tool call activity notifications
- File diff rendering
- Approval prompts for dangerous commands
- Model switching per-session
- Session listing and forking

Hermes implements an ACP server in the `acp_adapter/` package. The adapter wraps Hermes' synchronous `AIAgent` in an async JSON-RPC stdio server using `asyncio.run_coroutine_threadsafe()` to bridge the synchronous agent thread with the async ACP I/O loop.

### Key implementation files

| File | Purpose |
|------|---------|
| `acp_adapter/entry.py` | Boot entrypoint, env loading, server startup |
| `acp_adapter/server.py` | `HermesACPAgent` -- implements the ACP agent protocol |
| `acp_adapter/session.py` | `SessionManager` -- tracks live ACP sessions |
| `acp_adapter/events.py` | Event bridge: AIAgent callbacks to ACP `session_update` events |
| `acp_adapter/permissions.py` | Permission bridge: dangerous command approval to ACP permission requests |
| `acp_adapter/tools.py` | Tool rendering helpers -- maps Hermes tools to ACP tool kinds and content |
| `acp_adapter/auth.py` | Provider/auth resolution for ACP |
| `acp_registry/agent.json` | ACP registry manifest |

---

## Supported Editors

| Editor | How to Connect |
|--------|---------------|
| **VS Code** | Install an ACP client extension, point it at `acp_registry/` directory |
| **Zed** | Add `acp.agents` to settings, point `registry_dir` at `acp_registry/` |
| **JetBrains** | Use an ACP-compatible plugin, point it at `acp_registry/` directory |

---

## Installation

Install Hermes normally, then add the `acp` extra:

```bash
uv pip install -e '.[acp]'
```

This installs `agent-client-protocol>=0.8.1,<1.0` and enables three entry points:

- `hermes acp` -- CLI subcommand
- `hermes-acp` -- standalone binary (defined as `acp_adapter.entry:main` in pyproject.toml)
- `python -m acp_adapter` -- module invocation (via `acp_adapter/__main__.py`)

---

## Launching the ACP Server

Any of the following starts Hermes in ACP mode:

```bash
hermes acp
```

```bash
hermes-acp
```

```bash
python -m acp_adapter
```

All three are equivalent. Hermes logs to stderr so stdout remains reserved for ACP JSON-RPC traffic.

### Boot flow

```
hermes acp / hermes-acp / python -m acp_adapter
  -> acp_adapter.entry.main()
  -> load ~/.hermes/.env (via hermes_cli.env_loader.load_hermes_dotenv)
  -> configure stderr logging (quiet httpx, httpcore, openai)
  -> ensure project root is on sys.path
  -> construct HermesACPAgent
  -> acp.run_agent(agent)
```

---

## Editor Setup

### VS Code

Install an ACP client extension, then point it at the repo's `acp_registry/` directory in your VS Code settings:

```json
{
  "acpClient.agents": [
    {
      "name": "hermes-agent",
      "registryDir": "/path/to/hermes-agent/acp_registry"
    }
  ]
}
```

Replace `/path/to/hermes-agent` with the absolute path to your cloned hermes-agent repository.

### Zed

Add an `acp` block to your Zed settings file:

```json
{
  "acp": {
    "agents": [
      {
        "name": "hermes-agent",
        "registry_dir": "/path/to/hermes-agent/acp_registry"
      }
    ]
  }
}
```

### JetBrains

Use an ACP-compatible plugin and configure it to point at:

```
/path/to/hermes-agent/acp_registry
```

---

## Registry Manifest

The ACP registry manifest is at `acp_registry/agent.json`:

```json
{
  "schema_version": 1,
  "name": "hermes-agent",
  "display_name": "Hermes Agent",
  "description": "AI agent by Nous Research with 90+ tools, persistent memory, and multi-platform support",
  "icon": "icon.svg",
  "distribution": {
    "type": "command",
    "command": "hermes",
    "args": ["acp"]
  }
}
```

This advertises a command-based agent whose launch command is `hermes acp`. The editor reads this manifest to discover how to start the server.

---

## What Hermes Exposes in ACP Mode

Hermes runs with a curated `hermes-acp` toolset designed for editor workflows. It includes:

- File tools: `read_file`, `write_file`, `patch`, `search_files`
- Terminal tools: `terminal`, `process`
- Web and browser tools
- Memory, todo, session search
- Skills
- `execute_code` and `delegate_task`
- Vision tools

The ACP toolset intentionally excludes features that do not fit typical editor UX:

- Messaging delivery tools
- Cron job management
- Gateway-specific tools

### Client-provided MCP servers (v0.7.0)

ACP editor integrations can now attach MCP server definitions to the ACP session. Hermes ingests those client-provided servers and merges their tools into the active ACP tool surface, so editor-local MCP ecosystems can flow directly into Hermes without separate global MCP configuration.

This is additive to normal Hermes MCP config:

- `config.yaml` MCP servers still load normally
- ACP-provided MCP servers are session-scoped to the editor connection
- tool naming and MCP safety rules still follow the normal Hermes MCP path

---

## Configuration and Credentials

ACP mode uses the same Hermes configuration as the CLI:

| Path | Purpose |
|------|---------|
| `~/.hermes/.env` | API keys and secrets |
| `~/.hermes/config.yaml` | Provider, model, toolset, and agent settings |
| `~/.hermes/skills/` | Active skills |
| `~/.hermes/state.db` | Session and conversation storage |

Provider resolution uses Hermes' normal runtime resolver, so ACP inherits the currently configured provider and credentials. ACP mode does not have its own authentication flow -- configure credentials first with `hermes model` or by editing `~/.hermes/.env`.

---

## ACP Copilot Auth

Hermes supports using GitHub Copilot as a provider in ACP mode via the `copilot-acp` provider.

### copilot-acp Provider

The `copilot-acp` provider (`auth_type: "external_process"`) routes through an external process that provides the Copilot API endpoint. The base URL is resolved from `COPILOT_ACP_BASE_URL` environment variable, falling back to `acp://copilot`.

### copilot Provider (Direct API)

The direct `copilot` provider authenticates via GitHub token with this search order:

1. `COPILOT_GITHUB_TOKEN` environment variable
2. `GH_TOKEN` environment variable
3. `GITHUB_TOKEN` environment variable
4. `gh auth token` (GitHub CLI fallback)

Supported token types: OAuth tokens (`gho_`), fine-grained PATs (`github_pat_`), GitHub App tokens (`ghu_`). Classic PATs (`ghp_`) are explicitly rejected.

The device code login flow for Copilot (`copilot_device_code_login` in `hermes_cli/copilot_auth.py`) uses:

- Client ID: `Ov23li8tweQw6odWQebz` (same client ID as opencode and the Copilot CLI)
- Device code URL: `https://github.com/login/device/code`
- Scope: `read:user`
- Polling interval: minimum 5 seconds with RFC 8628 `slow_down` error handling

---

## ACP Internals: Protocol Details

### HermesACPAgent

`acp_adapter/server.py` implements the ACP agent protocol via the `HermesACPAgent` class, which extends `acp.Agent`.

Responsibilities:
- `initialize()` -- returns protocol version, agent info (`hermes-agent` + version), and capabilities (session fork/list support). Advertises authentication methods based on the detected provider.
- `authenticate()` -- validates that a provider is configured
- Session methods: `new_session`, `load_session`, `resume_session`, `fork_session`, `list_sessions`, `cancel`
- `prompt()` -- runs the agent on user input and streams events back to the editor
- `set_session_model()` -- switch model for a session (ACP protocol method)
- Slash commands: `/help`, `/model`, `/tools`, `/context`, `/reset`, `/compact`, `/version`

The agent uses a `ThreadPoolExecutor` with 4 worker threads (`thread_name_prefix="acp-agent"`) to run the synchronous `AIAgent` in parallel.

### SessionManager

`acp_adapter/session.py` tracks live ACP sessions. Each session stores:

| Field | Description |
|-------|-------------|
| `session_id` | Unique session identifier |
| `agent` | The underlying `AIAgent` instance |
| `cwd` | Working directory bound to this session |
| `model` | Selected model for this session |
| `history` | Current conversation message list |
| `cancel_event` | Threading event for cancellation |

The manager is thread-safe and supports: create, get, remove, fork, list, cleanup, and cwd updates.

### Event Bridge

`acp_adapter/events.py` converts AIAgent callbacks into ACP `session_update` events.

Bridged callbacks:
- `tool_progress_callback`
- `thinking_callback`
- `step_callback`
- `message_callback`

Because `AIAgent` runs in a worker thread while ACP I/O lives on the main event loop, the bridge uses `asyncio.run_coroutine_threadsafe()` to safely dispatch updates.

The event bridge tracks tool IDs with FIFO queues per tool name (not a single ID per name). This handles parallel same-name calls and repeated same-name calls in one step without attaching completion events to the wrong tool invocation.

### Permission Bridge

`acp_adapter/permissions.py` adapts dangerous terminal approval prompts into ACP permission requests.

Mapping:
- `allow_once` -> Hermes `once`
- `allow_always` -> Hermes `always`
- Reject options -> Hermes `deny`

Timeouts and bridge failures deny by default.

The approval callback is temporarily installed on the terminal tool during prompt execution and restored afterward. This prevents session-specific approval handlers from persisting globally.

### Tool Rendering Helpers

`acp_adapter/tools.py` maps Hermes tools to ACP tool kinds and builds editor-facing content:

| Hermes Tool | ACP Rendering |
|-------------|---------------|
| `patch`, `write_file` | File diffs |
| `terminal` | Shell command text |
| `read_file`, `search_files` | Text previews |
| Large results | Truncated text blocks (for UI safety) |

### Provider and Auth

ACP does not implement its own auth store. It reuses Hermes' runtime resolver from `acp_adapter/auth.py`, so ACP advertises and uses the currently configured Hermes provider and credentials.

---

## Session Lifecycle

```
new_session(cwd)
  -> create SessionState
  -> create AIAgent(platform="acp", enabled_toolsets=["hermes-acp"])
  -> bind task_id/session_id to cwd override

prompt(..., session_id)
  -> extract text from ACP content blocks
  -> intercept slash commands (return immediately if recognized)
  -> reset cancel event
  -> install callbacks + approval bridge
  -> run AIAgent in ThreadPoolExecutor
  -> update session history
  -> emit final agent message chunk
  -> return PromptResponse with usage stats and stop_reason
```

### Cancellation

`cancel(session_id)`:
- Sets the session cancel event
- Calls `agent.interrupt()` when available
- Causes the prompt response to return `stop_reason="cancelled"`

### Forking

`fork_session()` deep-copies message history into a new live session, preserving conversation state while giving the fork its own session ID and cwd.

### Working Directory Binding

ACP sessions bind the editor's cwd to the Hermes task ID. File and terminal tools operate relative to the editor workspace, not the server process cwd.

### Session Scope

ACP sessions are tracked in-memory while the server is running. The underlying `AIAgent` still uses Hermes' normal persistence and logging paths (`~/.hermes/state.db`, `~/.hermes/sessions/`), but ACP `list/load/resume/fork` are scoped to the currently running ACP server process.

### Slash Commands

ACP mode supports these headless slash commands that execute locally without calling the LLM:

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/model [name]` | Show or change current model (with auto-provider detection) |
| `/tools` | List available tools |
| `/context` | Show conversation context info (message count by role) |
| `/reset` | Clear conversation history |
| `/compact` | Compress conversation context |
| `/version` | Show Hermes version |

Unrecognized `/commands` are sent to the model as normal messages.

---

## Current Limitations

- ACP sessions are process-local from the ACP server's perspective
- Non-text prompt blocks (images, audio, resources) are currently ignored for request text extraction
- Editor-specific UX varies by ACP client implementation

---

## Troubleshooting ACP Connection Issues

### ACP agent does not appear in the editor

Check:
- The editor is pointed at the correct `acp_registry/` path (must be an absolute path)
- Hermes is installed and on your PATH: `which hermes`
- The ACP extra is installed: `uv pip install -e '.[acp]'`
- Run `hermes doctor` to verify the installation is healthy

### ACP starts but immediately errors

```bash
hermes doctor    # Check general health
hermes status    # Check configuration
hermes acp       # Start manually to see stderr error output
```

Errors from the ACP server appear on stderr, not stdout. Your terminal shows stderr; the editor only sees stdout (JSON-RPC traffic).

### Missing credentials

ACP mode does not have its own login flow. Configure credentials before starting ACP:

```bash
hermes model     # Interactive provider and credential setup
```

Or edit `~/.hermes/.env` directly to add API keys.

### ACP agent appears but does not respond

- Ensure no other process is running on the same ACP server instance
- Check that the `hermes-acp` binary is in PATH: `which hermes-acp`
- Verify provider credentials are valid: `hermes chat -q "hello"`

### Tool outputs not appearing in editor

Different ACP client implementations render tool activity differently. File diffs, terminal output, and text previews are passed as structured content blocks. If your editor's ACP plugin does not render them, check the plugin's documentation for supported content types.

## What's New

### v0.4.0

No changes to the ACP adapter in v0.4.0.

### v0.5.0

- **Plugin lifecycle hooks activated** (PR #3542) — `pre_llm_call`, `post_llm_call`, `on_session_start`, and `on_session_end` hooks now fire in the ACP agent loop, enabling plugin authors to hook into ACP sessions.
- **Fix plugin toolsets invisible in standalone processes** (PR #3457) — Plugin-registered toolsets are now visible in `hermes tools` and in the ACP server process.

### v0.7.0

- **Client-provided MCP servers** — ACP editor integrations can attach MCP server definitions to the ACP session. Hermes ingests those client-provided servers and merges their tools into the active ACP tool surface. Session-scoped; additive to `config.yaml` MCP servers.

### v0.8.0

- **Aggregate ACP improvements** (PR #5292) — auth compatibility fixes, protocol fixes, command advertisements, delegation support, and SSE event handling.
- **Plugin session lifecycle hooks** (PR #6129) — `on_session_finalize` and `on_session_reset` now fire in ACP sessions.
