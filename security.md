# Security

Hermes Agent is designed with a defense-in-depth security model. This page covers every security boundary — from command approval to container isolation to user authorization and credential handling.

---

## Security Model Overview

The security model has five layers:

1. **User authorization** — who can talk to the agent (allowlists, DM pairing)
2. **Dangerous command approval** — human-in-the-loop for destructive operations
3. **Container isolation** — Docker/Singularity/Modal sandboxing with hardened settings
4. **MCP credential filtering** — environment variable isolation for MCP subprocesses
5. **Context file scanning** — prompt injection detection in project files

Each layer is independent. Enabling container isolation does not disable the other layers unless explicitly configured (the approval check is skipped when running in a container because the container is itself the security boundary).

---

## Dangerous Command Approval

Before executing any terminal command, Hermes checks it against a curated list of dangerous patterns defined in `tools/approval.py`. If a match is found, execution pauses and the user must explicitly approve it.

### What Triggers Approval

| Pattern | Description |
|---------|-------------|
| `rm -r` / `rm --recursive` | Recursive delete |
| `rm ... /` | Delete in root path |
| `chmod 777` | World-writable permissions |
| `mkfs` | Format filesystem |
| `dd if=` | Disk copy |
| `DROP TABLE` / `DROP DATABASE` | SQL DROP |
| `DELETE FROM` (without WHERE clause) | SQL DELETE without WHERE |
| `TRUNCATE TABLE` | SQL TRUNCATE |
| `> /etc/` | Overwrite system config file |
| `systemctl stop` / `disable` / `mask` | Stop or disable system services |
| `kill -9 -1` | Kill all processes |
| `curl ... \| sh` | Pipe remote content directly to shell |
| `bash -c`, `python -e` | Shell/script execution via flags |
| `find -exec rm` / `find -delete` | Find with destructive actions |
| Fork bomb patterns | Fork bombs |

Container bypass: when running in `docker`, `singularity`, `modal`, or `daytona` backends, dangerous command checks are skipped because the container itself is the security boundary. Destructive commands inside a container cannot harm the host.

### Approval Flow: CLI

In the interactive CLI, dangerous commands show an inline approval prompt:

```
  DANGEROUS COMMAND: recursive delete
      rm -rf /tmp/old-project

      [o]nce  |  [s]ession  |  [a]lways  |  [d]eny

      Choice [o/s/a/D]:
```

Options:

- **once** — allow this single execution
- **session** — allow this pattern for the rest of the current session
- **always** — add to the permanent allowlist (saved to `~/.hermes/config.yaml`)
- **deny** (default) — block the command

### Approval Flow: Gateway / Messaging

On messaging platforms, the agent sends the dangerous command details to the chat and waits for a reply:

- Reply `yes`, `y`, `approve`, `ok`, or `go` to approve
- Reply `no`, `n`, `deny`, or `cancel` to deny

The `HERMES_EXEC_ASK=1` environment variable is automatically set when running the gateway.

### Permanent Allowlist

Commands approved with "always" are saved to `~/.hermes/config.yaml`:

```yaml
# Permanently allowed dangerous command patterns
command_allowlist:
  - rm
  - systemctl
```

These patterns are loaded at startup and silently approved in all future sessions. Review and audit this list periodically with `hermes config edit`.

### Smart Learning (v0.3.0)

In v0.3.0, the approval system learns from your session choices. If you approve a pattern with "session", the agent remembers the choice within that session and does not re-prompt for the same pattern.

---

## /stop Command (v0.3.0)

v0.3.0 adds the `/stop` command, which immediately halts the current agent task without requiring keyboard interaction. This is available in the CLI and can be sent via messaging platforms. It is equivalent to an interrupt signal but goes through the agent's normal cancellation path, ensuring clean shutdown of any running tool executions.

---

## User Authorization (Gateway)

When running the messaging gateway, Hermes controls who can interact with the bot through a layered authorization system.

### Authorization Check Order

The `_is_user_authorized()` method checks in this order:

1. Per-platform allow-all flag (e.g., `DISCORD_ALLOW_ALL_USERS=true`)
2. DM pairing approved list (users approved via pairing codes)
3. Platform-specific allowlists (e.g., `TELEGRAM_ALLOWED_USERS=12345,67890`)
4. Global allowlist (`GATEWAY_ALLOWED_USERS=12345,67890`)
5. Global allow-all (`GATEWAY_ALLOW_ALL_USERS=true`)
6. Default: deny

If no allowlists are configured and `GATEWAY_ALLOW_ALL_USERS` is not set, all users are denied. The gateway logs a warning at startup:

```
No user allowlists configured. All unauthorized users will be denied.
Set GATEWAY_ALLOW_ALL_USERS=true in ~/.hermes/.env to allow open access,
or configure platform allowlists (e.g., TELEGRAM_ALLOWED_USERS=your_id).
```

### Platform Allowlists

Set allowed user IDs as comma-separated values in `~/.hermes/.env`:

```bash
# Platform-specific allowlists
TELEGRAM_ALLOWED_USERS=123456789,987654321
DISCORD_ALLOWED_USERS=111222333444555666
WHATSAPP_ALLOWED_USERS=15551234567
SLACK_ALLOWED_USERS=U01ABC123

# Cross-platform allowlist (checked for all platforms)
GATEWAY_ALLOWED_USERS=123456789

# Per-platform allow-all (use with caution)
DISCORD_ALLOW_ALL_USERS=true

# Global allow-all (use with extreme caution — not for production)
GATEWAY_ALLOW_ALL_USERS=true
```

### DM Pairing System

For flexible authorization without hardcoding user IDs upfront, Hermes provides a code-based pairing system. Unknown users receive a one-time pairing code that the bot owner approves via the CLI.

How it works:

1. An unknown user sends a DM to the bot
2. The bot replies with an 8-character pairing code
3. The bot owner runs `hermes pairing approve <platform> <code>` on the CLI
4. The user is permanently approved for that platform

Control how unauthorized DMs are handled in `~/.hermes/config.yaml`:

```yaml
unauthorized_dm_behavior: pair

whatsapp:
  unauthorized_dm_behavior: ignore
```

- `pair` (default): Unauthorized DMs receive a pairing code reply
- `ignore`: Unauthorized DMs are silently dropped

Platform sections override the global default.

Security properties (based on OWASP and NIST SP 800-63-4 guidance):

| Feature | Details |
|---------|---------|
| Code format | 8 characters from a 32-character unambiguous alphabet (excludes 0/O/1/I) |
| Randomness | Cryptographic (`secrets.choice()`) |
| Code TTL | 1 hour expiry |
| Rate limiting | 1 request per user per 10 minutes |
| Pending limit | Maximum 3 pending codes per platform |
| Lockout | 5 failed approval attempts triggers a 1-hour lockout |
| File security | `chmod 0600` on all pairing data files |
| Logging | Codes are never logged to stdout |

Pairing CLI commands:

```bash
hermes pairing list                          # List pending and approved users
hermes pairing approve telegram ABC12DEF     # Approve a pairing code
hermes pairing revoke telegram 123456789     # Revoke a user's access
hermes pairing clear-pending                 # Clear all pending codes
```

Pairing data is stored in `~/.hermes/pairing/` with per-platform JSON files:

- `{platform}-pending.json` — pending pairing requests
- `{platform}-approved.json` — approved users
- `_rate_limits.json` — rate limit and lockout tracking

---

## OAuth and Authentication

### Nous Portal OAuth (Device Code Flow)

The Nous Portal provider uses OAuth 2.0 device code flow. When you run `hermes model` and select Nous Portal:

1. A device code is requested from `https://portal.nousresearch.com`
2. You visit the authorization URL and enter the code shown in the terminal
3. Hermes polls for completion and stores the OAuth token in `~/.hermes/auth.json`
4. Tokens are refreshed automatically 2 minutes before expiry (`ACCESS_TOKEN_REFRESH_SKEW_SECONDS = 120`)
5. The agent key is minted fresh when it has less than 30 minutes remaining TTL

The `~/.hermes/auth.json` file uses `chmod 0600` permissions and cross-process file locking to prevent race conditions.

### Anthropic OAuth PKCE (v0.3.0)

v0.3.0 adds support for Anthropic OAuth using PKCE (Proof Key for Code Exchange). The provider resolves Anthropic credentials via `agent/anthropic_adapter.py` with this search order:

1. `ANTHROPIC_TOKEN` environment variable
2. `ANTHROPIC_API_KEY` environment variable
3. Claude Code credential auto-discovery (reads from Claude Code's stored credentials)

If none of these are present, authentication fails with a clear error message directing you to set `ANTHROPIC_TOKEN` or `ANTHROPIC_API_KEY`, run `claude setup-token`, or authenticate with `claude /login`.

### GitHub Copilot Authentication

The `copilot_auth.py` module implements the OAuth device code flow used by the Copilot CLI:

```
Credential search order:
  1. COPILOT_GITHUB_TOKEN env var
  2. GH_TOKEN env var
  3. GITHUB_TOKEN env var
  4. gh auth token  (GitHub CLI fallback)
```

Supported token types:

| Token prefix | Type | Supported |
|-------------|------|-----------|
| `gho_` | OAuth token (default via `copilot login`) | Yes |
| `github_pat_` | Fine-grained PAT (needs Copilot Requests permission) | Yes |
| `ghu_` | GitHub App token (via env var) | Yes |
| `ghp_` | Classic PAT | **No** |

Classic PATs (`ghp_*`) are explicitly rejected with a helpful error message explaining the alternatives.

### OpenAI Codex OAuth

The Codex provider uses an external OAuth flow:

- Client ID: `app_EMoamEEZ73f0CkXaXp7hrann`
- Token URL: `https://auth.openai.com/oauth/token`
- Access tokens are refreshed 2 minutes before expiry

### Claude Code Credential Auto-Discovery

When the Anthropic provider is selected, Hermes reads credentials from Claude Code's stored auth if present, enabling zero-configuration use when Claude Code is already authenticated.

---

## API Key Handling

### Environment Variables

API keys are stored in `~/.hermes/.env` and loaded via `python-dotenv`. The file should have restricted permissions:

```bash
chmod 600 ~/.hermes/.env
```

The file is never committed to version control. Example entries:

```bash
OPENROUTER_API_KEY=sk-or-v1-your-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here
FIRECRAWL_API_KEY=fc-your-key
FAL_KEY=your-fal-key
```

### Config File Keys

Keys can also be set in `~/.hermes/config.yaml` under the `model` block:

```yaml
model:
  provider: openrouter
  api_key: sk-or-v1-your-key-here
```

### Credential Priority

For OpenRouter and custom endpoints (`hermes_cli/runtime_provider.py`):

1. Explicit CLI argument
2. `model.api_key` in `~/.hermes/config.yaml`
3. `OPENROUTER_API_KEY` (for OpenRouter URLs)
4. `OPENAI_API_KEY` (for custom endpoints)

The resolver deliberately prevents `OPENROUTER_API_KEY` from being sent to non-OpenRouter custom endpoints to avoid credential leakage.

---

## PII Redaction

### MCP Error Redaction

Error messages from MCP tool calls are sanitized before being returned to the LLM. The following patterns are replaced with `[REDACTED]`:

- GitHub PATs (`ghp_...`)
- OpenAI-style API keys (`sk-...`)
- Bearer tokens
- `token=`, `key=`, `API_KEY=`, `password=`, `secret=` query parameters

This prevents accidental credential exposure in error messages that go into the LLM context window.

### Code Execution Sandbox

The `execute_code` tool strips API keys from the child process environment before running user code. This ensures code running in the sandbox cannot read host API keys even if it tries to inspect `os.environ`.

---

## Container Isolation

When using the `docker` terminal backend, Hermes applies strict security hardening to every container (defined in `tools/environments/docker.py`):

```python
_SECURITY_ARGS = [
    "--cap-drop", "ALL",                          # Drop ALL Linux capabilities
    "--security-opt", "no-new-privileges",         # Block privilege escalation
    "--pids-limit", "256",                         # Limit process count
    "--tmpfs", "/tmp:rw,nosuid,size=512m",         # Size-limited /tmp
    "--tmpfs", "/var/tmp:rw,noexec,nosuid,size=256m",  # No-exec /var/tmp
    "--tmpfs", "/run:rw,noexec,nosuid,size=64m",   # No-exec /run
]
```

### Resource Limits

Container resources are configurable in `~/.hermes/config.yaml`:

```yaml
terminal:
  backend: docker
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_forward_env: []  # Explicit allowlist — empty keeps secrets out of container
  container_cpu: 1        # CPU cores
  container_memory: 5120  # MB (default 5 GB)
  container_disk: 51200   # MB (default 50 GB, requires overlay2 on XFS)
  container_persistent: true  # Persist filesystem across sessions
```

### Filesystem Persistence

- **Persistent mode** (`container_persistent: true`): Bind-mounts `/workspace` and `/root` from `~/.hermes/sandboxes/docker/<task_id>/`
- **Ephemeral mode** (`container_persistent: false`): Uses tmpfs for workspace — everything is discarded on cleanup

### docker_forward_env Warning

`docker_forward_env` is an explicit allowlist of host environment variables to inject into containers. It is empty by default to keep host secrets out of containers. If you add names to this list, those variables are intentionally injected and code running in the container can read and exfiltrate them.

---

## Terminal Backend Security Comparison

| Backend | Isolation | Dangerous Cmd Check | Best For |
|---------|-----------|---------------------|----------|
| **local** | None — runs on host | Yes | Development, trusted users |
| **ssh** | Remote machine | Yes | Running on a separate server |
| **docker** | Container | Skipped (container is boundary) | Production gateway |
| **singularity** | Container | Skipped | HPC environments |
| **modal** | Cloud sandbox | Skipped | Scalable cloud isolation |
| **daytona** | Cloud sandbox | Skipped | Persistent cloud workspaces |

For production gateway deployments, use `docker`, `modal`, or `daytona` to isolate agent commands from the host system. This eliminates the need for dangerous command approval entirely.

---

## MCP Credential Handling

MCP (Model Context Protocol) server subprocesses receive a filtered environment to prevent accidental credential leakage.

### Safe Environment Variables

Only these variables are passed through from the host to MCP stdio subprocesses:

```
PATH, HOME, USER, LANG, LC_ALL, TERM, SHELL, TMPDIR
```

Plus any `XDG_*` variables. All other environment variables including API keys and tokens are stripped.

Variables explicitly defined in the MCP server's `env` config block are passed through:

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."  # Only this specific variable is passed
```

### Website Access Policy

You can restrict which websites the agent can access through web and browser tools:

```yaml
# In ~/.hermes/config.yaml
website_blocklist:
  enabled: true
  domains:
    - "*.internal.company.com"
    - "admin.example.com"
  shared_files:
    - "/etc/hermes/blocked-sites.txt"
```

The blocklist is enforced across `web_search`, `web_extract`, `browser_navigate`, and all URL-capable tools. When a blocked URL is requested, the tool returns an error explaining the domain is blocked by policy.

---

## Context File Injection Protection

Context files (`AGENTS.md`, `.cursorrules`, `SOUL.md`) are scanned for prompt injection before being included in the system prompt. The scanner checks for:

- Instructions to ignore or disregard prior instructions
- Hidden HTML comments with suspicious keywords
- Attempts to read secrets (`.env`, `credentials`, `.netrc`)
- Credential exfiltration via `curl`
- Invisible Unicode characters (zero-width spaces, bidirectional overrides)

Blocked files show a warning in the session:

```
[BLOCKED: AGENTS.md contained potential prompt injection (prompt_injection). Content not loaded.]
```

Memory entries are also scanned for injection and exfiltration patterns before being accepted, since they are injected into the system prompt.

---

## Secrets Management

### Secrets Storage Locations

| Location | Contents | Permissions |
|----------|----------|-------------|
| `~/.hermes/.env` | API keys and secrets | `chmod 600` recommended |
| `~/.hermes/auth.json` | OAuth credentials (Nous Portal, Codex) | `chmod 600` enforced by code |
| `~/.hermes/config.yaml` | Configuration (may contain API keys) | User-controlled |
| `~/.hermes/pairing/` | Gateway pairing data | `chmod 600` enforced by code |

### What to Never Do

- Never commit `~/.hermes/.env` to version control
- Never use `GATEWAY_ALLOW_ALL_USERS=true` in production
- Never run the gateway as root
- Never add secrets to `docker_forward_env` unless the code running in the container absolutely requires them

---

## Gateway Security: Multi-User Isolation

When multiple users interact with the gateway (e.g., via Telegram or Discord), each user session is isolated:

- Each user gets their own agent instance and conversation history
- Tool calls for one user cannot read another user's session state
- Honcho memory (when enabled) is scoped per-session and uses the platform and chat ID as the session key
- Background memory flushes preserve the original session key to ensure writes go to the correct user's Honcho session

Per-user session context prompts are injected at request time so different users can have different personas or toolset configurations.

---

## Security Best Practices for Production Deployment

### Gateway Deployment Checklist

1. **Set explicit allowlists** — never use `GATEWAY_ALLOW_ALL_USERS=true` in production
2. **Use container backend** — set `terminal.backend: docker` in config.yaml
3. **Restrict resource limits** — set appropriate CPU, memory, and disk limits in the container config
4. **Store secrets securely** — keep API keys in `~/.hermes/.env` with `chmod 600`
5. **Enable DM pairing** — use pairing codes instead of hardcoding user IDs
6. **Review command allowlist** — periodically audit `command_allowlist` in config.yaml with `hermes config edit`
7. **Set `MESSAGING_CWD`** — do not let the agent operate from sensitive directories
8. **Run as non-root** — never run the gateway as root
9. **Monitor logs** — check `~/.hermes/logs/` for unauthorized access attempts
10. **Keep updated** — run `hermes update` regularly for security patches

### Securing API Keys

```bash
chmod 600 ~/.hermes/.env
chmod 600 ~/.hermes/auth.json

# Keep separate keys for different services
# Never commit .env files to version control
# Rotate keys periodically
```

### Network Isolation

For maximum security, run the gateway on a separate machine or VM:

```yaml
terminal:
  backend: ssh
  ssh_host: "agent-worker.local"
  ssh_user: "hermes"
  ssh_key: "~/.ssh/hermes_agent_key"
```

This keeps the gateway's messaging connections separate from the agent's command execution.

---

## Contributing Security-Sensitive Code

When writing code that touches security boundaries:

- Always use `shlex.quote()` when interpolating user input into shell commands
- Resolve symlinks with `os.path.realpath()` before path-based access control checks
- Do not log secrets — API keys, tokens, and passwords must never appear in log output
- Catch broad exceptions around tool execution so a single failure does not crash the agent loop
- Test on all platforms if your change touches file paths, process management, or shell commands

Existing protections to be aware of:

| Layer | Implementation |
|-------|---------------|
| Sudo password piping | Uses `shlex.quote()` to prevent shell injection |
| Dangerous command detection | Regex patterns in `tools/approval.py` with user approval flow |
| Cron prompt injection | Scanner in `tools/cronjob_tools.py` blocks instruction-override patterns |
| Write deny list | Protected paths resolved via `os.path.realpath()` to prevent symlink bypass |
| Skills guard | Security scanner for hub-installed skills (`tools/skills_guard.py`) |
| Code execution sandbox | `execute_code` child process runs with API keys stripped from environment |
| Container hardening | Docker: all capabilities dropped, no privilege escalation, PID limits, size-limited tmpfs |

If your PR affects security, note it explicitly in the PR description.
