# Security

Hermes Agent is designed with a defense-in-depth security model. This page covers every security boundary -- from command approval to container isolation to user authorization, credential handling, PII redaction, and tirith scanning. Sections marked **v0.4.0** and **v0.5.0** document changes introduced in those releases.

---

## Security Model Overview

The security model has six layers:

1. **User authorization** -- who can talk to the agent (allowlists, DM pairing)
2. **Dangerous command approval** -- human-in-the-loop for destructive operations
3. **Tirith pre-exec scanning** -- content-level threat detection (homograph URLs, terminal injection)
4. **Container isolation** -- Docker/Singularity/Modal sandboxing with hardened settings
5. **PII and secret redaction** -- automatic masking in logs, tool output, and MCP errors
6. **Context file scanning** -- prompt injection detection in project files and memory entries

Each layer is independent. Enabling container isolation does not disable the other layers unless explicitly configured (the approval check is skipped when running in a container because the container is itself the security boundary).

Released v0.7.0 extends the existing layers with stronger secret-exfiltration checks in browser and tool surfaces, broader protected credential directories, and secret redaction for `execute_code` sandbox output. Released v0.8.0 adds consolidated SSRF protections, timing attack mitigations, tar traversal prevention, credential leakage guards, cron path traversal hardening, cross-session isolation, terminal workdir sanitization across all backends, and MCP package malware scanning.

---

## Dangerous Command Approval

Before executing any terminal command, Hermes checks it against a curated list of dangerous patterns defined in `tools/approval.py`. If a match is found, execution pauses and the user must explicitly approve it.

### What Triggers Approval

The complete list of patterns from `tools/approval.py`:

| Pattern | Description |
|---------|-------------|
| `rm ... /` | Delete in root path |
| `rm -r` / `rm --recursive` | Recursive delete |
| `chmod 777` / `chmod --recursive ... 777` | World-writable permissions |
| `chown -R root` / `chown --recursive ... root` | Recursive chown to root |
| `mkfs` | Format filesystem |
| `dd if=` | Disk copy |
| `> /dev/sd*` | Write to block device |
| `DROP TABLE` / `DROP DATABASE` | SQL DROP |
| `DELETE FROM` (without `WHERE`) | SQL DELETE without WHERE |
| `TRUNCATE TABLE` | SQL TRUNCATE |
| `> /etc/` | Overwrite system config file |
| `systemctl stop` / `disable` / `mask` | Stop or disable system services |
| `kill -9 -1` | Kill all processes |
| `pkill -9` | Force kill processes |
| Fork bomb patterns | Fork bombs (`:(){:|:&};:`) |
| `bash -c` / `sh -lc` / `zsh -c` | Shell command via `-c` flag (includes combined flags like `-lc`, `-ic`) |
| `python -e` / `perl -e` / `ruby -e` / `node -c` | Script execution via `-e`/`-c` flag |
| `curl ... \| sh` / `wget ... \| bash` | Pipe remote content to shell |
| `bash < <(curl ...)` | Execute remote script via process substitution |
| `tee ... /etc/` / `tee ... .ssh/` / `tee ... .hermes/.env` | Overwrite system file via tee |
| `xargs ... rm` | xargs with rm |
| `find ... -exec rm` | find -exec rm |
| `find ... -delete` | find -delete |

Container bypass: when running in `docker`, `singularity`, `modal`, or `daytona` backends, dangerous command checks are skipped because the container itself is the security boundary. Destructive commands inside a container cannot harm the host. This assumes the container is correctly configured -- if `container_cpu`, `container_memory`, or `container_disk` limits are set too high or if `docker_forward_env` passes sensitive host credentials into the container, the isolation guarantee weakens. An agent inside a misconfigured container could exhaust host resources (if limits are not enforced by the runtime) or exfiltrate forwarded credentials. Always verify that resource limits are enforced by your container runtime and keep `docker_forward_env` empty unless specific variables are required.

### Approval Modes

The approval system supports three modes, configured via `approvals.mode` in `config.yaml`:

| Mode | Behavior |
|------|----------|
| `manual` | Always prompt the user (default) |
| `smart` | Ask an auxiliary LLM to assess risk first; auto-approve low-risk commands, escalate to user for genuinely dangerous ones |
| `off` | Skip all approval prompts (equivalent to `HERMES_YOLO_MODE=1`) |

Smart approval uses the auxiliary LLM client to classify commands as APPROVE, DENY, or ESCALATE. Only ESCALATE falls through to the manual prompt. When smart mode auto-approves a command, it also grants session-level approval for subsequent identical commands -- the same pattern will not be re-evaluated by the LLM for the remainder of the session.

### Approval Flow: CLI

In the interactive CLI, dangerous commands show an inline approval prompt:

```
  DANGEROUS COMMAND: recursive delete
      rm -rf /tmp/old-project

      [o]nce  |  [s]ession  |  [a]lways  |  [d]eny

      Choice [o/s/a/D]:
```

Options:

- **once** -- allow this single execution
- **session** -- allow this pattern for the rest of the current session
- **always** -- add to the permanent allowlist (saved to `~/.hermes/config.yaml`)
- **deny** (default) -- block the command; times out after 60 seconds

When tirith security warnings are present alongside a dangerous command pattern, both findings are combined into a single approval prompt. The `[a]lways` option is hidden when tirith warnings are present, since broad permanent allowlisting is inappropriate for content-level security findings.

### Approval Flow: Gateway / Messaging

On messaging platforms, the agent sends the dangerous command details to the chat and waits for an explicit slash-command response:

- Reply with `/approve` to approve the pending command
- Reply with `/deny` to reject it

The `HERMES_EXEC_ASK=1` environment variable is automatically set when running the gateway.

### Permanent Allowlist

Commands approved with "always" are saved to `~/.hermes/config.yaml`:

```yaml
# Permanently allowed dangerous command patterns
command_allowlist:
  - recursive delete
  - stop/disable system service
```

These patterns are loaded at startup and silently approved in all future sessions. The allowlist uses human-readable description strings (e.g., "recursive delete") rather than regex patterns. Legacy regex-derived keys from older versions are automatically recognized via alias mapping. Review and audit this list periodically with `hermes config edit`.

### Per-Session Approval State

Approval state is thread-safe and keyed by session. Each session maintains its own set of approved patterns. When a session ends, its approvals are cleared. The permanent allowlist persists across sessions and processes.

---

## Supply Chain Hardening (v0.5.0)

v0.5.0 included a comprehensive supply chain audit (PRs [#2796](https://github.com/NousResearch/hermes-agent/pull/2796), [#2810](https://github.com/NousResearch/hermes-agent/pull/2810), [#2812](https://github.com/NousResearch/hermes-agent/pull/2812), [#2816](https://github.com/NousResearch/hermes-agent/pull/2816), [#3073](https://github.com/NousResearch/hermes-agent/pull/3073)).

### litellm Removal

The `litellm`, `typer`, and `platformdirs` packages were removed from `pyproject.toml` after a compromise assessment ([#2796](https://github.com/NousResearch/hermes-agent/pull/2796)). Hermes previously relied on litellm as a provider abstraction layer; that functionality is now provided directly through the native `openai`, `anthropic`, and custom HTTP clients already present in the codebase. Verify the absence by inspecting `pyproject.toml` -- neither `litellm` nor `typer` appears in any dependency block.

### Pinned Dependency Ranges

All dependency version ranges in `pyproject.toml` are now pinned with explicit upper bounds (e.g., `openai>=2.21.0,<3`, `requests>=2.33.0,<3`) to reduce the window for malicious version-bump attacks ([#2810](https://github.com/NousResearch/hermes-agent/pull/2810)). Several pins address specific CVEs:

| Package | Constraint | Reason |
|---------|-----------|--------|
| `requests` | `>=2.33.0,<3` | CVE-2026-25645 |
| `PyJWT[crypto]` | `>=2.12.0,<3` | CVE-2026-32597 |

### uv.lock with Hashes

`uv.lock` is regenerated with cryptographic hashes for every dependency entry and is consumed by the installer during `hermes setup` ([#2812](https://github.com/NousResearch/hermes-agent/pull/2812)). This ensures that every package installed during setup matches an exact, pre-verified artifact -- any tampered package would fail the hash check before it reaches disk. Reproducibility: two machines running `hermes setup` from the same lockfile install byte-for-byte identical packages regardless of upstream registry state.

### CI Supply Chain Scanning

A GitHub Actions workflow was added that runs automatically on every pull request and scans for supply chain attack patterns ([#2816](https://github.com/NousResearch/hermes-agent/pull/2816)). The workflow flags suspicious additions such as new packages with typosquatting-like names, unexpected `install_requires` mutations, and patterns used in known supply chain attacks. This provides a continuous audit layer for all external contributions.

---

## Tirith Pre-Exec Security Scanning

Hermes integrates the tirith binary (`tools/tirith_security.py`) as an additional pre-exec security layer that scans commands for content-level threats invisible to regex pattern matching:

- Homograph URL attacks (unicode lookalikes)
- Pipe-to-interpreter chains
- Terminal escape sequence injection
- Other content-level threats

### How Tirith Works

Tirith runs as a subprocess before every terminal command execution. Its exit code determines the verdict:

| Exit code | Action | Meaning |
|-----------|--------|---------|
| 0 | `allow` | Command is safe |
| 1 | `block` | Verdict presented for approval (see v0.5.0 change below) |
| 2 | `warn` | Warning shown, user can approve |

JSON stdout from tirith enriches findings and summary text but never overrides the exit code verdict.

### Approvable Block Verdicts (v0.5.0)

Prior to v0.5.0, a tirith exit code of `1` hard-blocked command execution with no recourse. As of v0.5.0 ([#3428](https://github.com/NousResearch/hermes-agent/pull/3428)), block verdicts are now surfaced as an approval prompt rather than an unconditional halt. This allows the agent owner to override a false positive without disabling tirith entirely. The prompt displays the tirith finding alongside the command and requires an explicit `[o]nce`, `[s]ession`, or `[a]lways` acknowledgement -- the same flow as dangerous-command approval. In gateway/messaging mode, the agent sends the findings to chat and waits for an `/approve` or `/deny` response.

### Auto-Install

If tirith is not found on PATH or at the configured path, it is automatically downloaded from GitHub releases to `$HERMES_HOME/bin/tirith`. The download:

1. Verifies SHA-256 checksums (always)
2. Verifies cosign provenance signatures when cosign is on PATH (optional but recommended)
3. Runs in a background thread so startup never blocks

Failed installs are cached for 24 hours to avoid repeated network attempts. The failure marker at `~/.hermes/.tirith-install-failed` records the reason and auto-clears when the cause is resolved (e.g., cosign becomes available).

### Configuration

```yaml
# In ~/.hermes/config.yaml
security:
  tirith_enabled: true          # Enable/disable tirith scanning
  tirith_path: tirith           # Path to tirith binary (bare name = auto-resolve)
  tirith_timeout: 5             # Subprocess timeout in seconds
  tirith_fail_open: true        # On spawn failure/timeout: allow (true) or block (false)
```

Environment variable overrides: `TIRITH_ENABLED`, `TIRITH_BIN`, `TIRITH_TIMEOUT`, `TIRITH_FAIL_OPEN`.

### Combined Guard

The `check_all_command_guards()` function in `tools/approval.py` runs both tirith scanning and dangerous-command regex detection simultaneously. Both checks execute in parallel, and their findings are merged into a single combined approval request. If tirith flags a homograph URL and the regex matcher flags a pipe-to-shell pattern in the same command, both findings appear together in one prompt. This prevents a gateway `force=True` replay from bypassing one check when only the other was shown to the user. The combined approach also means a command cannot slip through by satisfying one guard while failing the other -- both must pass for automatic approval.

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

# Global allow-all (use with extreme caution -- not for production)
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

Security properties:

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

- `{platform}-pending.json` -- pending pairing requests
- `{platform}-approved.json` -- approved users
- `_rate_limits.json` -- rate limit and lockout tracking

---

## Secret Exfiltration Hardening (v0.7.0)

Released v0.7.0 adds several concrete protections on top of the existing redaction and approval model:

- **Browser URL exfiltration blocking** -- browser navigation rejects URLs that embed apparent secrets or encoded secret material
- **`execute_code` output redaction** -- sandbox output is scrubbed before it is surfaced back to the model
- **Expanded protected directories** -- file/context guards now explicitly cover `.docker`, `.azure`, and `.config/gh` in addition to earlier sensitive locations
- **Broader token patterns** -- GitHub OAuth-style tokens and related credential patterns are redacted more consistently

These protections are additive. They do not replace approval prompts, tirith scanning, or container isolation.

---

## OAuth and Authentication

### Nous Portal OAuth (Device Code Flow)

The Nous Portal provider uses OAuth 2.0 device code flow. When you run `hermes model` and select Nous Portal:

1. A device code is requested from `https://portal.nousresearch.com`
2. You visit the authorization URL and enter the code shown in the terminal
3. Hermes polls for completion and stores the OAuth token in `~/.hermes/auth.json`
4. Tokens are refreshed automatically 2 minutes before expiry (`ACCESS_TOKEN_REFRESH_SKEW_SECONDS = 120`)
5. The agent key is minted fresh when it has less than 30 minutes remaining TTL (`DEFAULT_AGENT_KEY_MIN_TTL_SECONDS = 1800`)

Constants from `hermes_cli/auth.py`:

- Portal URL: `https://portal.nousresearch.com`
- Inference URL: `https://inference-api.nousresearch.com/v1`
- Client ID: `hermes-cli`
- Scope: `inference:mint_agent_key`
- Poll interval cap: 1 second
- File lock timeout: 15 seconds

The `~/.hermes/auth.json` file uses `chmod 0600` permissions and cross-process file locking (fcntl on Unix, msvcrt on Windows) to prevent race conditions.

### GitHub Copilot Authentication

The `copilot_auth.py` module implements the OAuth device code flow used by the Copilot CLI:

```
Credential search order:
  1. COPILOT_GITHUB_TOKEN env var
  2. GH_TOKEN env var
  3. GITHUB_TOKEN env var
  4. gh auth token  (GitHub CLI fallback, checks multiple paths including /opt/homebrew/bin/gh)
```

**Token ambiguity note:** `GH_TOKEN` and `GITHUB_TOKEN` are commonly set for GitHub repository access (e.g., by CI systems or the `gh` CLI) rather than for Copilot/model access. If multiple GitHub-related tokens are present in the environment, the Copilot auth module may pick a repo-scoped token that lacks Copilot permissions, resulting in authentication failures. Use `COPILOT_GITHUB_TOKEN` explicitly to avoid ambiguity when other GitHub tokens are also set.

Supported token types:

| Token prefix | Type | Supported |
|-------------|------|-----------|
| `gho_` | OAuth token (default via `copilot login`) | Yes |
| `github_pat_` | Fine-grained PAT (needs Copilot Requests permission) | Yes |
| `ghu_` | GitHub App token (via env var) | Yes |
| `ghp_` | Classic PAT | **No** |

Classic PATs (`ghp_*`) are explicitly rejected with a helpful error message explaining the alternatives.

The device code login flow uses:

- Client ID: `Ov23li8tweQw6odWQebz` (same as opencode and the Copilot CLI)
- Device code URL: `https://github.com/login/device/code`
- Access token URL: `https://github.com/login/oauth/access_token`
- Scope: `read:user`
- Polling safety margin: 3 seconds added to server-specified interval

### OpenAI Codex OAuth

The Codex provider uses an external OAuth flow:

- Client ID: `app_EMoamEEZ73f0CkXaXp7hrann`
- Token URL: `https://auth.openai.com/oauth/token`
- Base URL: `https://chatgpt.com/backend-api/codex`
- Access tokens are refreshed 2 minutes before expiry

### Claude Code Credential Auto-Discovery

When the Anthropic provider is selected, Hermes reads credentials from Claude Code's stored auth if present, enabling zero-configuration use when Claude Code is already authenticated. The credential search order is: `ANTHROPIC_TOKEN` > `ANTHROPIC_API_KEY` > `CLAUDE_CODE_OAUTH_TOKEN`.

---

## PII and Secret Redaction

### Comprehensive Secret Redaction (`agent/redact.py`)

Hermes applies regex-based secret redaction across logs, tool output, and verbose output. The `redact_sensitive_text()` function masks secrets before they reach log files or gateway messages.

**Known API key prefix patterns:**

| Pattern | Service |
|---------|---------|
| `sk-*` | OpenAI / OpenRouter / Anthropic |
| `ghp_*` | GitHub PAT (classic) |
| `github_pat_*` | GitHub PAT (fine-grained) |
| `xox[baprs]-*` | Slack tokens |
| `AIza*` | Google API keys |
| `pplx-*` | Perplexity |
| `fal_*` | Fal.ai |
| `fc-*` | Firecrawl |
| `bb_live_*` | BrowserBase |
| `gAAAA*` | Codex encrypted tokens |
| `AKIA*` | AWS Access Key ID |
| `sk_live_*` / `sk_test_*` | Stripe secret keys |
| `rk_live_*` | Stripe restricted keys |
| `SG.*` | SendGrid API keys |
| `hf_*` | HuggingFace tokens |
| `r8_*` | Replicate API tokens |
| `npm_*` | npm access tokens |
| `pypi-*` | PyPI API tokens |
| `dop_v1_*` / `doo_v1_*` | DigitalOcean tokens |
| `am_*` | AgentMail API keys |
| `elevenlabs_sk_*` | ElevenLabs API keys |
| `tvly-*` | Tavily API keys |
| `exa-*` | Exa API keys |

**Additional redaction patterns:**

| Category | What is redacted |
|----------|-----------------|
| Environment variable assignments | `API_KEY=value`, `TOKEN=value`, `SECRET=value`, `PASSWORD=value` |
| JSON fields | `"apiKey": "value"`, `"token": "value"`, `"secret": "value"`, etc. |
| Authorization headers | `Authorization: Bearer <token>` |
| Telegram bot tokens | `bot<digits>:<token>` (30+ char token portion) |
| Private key blocks | `-----BEGIN RSA PRIVATE KEY-----` through `-----END RSA PRIVATE KEY-----` |
| Database connection strings | `postgres://user:PASSWORD@host`, `mongodb://`, `redis://`, `amqp://` |
| Phone numbers | E.164 format (`+1234567890`) partially masked |

**Masking behavior:** Short tokens (under 18 characters) are fully replaced with `***`. Longer tokens preserve the first 6 and last 4 characters for debuggability (e.g., `sk-or-...xyz9`).

**Disabling redaction:** Set `HERMES_REDACT_SECRETS=false` environment variable or `security.redact_secrets: false` in config.yaml.

### RedactingFormatter

The `RedactingFormatter` class in `agent/redact.py` is a `logging.Formatter` subclass that automatically applies redaction to all log messages. It is applied to the root logger so all Hermes logging output is protected.

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
  docker_forward_env: []  # Explicit allowlist -- empty keeps secrets out of container
  container_cpu: 1        # CPU cores
  container_memory: 5120  # MB (default 5 GB)
  container_disk: 51200   # MB (default 50 GB, requires overlay2 on XFS)
  container_persistent: true  # Persist filesystem across sessions
```

### Filesystem Persistence

- **Persistent mode** (`container_persistent: true`): Bind-mounts `/workspace` and `/root` from `~/.hermes/sandboxes/docker/<task_id>/`
- **Ephemeral mode** (`container_persistent: false`): Uses tmpfs for workspace -- everything is discarded on cleanup

### docker_forward_env Warning

`docker_forward_env` is an explicit allowlist of host environment variables to inject into containers. It is empty by default to keep host secrets out of containers. If you add names to this list, those variables are intentionally injected and code running in the container can read and exfiltrate them.

---

## Terminal Backend Security Comparison

| Backend | Isolation | Dangerous Cmd Check | Best For |
|---------|-----------|---------------------|----------|
| **local** | None -- runs on host | Yes | Development, trusted users |
| **ssh** | Remote machine | Yes | Running on a separate server |
| **docker** | Container | Skipped (container is boundary) | Production gateway |
| **singularity** | Container | Skipped | HPC environments |
| **modal** | Cloud sandbox | Skipped | Scalable cloud isolation |
| **daytona** | Cloud sandbox | Skipped | Persistent cloud workspaces |

For production gateway deployments, use `docker`, `modal`, or `daytona` to isolate agent commands from the host system. This eliminates the need for dangerous command approval entirely.

---

## Additional Security Hardening (v0.4.0 and v0.5.0)

The following security fixes shipped across v0.4.0 and v0.5.0:

| Change | PR | Release |
|--------|-----|---------|
| SSRF protection added to `browser_navigate` | [#3058](https://github.com/NousResearch/hermes-agent/pull/3058) | v0.5.0 |
| SSRF protection added to `vision_tools` and `web_tools` | [#2679](https://github.com/NousResearch/hermes-agent/pull/2679) | v0.4.0 |

### SSRF Protection

The SSRF checks in `browser_navigate`, `vision_tools`, and `web_tools` resolve the target hostname and reject requests to private/internal IP ranges (RFC 1918, link-local, loopback) before the HTTP connection is made.

**Known limitation -- DNS rebinding:** The SSRF pre-flight check is vulnerable to a TOCTOU (time-of-check-time-of-use) race via DNS rebinding. An attacker-controlled DNS server can return a public IP during the pre-flight resolution (passing the check), then return a private/internal IP for the actual connection by setting TTL=0 on the DNS record. This causes the subsequent HTTP request to hit an internal host despite the pre-flight passing. Mitigation: run the agent behind a DNS resolver that enforces minimum TTL (e.g., dnsmasq with `--min-cache-ttl=300`) or use container isolation so internal network access from the container is restricted at the network level.
| Restrict subagent toolsets to parent's enabled set | [#3269](https://github.com/NousResearch/hermes-agent/pull/3269) | v0.5.0 |
| Prevent zip-slip path traversal in self-update | [#3250](https://github.com/NousResearch/hermes-agent/pull/3250) | v0.5.0 |
| Prevent shell injection in `_expand_path` via `~user` path suffix | [#2685](https://github.com/NousResearch/hermes-agent/pull/2685) | v0.4.0 |
| Normalize input before dangerous command detection | [#3260](https://github.com/NousResearch/hermes-agent/pull/3260) | v0.5.0 |
| Block untrusted browser-origin API server access (CORS) | [#2451](https://github.com/NousResearch/hermes-agent/pull/2451) | v0.4.0 |
| Block sandbox backend credentials from subprocess env | [#1658](https://github.com/NousResearch/hermes-agent/pull/1658) | v0.4.0 |
| Block `@file` references from reading secrets outside workspace | [#2601](https://github.com/NousResearch/hermes-agent/pull/2601) | v0.4.0 |
| Malicious code pattern pre-exec scanner for `terminal_tool` | [#2245](https://github.com/NousResearch/hermes-agent/pull/2245) | v0.4.0 |
| PKCE verifier leak fix + OAuth refresh Content-Type | [#1775](https://github.com/NousResearch/hermes-agent/pull/1775) | v0.4.0 |
| Prevent Anthropic token leaking to third-party providers | [#2389](https://github.com/NousResearch/hermes-agent/pull/2389) | v0.4.0 |

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

The blocklist is enforced across `web_search`, `web_extract`, `browser_navigate`, and all URL-capable tools.

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

### Memory Content Scanning

Memory entries (`tools/memory_tool.py`) are also scanned before being accepted into MEMORY.md or USER.md. The scanner detects:

| Pattern ID | What it catches |
|------------|-----------------|
| `prompt_injection` | "ignore previous instructions" |
| `role_hijack` | "you are now" |
| `deception_hide` | "do not tell the user" |
| `sys_prompt_override` | "system prompt override" |
| `disregard_rules` | "disregard your instructions" |
| `bypass_restrictions` | "act as if you have no restrictions" |
| `exfil_curl` | `curl` with API key/token/secret variables |
| `exfil_wget` | `wget` with API key/token/secret variables |
| `read_secrets` | `cat .env`, `cat credentials`, `cat .netrc` |
| `ssh_backdoor` | `authorized_keys` |
| `ssh_access` | `$HOME/.ssh` |
| `hermes_env` | `$HOME/.hermes/.env` |

Invisible Unicode characters (zero-width spaces, bidirectional overrides, word joiners, byte order marks) are also blocked.

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

1. **Set explicit allowlists** -- never use `GATEWAY_ALLOW_ALL_USERS=true` in production
2. **Use container backend** -- set `terminal.backend: docker` in config.yaml
3. **Restrict resource limits** -- set appropriate CPU, memory, and disk limits in the container config
4. **Store secrets securely** -- keep API keys in `~/.hermes/.env` with `chmod 600`
5. **Enable DM pairing** -- use pairing codes instead of hardcoding user IDs
6. **Review command allowlist** -- periodically audit `command_allowlist` in config.yaml with `hermes config edit`
7. **Set `MESSAGING_CWD`** -- do not let the agent operate from sensitive directories
8. **Run as non-root** -- never run the gateway as root
9. **Monitor logs** -- check `~/.hermes/logs/` for unauthorized access attempts
10. **Keep updated** -- run `hermes update` regularly for security patches
11. **Enable tirith** -- keep `security.tirith_enabled: true` for content-level threat detection

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

## v0.6.0 Security Hardening

The following security improvements shipped in v0.6.0 (v2026.3.30):

### Hardened Dangerous Command Detection (PR #3872)

The dangerous command detection patterns in `tools/approval.py` were expanded to cover additional risky shell command variants. File tool path guards were also added for sensitive filesystem locations: writes to `/etc/`, `/boot/`, and the Docker socket (`docker.sock`) via file tools now trigger the same approval flow that was previously applied only to terminal commands.

Previously, path-based approval checks applied exclusively to `terminal_tool` invocations. With this change, `write_file`, `patch`, and other file tools check the destination path against the sensitive-path list before executing.

### Sensitive Path Write Checks for File Tools (PR #3859)

Separate from the dangerous command expansion above, the approval system was extended to catch writes to system config files through file tools. The `write_deny_list` path resolution (which uses `os.path.realpath()` to prevent symlink bypass) now applies to all file-writing tools, not just terminal commands. This closes a gap where an agent could write to `/etc/passwd` or similar protected paths via `write_file` without triggering approval.

### Expanded Secret Redaction (PR #3920)

The `agent/redact.py` secret redaction patterns were expanded to cover three new API key prefixes:

| Pattern | Service |
|---------|---------|
| `elevenlabs_sk_*` | ElevenLabs API keys |
| `tvly-*` | Tavily API keys |
| `exa-*` | Exa API keys |

These join the existing 20+ patterns in `redact_sensitive_text()`. The `RedactingFormatter` log handler applies these to all log output automatically.

### Vision File Type Enforcement (PR #3845)

The vision analysis tool now rejects non-image files passed to it. Previously, passing a PDF, text file, or binary to the vision tool would attempt analysis and potentially disclose file contents or metadata. Non-image MIME types now return an error before the file is sent to the model. This also enforces the existing website-only policy for vision analysis.

### Skill Category Path Traversal Blocking (PR #3844)

The skills system now blocks `../` sequences in category names. A crafted skill category name containing path traversal components could previously escape the skills directory. The fix normalizes and validates category names before constructing any filesystem paths, blocking directory traversal attempts.

### Configurable Approval Timeouts (PR #3886)

The dangerous command approval prompt now has a configurable timeout. Previously the CLI prompt always timed out after 60 seconds and auto-denied. The timeout is now configurable via `approvals.timeout` in config.yaml:

```yaml
approvals:
  timeout: 60    # Default: 60 seconds
```

See [configuration.md](configuration.md) for the full config.yaml reference.

---

## v0.8.0 Security Hardening

The following security improvements shipped in v0.8.0 (v2026.4.8):

### Consolidated Security Hardening (PR #5944)

A single PR consolidated multiple SSRF, timing, and traversal fixes across the codebase:

- **SSRF consolidation** — SSRF protections are now applied uniformly across all URL-fetching paths, not just `browser_navigate` and `vision_tools`.
- **Timing attack mitigations** — constant-time comparisons used where timing-sensitive checks were previously variable.
- **Tar traversal prevention** — archives unpacked during self-update and skill install are now checked for path traversal entries (e.g. `../`) before extraction.
- **Credential leakage guards** — additional scrubbing applied to error messages and log lines that might contain credential-like patterns.

### Cross-Session Isolation and Cron Path Traversal (PR #5613)

- **Cross-session isolation** — session state is scoped more tightly to prevent one session from reading another's in-progress state under concurrent gateway load.
- **Cron path traversal hardening** — cron job IDs and output paths are validated and normalized before use in filesystem operations, blocking crafted job IDs from escaping the cron output directory.

### Terminal Workdir Sanitization (PR #5629)

The `workdir` parameter to the terminal tool is now sanitized before being passed to any backend (local, Docker, SSH, Modal, Daytona). This prevents injected path components from redirecting terminal operations to unintended directories.

### MCP Package OSV Malware Scanning (PR #5305)

Before spawning a stdio MCP server via `npx` or `uvx`, Hermes queries the OSV (Open Source Vulnerabilities) API to check whether the package has known malware advisories (MAL-* IDs). Regular CVEs are not blocked — only confirmed malware triggers a block. The check is fail-open: network errors, timeouts, and unrecognized package formats allow the launch to proceed. See [mcp.md](mcp.md) for configuration details.

### Approval Session Escalation Prevention (PR #5280)

`allow-once` approval grants are now strictly session-scoped and cannot be escalated to permanent `allow-always` state by a subsequent crafted approval request. Cron delivery platform names are validated against a known platform list to prevent environment variable enumeration via crafted delivery targets.

---

## Contributing Security-Sensitive Code

When writing code that touches security boundaries:

- Always use `shlex.quote()` when interpolating user input into shell commands
- Resolve symlinks with `os.path.realpath()` before path-based access control checks
- Do not log secrets -- API keys, tokens, and passwords must never appear in log output
- Catch broad exceptions around tool execution so a single failure does not crash the agent loop
- Test on all platforms if your change touches file paths, process management, or shell commands

Existing protections to be aware of:

| Layer | Implementation |
|-------|---------------|
| Sudo password piping | Uses `shlex.quote()` to prevent shell injection |
| Dangerous command detection | Regex patterns in `tools/approval.py` with user approval flow |
| Tirith pre-exec scanning | Content-level threat detection via `tools/tirith_security.py`; block verdicts are now approvable (v0.5.0, [#3428](https://github.com/NousResearch/hermes-agent/pull/3428)) |
| Smart approval | Auxiliary LLM risk assessment (`tools/approval.py`) |
| Cron prompt injection | Scanner in `tools/cronjob_tools.py` blocks instruction-override patterns |
| Write deny list | Protected paths resolved via `os.path.realpath()` to prevent symlink bypass |
| Skills guard | Security scanner for hub-installed skills (`tools/skills_guard.py`) |
| Code execution sandbox | `execute_code` child process runs with API keys stripped from environment |
| Container hardening | Docker: all capabilities dropped, no privilege escalation, PID limits, size-limited tmpfs |
| PII redaction | `agent/redact.py` masks 20+ secret patterns in all log output |
| Memory content scanning | `tools/memory_tool.py` blocks injection/exfiltration in memory entries |
| Supply chain hardening | Pinned deps, `uv.lock` with hashes, litellm removed, CI PR scanning (v0.5.0, [#2796](https://github.com/NousResearch/hermes-agent/pull/2796)–[#3073](https://github.com/NousResearch/hermes-agent/pull/3073)) |
| SSRF protection | `browser_navigate`, `vision_tools`, `web_tools` block internal network requests; consolidated across all URL-fetching paths (v0.4.0–v0.8.0) |
| Subagent toolset restriction | Subagents inherit only the parent's enabled toolset (v0.5.0, [#3269](https://github.com/NousResearch/hermes-agent/pull/3269)) |
| Timing attack mitigations | Constant-time comparisons for timing-sensitive checks (v0.8.0, [#5944](https://github.com/NousResearch/hermes-agent/pull/5944)) |
| Tar traversal prevention | Path traversal entries in archives blocked before extraction (v0.8.0, [#5944](https://github.com/NousResearch/hermes-agent/pull/5944)) |
| Cron path traversal hardening | Cron job IDs and output paths normalized and validated (v0.8.0, [#5613](https://github.com/NousResearch/hermes-agent/pull/5613)) |
| Cross-session isolation | Session state scoped to prevent cross-session reads under concurrent load (v0.8.0, [#5613](https://github.com/NousResearch/hermes-agent/pull/5613)) |
| Terminal workdir sanitization | `workdir` parameter sanitized across all terminal backends (v0.8.0, [#5629](https://github.com/NousResearch/hermes-agent/pull/5629)) |
| MCP OSV malware scanning | stdio MCP packages scanned via OSV API before spawn; fail-open (v0.8.0, [#5305](https://github.com/NousResearch/hermes-agent/pull/5305)) |
| Approval session escalation | `allow-once` grants strictly session-scoped; cannot be escalated to `allow-always` (v0.8.0, [#5280](https://github.com/NousResearch/hermes-agent/pull/5280)) |

If your PR affects security, note it explicitly in the PR description.
