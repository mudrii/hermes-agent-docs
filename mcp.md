# MCP (Model Context Protocol)

MCP lets Hermes Agent connect to external tool servers so the agent can use tools that live outside Hermes itself — GitHub, databases, file systems, browser stacks, internal APIs, and more.

## What MCP Gives You

- Access to external tool ecosystems without writing a native Hermes tool
- Local stdio servers and remote HTTP MCP servers in the same config
- Automatic tool discovery and registration at startup
- Utility wrappers for MCP resources and prompts when supported by the server
- Per-server tool filtering to expose only the tools you actually want Hermes to see
- MCP sampling: server-initiated LLM requests routed through Hermes

## Supported Transports

### Stdio transport

Stdio servers run as local subprocesses and communicate over stdin/stdout.

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
```

Use stdio servers when:
- the server is installed locally
- you want low-latency access to local resources
- the MCP server docs show `command`, `args`, and `env` fields

The `mcp` Python package must be installed (`uv pip install -e ".[mcp]"` in `~/.hermes/hermes-agent`). For npm-based servers, `npx` must be on PATH.

### HTTP (StreamableHTTP) transport

HTTP MCP servers are remote endpoints Hermes connects to directly using the StreamableHTTP transport.

```yaml
mcp_servers:
  remote_api:
    url: "https://mcp.example.com/mcp"
    headers:
      Authorization: "Bearer sk-..."
```

Use HTTP servers when:
- the MCP server is hosted elsewhere
- your organization exposes internal MCP endpoints
- you do not want Hermes spawning a local subprocess for that integration

If both `url` and `command` are present in one server config, HTTP transport (`url`) wins and Hermes logs a warning.

## How to Configure MCP Servers

MCP configuration lives in `~/.hermes/config.yaml` under the `mcp_servers` key:

```yaml
mcp_servers:
  <server_name>:
    command: "..."      # stdio servers: executable to launch
    args: []            # stdio servers: arguments for the subprocess
    env: {}             # stdio servers: environment variables

    # OR for HTTP servers:
    url: "..."          # HTTP MCP endpoint
    headers: {}         # HTTP headers

    enabled: true       # Set to false to skip entirely
    timeout: 120        # Tool call timeout in seconds (default: 120)
    connect_timeout: 60 # Initial connection timeout in seconds (default: 60)
    tools:
      include: []       # Whitelist: only these tools are registered
      exclude: []       # Blacklist: all tools except these
      resources: true   # Register list_resources/read_resource utilities
      prompts: true     # Register list_prompts/get_prompt utilities

    auth: oauth         # Enable OAuth 2.1 PKCE (HTTP servers only)
    oauth:              # Optional pre-registered client (overrides DCR)
      client_id: "..."
      client_secret: "..."   # confidential client only
      scope: "read write"    # overrides server-advertised scope

    sampling:           # Optional: server-initiated LLM requests
      enabled: true     # Default: true
      model: "..."      # Override model for sampling calls
      max_tokens_cap: 4096
      timeout: 30
      max_rpm: 10
      allowed_models: []
      max_tool_rounds: 5
      log_level: "info"
```

### Server key reference

| Key | Type | Applies to | Description |
|-----|------|------------|-------------|
| `command` | string | stdio | Executable to launch |
| `args` | list | stdio | Arguments for the subprocess |
| `env` | mapping | stdio | Environment variables passed to the subprocess |
| `url` | string | HTTP | Remote MCP endpoint URL |
| `headers` | mapping | HTTP | HTTP headers for remote server requests |
| `enabled` | bool | both | If `false`, Hermes skips the server entirely |
| `timeout` | number | both | Per-tool-call timeout in seconds |
| `connect_timeout` | number | both | Initial connection timeout in seconds |
| `tools` | mapping | both | Tool filtering and utility policy |
| `auth` | string | HTTP | Set to `oauth` to enable OAuth 2.1 PKCE authentication |
| `oauth` | mapping | HTTP | Optional pre-registered client credentials for servers that do not support Dynamic Client Registration |

### Tools policy keys

| Key | Type | Description |
|-----|------|-------------|
| `tools.include` | string or list | Whitelist: only these server-native MCP tools are registered |
| `tools.exclude` | string or list | Blacklist: all server-native MCP tools except these are registered |
| `tools.resources` | bool | Enable/disable `list_resources` and `read_resource` utility wrappers |
| `tools.prompts` | bool | Enable/disable `list_prompts` and `get_prompt` utility wrappers |

## Tool Naming Convention

Hermes prefixes MCP tools to avoid name collisions with built-in tools:

```
mcp_<server_name>_<tool_name>
```

Hyphens and dots in server names and tool names are replaced with underscores.

| Server | MCP tool name | Registered name in Hermes |
|--------|---------------|--------------------------|
| `filesystem` | `read_file` | `mcp_filesystem_read_file` |
| `github` | `create-issue` | `mcp_github_create_issue` |
| `my-api` | `query.data` | `mcp_my_api_query_data` |

You do not need to call the prefixed name manually — Hermes sees the tool and chooses it during normal reasoning.

## MCP Utility Tools

When an MCP server supports resources or prompts, Hermes registers utility wrappers per server:

**Resource utilities:**
- `mcp_<server>_list_resources` — list available resources
- `mcp_<server>_read_resource` — read a resource by URI

**Prompt utilities:**
- `mcp_<server>_list_prompts` — list available prompts
- `mcp_<server>_get_prompt` — get a prompt by name with optional arguments

These are capability-aware: Hermes only registers a resource utility if the MCP session actually supports resource operations, and only registers a prompt utility if the session supports prompt operations. A server that exposes callable tools but no resources/prompts will not get those extra wrappers.

## Per-Server Tool Filtering

### Disable a server entirely

```yaml
mcp_servers:
  legacy:
    url: "https://mcp.legacy.internal"
    enabled: false
```

When `enabled: false`, Hermes makes no connection attempt, performs no discovery, and registers no tools. The config remains in place for later reuse.

### Whitelist server tools (include)

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
    tools:
      include: [create_issue, list_issues, search_code]
```

Only those listed MCP server tools are registered. All others are silently dropped.

### Blacklist server tools (exclude)

```yaml
mcp_servers:
  stripe:
    url: "https://mcp.stripe.com"
    headers:
      Authorization: "Bearer ***"
    tools:
      exclude: [delete_customer, refund_payment]
```

All server tools are registered except the excluded ones.

### Precedence rule

If both `include` and `exclude` are present, `include` wins:

```yaml
tools:
  include: [create_issue]
  exclude: [create_issue, delete_issue]
```

Result: `create_issue` is still registered; `exclude` has no effect on it.

### Disable utility wrappers

```yaml
mcp_servers:
  docs:
    url: "https://mcp.docs.example.com"
    tools:
      prompts: false
      resources: false
```

`tools.resources: false` disables `list_resources` and `read_resource`. `tools.prompts: false` disables `list_prompts` and `get_prompt`.

### Empty result behavior

If your config filters out all callable tools and disables or omits all supported utilities, Hermes does not create a runtime MCP toolset for that server. This keeps the tool list clean.

## MCP Sampling

Sampling lets MCP servers request LLM completions from Hermes. This allows MCP servers to drive agentic workflows — the server sends a request, Hermes calls the configured LLM, and the result is returned to the server.

Configure sampling per server:

```yaml
mcp_servers:
  analysis:
    command: "npx"
    args: ["-y", "analysis-server"]
    sampling:
      enabled: true              # Default: true when MCP sampling types are available
      model: "gemini-3-flash"    # Override which model handles sampling requests
      max_tokens_cap: 4096       # Maximum tokens per sampling request
      timeout: 30                # LLM call timeout in seconds
      max_rpm: 10                # Rate limit: maximum requests per minute
      allowed_models: []         # Whitelist of allowed model names (empty = all)
      max_tool_rounds: 5         # Tool loop limit (0 = disable tool loops)
      log_level: "info"          # Audit verbosity: debug, info, warning
```

Sampling requires the MCP SDK to expose sampling types (`CreateMessageResult`, `SamplingCapability`). If those types are unavailable in the installed SDK version, sampling is disabled with a debug log message.

The `SamplingHandler` class in `tools/mcp_tool.py` manages per-server sampling state:
- Sliding-window rate limiting (60-second window)
- Model resolution: config override > server hint > default
- Token and request metrics per server
- Tool loop governance via `max_tool_rounds`
- Credential scrubbing from error messages before returning to the LLM

## Reconnection Behavior

Hermes implements automatic reconnection with exponential backoff for dropped connections:

- Maximum reconnection attempts: 5
- Initial backoff: 1 second
- Backoff multiplier: 2x per retry
- Maximum backoff: 60 seconds

If the initial connection (first attempt) fails, Hermes logs a warning and continues without that server. If the connection drops after being established, Hermes attempts to reconnect up to 5 times before giving up.

During reconnection, the server's tools remain registered in the tool list but calls to them return `{"error": "MCP server '<name>' is not connected"}` until reconnection succeeds.

Reconnection is skipped if a graceful shutdown has been requested.

## Architecture

Hermes runs a dedicated background event loop (`mcp-event-loop` daemon thread) for all MCP connections. Each MCP server runs as a long-lived `asyncio.Task` on this loop, keeping its transport context alive.

This architecture is required by `anyio`: cancel-scope cleanup must happen in the same Task that opened the connection.

Tool call coroutines are scheduled onto the background loop via `asyncio.run_coroutine_threadsafe()` and blocked on from the caller thread. The lock `_lock` protects the `_servers` dict and the loop/thread references from concurrent access.

On shutdown, each server Task is signalled to exit its `async with` block. All servers are shut down in parallel via `asyncio.gather`.

## OAuth 2.1 Authentication (v0.8.0)

For HTTP MCP servers that require OAuth, add `auth: oauth` to the server entry:

```yaml
mcp_servers:
  sentry:
    url: "https://mcp.sentry.dev/mcp"
    auth: oauth
```

When Hermes connects to this server, it runs a full OAuth 2.1 PKCE flow:

1. Discovers the authorization server metadata from the MCP server's well-known endpoint
2. Attempts Dynamic Client Registration (DCR) to obtain a client ID automatically
3. Opens the browser to the authorization URL with a PKCE challenge
4. Starts a local callback server to receive the authorization code
5. Exchanges the code for access and refresh tokens using the PKCE verifier
6. Persists tokens to `~/.hermes/mcp-tokens/<server>.json` (permissions: `0o600`)
7. Reuses cached tokens on subsequent connections; refreshes automatically
8. Re-authorizes only when refresh fails

For servers that do not support DCR (such as Slack), supply pre-registered credentials:

```yaml
mcp_servers:
  slack:
    url: "https://mcp.slack.com/sse"
    auth: oauth
    oauth:
      client_id: "..."
      client_secret: "..."    # confidential client
      scope: "channels:read chat:write"
```

Non-interactive environments (gateway, cron) detect when no cached tokens are present and log a warning rather than blocking on a browser prompt. Run an interactive `hermes mcp add` or `hermes chat` session first to authorize, then the gateway can reuse the cached tokens.

## Excluding MCP Servers Per Platform (`no_mcp` sentinel) (v0.8.0)

By default, all enabled MCP servers are injected into every platform. To exclude all MCP servers from a specific platform, add `no_mcp` to its `platform_toolsets` list:

```yaml
platform_toolsets:
  api_server:
    - terminal
    - file
    - web
    - no_mcp    # exclude all MCP servers from this platform
```

The `no_mcp` value is a sentinel — it is filtered out of the actual toolset list and does not appear as a toolset name. Other platforms are unaffected. This is useful for the API server platform to avoid inflating token usage with full MCP tool schemas.

## Security Model

### stdio environment filtering

For stdio servers, Hermes does not pass your full shell environment to the subprocess. Only a safe baseline is passed:
- `PATH`, `HOME`, `USER`, `LANG`, `LC_ALL`, `TERM`, `SHELL`, `TMPDIR`
- All `XDG_*` variables
- Variables explicitly listed in the server's `env` config block

This prevents accidental leakage of API keys, tokens, and credentials to MCP server subprocesses.

### Credential scrubbing in error messages

Error messages returned from MCP tool calls are scrubbed for credential-like patterns before the LLM sees them. Scrubbed patterns include:
- GitHub PATs (`ghp_...`)
- OpenAI-style keys (`sk-...`)
- Bearer tokens
- `token=`, `key=`, `API_KEY=`, `password=`, `secret=` patterns

### Config-level exposure control

Per-server filtering is also a security control:
- Use `tools.include` to expose a minimal surface for sensitive servers
- Use `tools.exclude` to block dangerous tools you do not want the model to call
- Use `tools.resources: false` and `tools.prompts: false` to prevent browsing server-provided assets

## Runtime Behavior

### Discovery time

Hermes connects to configured MCP servers at startup, in parallel (`asyncio.gather`). The outer discovery timeout is 120 seconds total. Per-server connection timeouts are controlled by `connect_timeout` (default: 60 seconds).

After discovery, each connected server's tools are injected into the normal tool registry and into all `hermes-*` platform toolsets.

### Reloading

If you change MCP config during a session, use the `/reload-mcp` slash command:

```
/reload-mcp
```

This reloads MCP servers from config and refreshes the available tool list without restarting Hermes.

### Toolset created per server

Each configured MCP server that contributes at least one registered tool also gets its own named toolset: `mcp-<server>`. This lets you target a specific MCP server in platform toolset overrides.

## Complete Config Example

```yaml
mcp_servers:
  # Stdio server: GitHub with a tight allowlist
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
    tools:
      include: [list_issues, create_issue, update_issue, search_code]
      prompts: false
      resources: false

  # Stdio server: Git access for one repository
  git:
    command: "uvx"
    args: ["mcp-server-git", "--repository", "/home/user/project"]

  # Stdio server: Filesystem access rooted to one directory
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/project"]
    connect_timeout: 30

  # HTTP server: Stripe with dangerous actions removed
  stripe:
    url: "https://mcp.stripe.com"
    headers:
      Authorization: "Bearer rk_live_..."
    tools:
      exclude: [delete_customer, refund_payment]
      resources: false

  # HTTP server: Internal API with strict allowlist
  internal_api:
    url: "https://mcp.internal.example.com"
    headers:
      Authorization: "Bearer ***"
    tools:
      include: [list_customers, get_customer, list_invoices]
      resources: false
      prompts: false

  # HTTP server: Documentation server with resources enabled
  docs:
    url: "https://mcp.docs.example.com"
    tools:
      prompts: true
      resources: true

  # Disabled server: config kept for later use
  legacy:
    url: "https://mcp.legacy.internal"
    enabled: false

  # Server with sampling enabled
  analysis:
    command: "npx"
    args: ["-y", "analysis-server"]
    sampling:
      enabled: true
      model: "gemini-3-flash"
      max_tokens_cap: 4096
      timeout: 30
      max_rpm: 10
      max_tool_rounds: 5
```

## hermes mcp CLI

The `hermes mcp` subcommand group (v0.4.0, PR #2465) provides interactive MCP server lifecycle management directly from the command line.

### Subcommands

```bash
hermes mcp serve                              # Run Hermes as an MCP server for editors
hermes mcp add <name> --url <endpoint>        # Add an HTTP MCP server
hermes mcp add <name> --command <cmd>         # Add a stdio MCP server
hermes mcp remove <name>                      # Remove a server and clean up tokens
hermes mcp list                               # List all configured servers
hermes mcp test <name>                        # Test connection and list available tools
hermes mcp configure <name>                   # Toggle individual tools on/off
```

### hermes mcp add

Discovery-first: connects to the server, lists its tools, and lets you select which ones to register.

```bash
# Add an HTTP server
hermes mcp add stripe --url "https://mcp.stripe.com"

# Add with OAuth 2.1 auth
hermes mcp add ink --url "https://mcp.ml.ink/mcp" --auth oauth

# Add a stdio server
hermes mcp add github --command npx --args @modelcontextprotocol/server-github
```

When `--auth oauth` is specified, `hermes mcp add` initiates the OAuth 2.1 PKCE flow:

1. Discovers the authorization endpoint from the MCP server's OAuth metadata
2. Generates a PKCE code verifier and challenge
3. Opens the browser to the authorization URL
4. Starts a local callback server to receive the authorization code
5. Exchanges the code for tokens (access + refresh)
6. Stores tokens to `~/.hermes/mcp-tokens/<name>.json` (0o600 permissions) and reuses them automatically across sessions.

**Note (v0.5.0):** The Hermes-native PKCE OAuth implementation was removed in v0.5.0 (PR #3107). MCP servers that require OAuth must use the standard OAuth 2.1 flow via `--auth oauth`. The flow above uses the standard MCP SDK OAuth support, not a Hermes-specific implementation.

### hermes mcp remove

Removes the server from `config.yaml` and deletes any stored OAuth tokens.

### hermes mcp test

Temporarily connects to the server and lists all available tools with their descriptions. Useful for verifying configuration before starting the gateway.

```bash
hermes mcp test github
# Auth: OAuth 2.1 PKCE
# Found 8 tools: create_issue, list_issues, ...
```

### hermes mcp serve

Starts Hermes as a stdio MCP server — exposes Hermes' messaging platform conversations as MCP tools so that MCP clients (Claude Code, Cursor, Codex) can read messages, send messages, and manage approvals across all connected gateway platforms.

```bash
hermes mcp serve
hermes mcp serve --verbose
```

MCP client config (e.g. `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

## What's New

### v0.4.0 (PR #2465)

- `hermes mcp` subcommand group introduced
- `add`, `remove`, `list`, `test`, and `configure` subcommands
- OAuth 2.1 PKCE flow for HTTP MCP servers that require authentication
- Interactive tool selection during `hermes mcp add`
- Expose MCP servers as standalone named toolsets (`mcp-<name>`) (PR #1907)
- Interactive MCP tool configuration in `hermes tools` (PR #1694)

### v0.5.0 (PR #3107, #3252, #3077)

- **Removed Hermes-native PKCE OAuth** (PR #3107) — The custom Hermes PKCE implementation was removed. MCP OAuth now delegates entirely to the standard MCP SDK OAuth 2.1 flow.
- **MCP toolset resolution for runtime and config** (PR #3252) — MCP toolsets are now properly resolved in both live sessions and static config.
- **MCP tool name collision protection** (PR #3077) — Duplicate tool names across MCP servers are now detected and handled gracefully.

### v0.6.0 (PR #3795, #3812, #3646)

- **MCP Server Mode** (PR #3795) — `hermes mcp serve` now exposes the full 10-tool conversation bridge surface. See [MCP Server Mode (v0.6.0)](#mcp-server-mode-v060) below.
- **Dynamic tool discovery** (PR #3812) — when Hermes is acting as an MCP client, it responds to `notifications/tools/list_changed` events and picks up new tools from connected servers without reconnecting.
- **Non-deprecated HTTP transport** (PR #3646) — Hermes' MCP client switched from `sse_client` to `streamable_http_client` for HTTP connections to external MCP servers.

### v0.8.0 (PR #5420, #5305, #5979)

- **Full OAuth 2.1 PKCE client** (PR #5420) — `tools/mcp_oauth.py` implements a complete OAuth 2.1 PKCE flow using the MCP SDK's `OAuthClientProvider`. Includes metadata discovery, Dynamic Client Registration (DCR), PKCE code exchange, token refresh, and step-up re-auth on `403 insufficient_scope`. Tokens are persisted to `~/.hermes/mcp-tokens/<server>.json` with `0o600` permissions and reused automatically across sessions. Pre-registered client credentials (`oauth.client_id` / `oauth.client_secret`) are supported for servers that do not allow DCR (e.g. Slack). Non-interactive environments (gateway, cron) detect when cached tokens are absent and warn rather than hanging on a browser prompt.
- **OSV malware scanning** (PR #5305) — before spawning an MCP stdio server via `npx` or `uvx`, Hermes queries the OSV API to check the package for known malware advisories (MAL-* IDs). Only confirmed malware is blocked; regular CVEs are ignored. The check is fail-open: network errors, timeouts, and unrecognized package managers allow the launch to proceed. Runs in parallel with other MCP server startup (no serial delay).
- **`no_mcp` sentinel** (PR #5979) — add `no_mcp` to a platform's `platform_toolsets` list to exclude all MCP servers from that platform. Useful for the API server platform to avoid inflating token usage with full MCP schemas. Other platforms are unaffected.
- **`structuredContent` preservation** (PR #5979) — MCP tool call results now correctly read the camelCase `structuredContent` attribute from `CallToolResult`. When present, the structured payload is returned as the result (preferred over the plain-text `content` field). This was previously a silent no-op due to an attribute name mismatch.

## Quickstart

1. Install MCP support (included in the standard install, but can be added separately):

```bash
cd ~/.hermes/hermes-agent
uv pip install -e ".[mcp]"
```

2. For npm-based servers, ensure Node.js and npx are available:

```bash
node --version
npx --version
```

3. Add a server to `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
```

4. Start Hermes:

```bash
hermes chat
```

5. Verify MCP loaded by asking the agent what tools are available, or use:

```
/reload-mcp
```

## Troubleshooting

### MCP server not connecting

Check:

```bash
# Verify MCP deps are installed
cd ~/.hermes/hermes-agent && uv pip install -e ".[mcp]"

# Check Node.js is available for npm-based servers
node --version
npx --version
```

Verify your config and restart Hermes, or use `/reload-mcp` inside chat.

### Tools not appearing after connection

Possible causes:
- Filtered by `tools.include` — only whitelisted tools appear
- Excluded by `tools.exclude`
- Utility wrappers disabled via `resources: false` or `prompts: false`
- Server does not actually support resources/prompts (capability-aware registration)
- Server is disabled with `enabled: false`

If you are intentionally filtering, the missing tools are expected.

### Resource or prompt utilities not appearing

Both conditions must be true for utility wrappers to appear:
1. Your config allows them (`resources: true` / `prompts: true`, which is the default)
2. The MCP server session actually supports the capability

If the server does not advertise the capability, Hermes will not register the wrapper even if your config enables it. This is intentional and keeps the tool list honest.

### Server connects but the wrong tools appear

If more tools appear than expected, add an `include` allowlist. If fewer appear than expected, remove any `include` or `exclude` filters and check whether the server actually exposes those tool names.

### "missing executable 'npx'" in logs

Node.js is not on the filtered PATH used by the MCP subprocess. Either:
- Ensure Node.js bin directory is in the system PATH
- Set the full absolute path in `command`: `command: "/usr/local/bin/npx"`
- Or set `PATH` explicitly in the server's `env` block:

```yaml
mcp_servers:
  github:
    command: "/usr/local/bin/npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
```

### How to disable a server without deleting the config

Set `enabled: false`:

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
    enabled: false
```

This prevents the connection and tool registration while preserving the config for later reuse.

### Reloading after config changes

After changing include/exclude lists, enabled flags, resources/prompts toggles, or auth headers/env, reload servers:

```
/reload-mcp
```

## Common Usage Patterns

### Local project assistant

```yaml
mcp_servers:
  fs:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/project"]

  git:
    command: "uvx"
    args: ["mcp-server-git", "--repository", "/home/user/project"]
```

Prompts:
- "Review the project structure and identify where configuration lives."
- "Check the local git state and summarize what changed recently."

### GitHub issue triage assistant

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."
    tools:
      include: [list_issues, create_issue, update_issue, search_code]
      prompts: false
      resources: false
```

Prompts:
- "List open issues labeled bug, cluster them by theme, and draft a high-quality issue for the most common bug."
- "Search the repo for uses of _discover_and_register_server and explain how MCP tools are registered."

### Internal API assistant

```yaml
mcp_servers:
  internal_api:
    url: "https://mcp.internal.example.com"
    headers:
      Authorization: "Bearer ***"
    tools:
      include: [list_customers, get_customer, list_invoices]
      resources: false
      prompts: false
```

Use `include` allowlists for any sensitive internal system — expose only what is needed for the task.

### Documentation and knowledge server

```yaml
mcp_servers:
  docs:
    url: "https://mcp.docs.example.com"
    tools:
      prompts: true
      resources: true
```

Prompts:
- "List available MCP resources from the docs server, then read the onboarding guide and summarize it."
- "List prompts exposed by the docs server and tell me which ones would help with incident response."

## Safe Usage Recommendations

- Prefer `include` allowlists for dangerous systems (financial, customer-facing, destructive). Start with the smallest set possible.
- Keep filesystem servers scoped to one directory, not your home directory.
- Keep git servers pointed at one repository.
- Disable resource and prompt wrappers when you do not want the model browsing server-provided assets.
- Use `enabled: false` to temporarily disable a server without losing the config.
- Reload after config changes with `/reload-mcp` — do not restart Hermes for every small config update.

## MCP Server Mode (v0.6.0)

Hermes itself can act as an MCP server, exposing its messaging platform sessions as tools for any MCP-compatible client (Claude Desktop, Cursor, VS Code, Codex, and others). This lets external agents browse conversations, read message history, send messages, manage approval requests, and poll for live events across all platforms connected to the Hermes gateway. (PR [#3795](https://github.com/NousResearch/hermes-agent/pull/3795))

### Starting the server — stdio transport only

MCP server mode uses **stdio transport only** (not HTTP). The server communicates over stdin/stdout, which is the standard transport for local MCP clients. Despite the HTTP transport being available for Hermes as an MCP *client*, the server mode does not expose an HTTP endpoint.

```bash
hermes mcp serve
hermes mcp serve --verbose   # enable debug logging on stderr
```

### Exposed tools

The server registers ten tools that match OpenClaw's channel bridge surface, plus one Hermes-specific extra:

| Tool | Description |
|------|-------------|
| `conversations_list` | List active conversations across all connected platforms; filter by platform name or search text |
| `conversation_get` | Get detailed info about one conversation by session key |
| `messages_read` | Read recent messages from a conversation in chronological order |
| `attachments_fetch` | List non-text attachments (images, media, file blocks) for a specific message |
| `events_poll` | Poll for new events (message, approval_requested, approval_resolved) since a cursor position |
| `events_wait` | Long-poll — block until the next matching event arrives or a timeout expires |
| `messages_send` | Send a message to a platform target in `platform:chat_id` format |
| `permissions_list_open` | List pending approval requests observed during this bridge session |
| `permissions_respond` | Respond to a pending approval with `allow-once`, `allow-always`, or `deny` |
| `channels_list` | List all available messaging channels and send targets (Hermes-specific) |

### MCP Client Configuration

Add Hermes as an MCP server in any MCP-compatible client. The config format is the same across clients.

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

**Cursor** (`.cursor/mcp.json` in your project or `~/.cursor/mcp.json` globally):

```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

**VS Code / Continue** (`.vscode/mcp.json`):

```json
{
  "mcpServers": {
    "hermes": {
      "command": "hermes",
      "args": ["mcp", "serve"]
    }
  }
}
```

### Dynamic tool discovery

Hermes responds to `notifications/tools/list_changed` events from other MCP servers, picking up newly added tools without requiring a reconnect. (PR [#3812](https://github.com/NousResearch/hermes-agent/pull/3812))

### Requirements

The `mcp` Python package must be installed:

```bash
cd ~/.hermes/hermes-agent
uv pip install -e ".[mcp]"
```
