# Security

Hermes Agent implements defense-in-depth with multiple security layers. This document covers the security model, hardening features, and best practices.

---

## Security Architecture

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| **File permissions** | 0700 (dirs), 0600 (files) | Prevent credential exposure |
| **Terminal isolation** | Container backends (Docker/Modal) | Sandbox command execution |
| **Path traversal protection** | Boundary checks on all file ops | Prevent escaping working directory |
| **Shell injection prevention** | Direct exec (not shell interpreter) | Prevent `; rm -rf` style attacks |
| **Prompt injection protection** | Context file scanning before load | Prevent malicious AGENTS.md injection |
| **Symlink boundary checks** | Follow-limit + boundary validation | Prevent symlink escape attacks |
| **Secret redaction** | Pattern-based log filtering | Prevent credential leakage in logs |
| **Atomic writes** | temp file + fsync + os.replace | Prevent partial-write corruption |
| **OAuth device flow** | PKCE + browser auth | No credential storage for Nous Portal |
| **Auth store locking** | File-based locking | Prevent race conditions on `auth.json` |
| **FTS5 query sanitization** | Input validation before FTS5 | Prevent SQLite injection via search |
| **Dangerous command detection** | Pattern matching on shell commands | Warn before destructive operations |

---

## File System Security

### Permissions

All Hermes data directories use strict permissions:

```
~/.hermes/                  (0700)
~/.hermes/.env              (0600)  ← API keys
~/.hermes/auth.json         (0600)  ← OAuth tokens
~/.hermes/state.db          (0600)  ← Session database
~/.hermes/skills/           (0700)
~/.hermes/cron/             (0700)
```

### Path Traversal Protection

All file operation tools enforce working directory boundaries. Attempts to read/write outside the permitted path are blocked:

```
read_file("../../etc/passwd")    → Blocked
write_file("/etc/cron.d/evil")   → Blocked
```

### Symlink Checks

Symbolic links are validated against configured boundaries. A symlink pointing outside the working directory is treated as a traversal attempt.

### Atomic Writes

All sensitive files are written atomically:

1. Write content to a temp file (`file.tmp`)
2. `fsync()` to flush to disk
3. `os.replace()` to atomically swap

Affected files: `auth.json`, `gateway.json`, `cron/jobs.json`, `state.db` (WAL), skill files, `.env`.

This prevents partial writes that could leave files in a corrupt state.

---

## Terminal Security

### Container Isolation

The Docker and Modal backends run commands in isolated containers:

```yaml
terminal:
  backend: "docker"
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  container_cpu: 1
  container_memory: 5120    # 5 GB limit
  container_disk: 51200     # 50 GB limit
```

Containers:
- Have no access to host filesystem (unless volumes explicitly mounted)
- Run with limited CPU/memory/disk
- Cannot access host network services by default

### Dangerous Command Detection

The terminal tool detects patterns associated with destructive operations:

- `rm -rf /`, `rm -rf /*`
- `dd if=/dev/zero of=/dev/sda`
- Fork bombs
- Overwriting critical system files

Detected commands trigger a warning before execution.

### Command Timeout

```yaml
terminal:
  timeout: 180    # Kill process after 3 minutes
```

Prevents runaway processes from consuming resources indefinitely.

---

## Prompt Injection Protection

### Context File Scanning

Before loading any context file (`AGENTS.md`, `.cursorrules`, `SOUL.md`), the prompt builder scans for injection patterns:

- Instructions to ignore previous system prompt
- Commands to override agent behavior
- Directives to exfiltrate data or credentials

Suspicious files are rejected with a warning logged.

### Input Sanitization

All user input that flows into SQL (FTS5 search queries) is sanitized to prevent injection attacks.

---

## Secret Management

### Credential Storage

API keys and tokens are stored in `~/.hermes/.env` with `0600` permissions (readable only by the owner).

OAuth tokens are stored in `~/.hermes/auth.json` with the same permissions.

### Secret Redaction in Logs

The `agent/redact.py` module filters sensitive patterns from all logs and exports:

- API keys matching known patterns (`sk-...`, `xoxb-...`, etc.)
- OAuth access tokens
- Environment variable values matching key names

An in-memory allowlist prevents accidental logging of secrets even in debug mode.

### Zero-Storage OAuth

The Nous Portal uses OAuth device code flow — credentials are never stored in the config file or shell history. The long-lived token is stored in `auth.json`.

---

## Authentication (Gateway)

### DM Pairing

For messaging platforms, Hermes implements a DM pairing model:

1. User sends a DM to the bot (not in a group)
2. Bot verifies the user is in the `allowed_users` list (if configured)
3. Paired users can issue admin commands

### Allowed Users

Restrict platform access to specific users:

```bash
# Telegram: by user ID
TELEGRAM_ALLOWED_USERS=123456789,987654321

# WhatsApp: by E164 phone number
WHATSAPP_ALLOWED_USERS=+1234567890,+0987654321
```

Empty = allow all users.

### Command Approval

By default, shell commands require explicit confirmation from the user before execution. This can be configured per-session or disabled for trusted automated environments.

---

## Network Security

### Local-Only Default

By default, Hermes listens only on localhost (`127.0.0.1:4200` for the API, if enabled). Gateway uses outbound-only connections to platforms.

### SSRF Protection

Web fetch tools implement protections against Server-Side Request Forgery:

- Block requests to `169.254.169.254` (AWS metadata)
- Block requests to `10.x.x.x`, `172.16.x.x`, `192.168.x.x` (private networks)
- Configurable allowlist/blocklist

---

## Best Practices

### Minimal Capability Principle

Enable only the toolsets you need:

```bash
hermes --toolset safe       # Web + vision, no terminal
hermes --toolset debugging  # Terminal + web + files
```

Fewer tools = smaller attack surface.

### Container Backend for Untrusted Work

When working with untrusted code or running agent-generated scripts:

```yaml
terminal:
  backend: "docker"
```

All shell commands run in an isolated container.

### Restrict Platform Access

For production gateway deployments, always configure allowed users:

```bash
TELEGRAM_ALLOWED_USERS=your_telegram_id
```

### Secure Skill Installation

- Use `official` source for production: `hermes skills install official/...`
- Review `dangerous` verdicts before using `--force`
- Audit community skills before installation

### Prompt Injection in Repos

When using Hermes in codebases you didn't author:

- Review `AGENTS.md` files before letting Hermes load them
- Use `--no-context-files` flag to disable automatic context loading in untrusted repos

---

## Security Fixes in v0.2.0

The v0.2.0 release included 20+ security improvements:

- Path traversal fix for file operations
- Shell injection prevention via direct exec
- Dangerous command detection patterns
- Symlink boundary checks
- Prompt injection bypass prevention
- FTS5 query sanitization
- `.env` permissions enforcement (0600/0700)
- Atomic writes for all critical files
- Secret redaction patterns + in-memory allowlist
- File locking for multi-process auth store access
