# MCP Config Reference

This page is the compact reference companion to the main MCP docs.

## Root config shape

```yaml
mcp_servers:
  <server_name>:
    command: "..."      # stdio servers
    args: []
    env: {}

    # OR
    url: "..."          # HTTP servers
    headers: {}

    enabled: true
    timeout: 120
    connect_timeout: 60
    tools:
      include: []
      exclude: []
      resources: true
      prompts: true
```

## Server keys

| Key | Type | Applies to | Meaning |
|-----|------|------------|---------|
| `command` | string | stdio | Executable to launch |
| `args` | list | stdio | Arguments for the subprocess |
| `env` | mapping | stdio | Environment passed to the subprocess |
| `url` | string | HTTP | Remote MCP endpoint |
| `headers` | mapping | HTTP | Headers for remote server requests |
| `enabled` | bool | both | Skip the server entirely when false |
| `timeout` | number | both | Tool call timeout |
| `connect_timeout` | number | both | Initial connection timeout |
| `tools` | mapping | both | Filtering and utility-tool policy |
| `auth` | string | HTTP | Authentication method. Set to `oauth` to enable OAuth 2.1 with PKCE |
| `sampling` | mapping | both | Server-initiated LLM request policy |

## `tools` policy keys

| Key | Type | Meaning |
|-----|------|---------|
| `include` | string or list | Whitelist server-native MCP tools |
| `exclude` | string or list | Blacklist server-native MCP tools |
| `resources` | bool-like | Enable/disable `list_resources` + `read_resource` |
| `prompts` | bool-like | Enable/disable `list_prompts` + `get_prompt` |

## Filtering semantics

### `include`

If `include` is set, only those server-native MCP tools are registered.

```yaml
tools:
  include: [create_issue, list_issues]
```

### `exclude`

If `exclude` is set and `include` is not, every server-native MCP tool except those names is registered.

```yaml
tools:
  exclude: [delete_customer]
```

### Precedence

If both are set, `include` wins.

```yaml
tools:
  include: [create_issue]
  exclude: [create_issue, delete_issue]
```

Result:
- `create_issue` is still allowed
- `delete_issue` is ignored because `include` takes precedence

## Utility-tool policy

Hermes may register these utility wrappers per MCP server:

Resources:
- `list_resources`
- `read_resource`

Prompts:
- `list_prompts`
- `get_prompt`

### Disable resources

```yaml
tools:
  resources: false
```

### Disable prompts

```yaml
tools:
  prompts: false
```

### Capability-aware registration

Even when `resources: true` or `prompts: true`, Hermes only registers those utility tools if the MCP session actually exposes the corresponding capability. So it's normal for prompts to be enabled but no prompt utilities to appear because the server does not support prompts.

## `enabled: false`

```yaml
mcp_servers:
  legacy:
    url: "https://mcp.legacy.internal"
    enabled: false
```

Behavior: no connection attempt, no discovery, no tool registration. Config remains in place for later reuse.

## Empty result behavior

If filtering removes all server-native tools and no utility tools are registered, Hermes does not create an empty MCP runtime toolset for that server.

## Example configs

### Safe GitHub allowlist

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "***"
    tools:
      include: [list_issues, create_issue, update_issue, search_code]
      resources: false
      prompts: false
```

### Stripe blacklist

```yaml
mcp_servers:
  stripe:
    url: "https://mcp.stripe.com"
    headers:
      Authorization: "Bearer ***"
    tools:
      exclude: [delete_customer, refund_payment]
```

### Resource-only docs server

```yaml
mcp_servers:
  docs:
    url: "https://mcp.docs.example.com"
    tools:
      include: []
      resources: true
      prompts: false
```

## Reloading config

After changing MCP config, reload servers with:

```
/reload-mcp
```

## Tool naming

Server-native MCP tools become:

```
mcp_<server>_<tool>
```

Examples:
- `mcp_github_create_issue`
- `mcp_filesystem_read_file`
- `mcp_my_api_query_data`

Utility tools follow the same prefixing pattern:
- `mcp_<server>_list_resources`
- `mcp_<server>_read_resource`
- `mcp_<server>_list_prompts`
- `mcp_<server>_get_prompt`

### Name sanitization

Hyphens (`-`) and dots (`.`) in both server names and tool names are replaced with underscores before registration. This ensures tool names are valid identifiers for LLM function-calling APIs.

For example, a server named `my-api` exposing a tool called `list-items.v2` becomes:

```
mcp_my_api_list_items_v2
```

Keep this in mind when writing `include` / `exclude` filters -- use the original MCP tool name (with hyphens/dots), not the sanitized version.

## OAuth 2.1 authentication (v0.8.0)

For HTTP servers that require OAuth, set `auth: oauth` on the server entry:

```yaml
mcp_servers:
  protected_api:
    url: "https://mcp.example.com/mcp"
    auth: oauth
```

For servers that do not support Dynamic Client Registration (DCR), supply pre-registered credentials via the optional `oauth` block:

```yaml
mcp_servers:
  slack:
    url: "https://mcp.slack.com/sse"
    auth: oauth
    oauth:
      client_id: "your-client-id"
      client_secret: "your-client-secret"   # confidential client only
      scope: "channels:read chat:write"      # overrides server-advertised scope
```

Behavior:
- Hermes uses `tools/mcp_oauth.py` — a full OAuth 2.1 PKCE adapter over the MCP SDK's `OAuthClientProvider`
- Flow: metadata discovery → DCR (or pre-registered client) → PKCE code exchange → token storage
- On first connect, a browser window opens for authorization
- Tokens are persisted to `~/.hermes/mcp-tokens/<server>.json` (permissions: `0o600`) and reused across sessions
- Token refresh is automatic; step-up re-authorization triggers on `403 insufficient_scope`
- Non-interactive environments (gateway, cron) log a warning when cached tokens are absent rather than hanging on a browser prompt
- Only applies to HTTP/StreamableHTTP transport (`url`-based servers)

## `no_mcp` sentinel (v0.8.0)

To exclude all MCP servers from a specific platform's toolset, add `no_mcp` to its `platform_toolsets` list:

```yaml
platform_toolsets:
  api_server:
    - terminal
    - file
    - no_mcp   # exclude all MCP servers from this platform
```

`no_mcp` is a sentinel — it is filtered out and does not appear as an actual toolset. Other platforms are unaffected. Useful for platforms where MCP tool schemas inflate token usage without providing value.
