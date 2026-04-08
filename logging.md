# Logging

Centralized, structured logging was introduced in v0.8.0 (PR [#5430](https://github.com/NousResearch/hermes-agent/pull/5430)). Log files are written to `~/.hermes/logs/` and rotate automatically. All log output passes through `RedactingFormatter` so secrets are never written to disk.

---

## Log Files

| File | Level | Purpose |
|------|-------|---------|
| `agent.log` | INFO and above | Main activity log -- tool calls, session events, provider switches, compression events |
| `errors.log` | WARNING and above | Quick triage log -- errors and warnings only |

Both files use `RotatingFileHandler`. The defaults are 5 MB max per file with 3 rotated backups kept. `errors.log` uses a smaller 2 MB limit with 2 backups.

---

## Initialization

Logging is initialized in `AIAgent.__init__()` via `hermes_logging.setup_logging()`. It is also called early in the CLI and gateway startup paths. The call is idempotent -- calling it multiple times is safe (gateway mode creates a new `AIAgent` per message).

`setup_logging()` always attaches the file handlers regardless of console quiet mode.

---

## Console Output Modes

### Quiet Mode (CLI default)

When `AIAgent` is constructed with `quiet_mode=True` (the CLI default), console output for these logger namespaces is suppressed at `ERROR` level:

- `tools` -- all `tools.*` subloggers (terminal, browser, web, file, etc.)
- `run_agent` -- agent runner internals
- `trajectory_compressor`
- `cron` -- scheduler (only relevant in daemon mode)
- `hermes_cli` -- CLI helpers

File handlers (`agent.log` and `errors.log`) are unaffected -- they still capture INFO and WARNING respectively, regardless of quiet mode. This means logs are always available for post-hoc debugging even when the terminal is clean.

### Verbose Mode (`--verbose` / `-v`)

Passing `--verbose` to `hermes chat` sets `verbose_logging=True` on `AIAgent`, which calls `hermes_logging.setup_verbose_logging()`. This adds a DEBUG-level console `StreamHandler`. Third-party libraries (`openai`, `httpx`, `httpcore`, `asyncio`, `modal`, `websockets`, etc.) remain at WARNING to reduce noise -- they are listed in `_NOISY_LOGGERS` in `hermes_logging.py`.

---

## Optional Config Keys

The logging subsystem supports optional overrides in `config.yaml`:

```yaml
logging:
  level: "INFO"           # Minimum level for agent.log. Default: INFO.
  max_size_mb: 5          # Max file size before rotation (MB). Default: 5.
  backup_count: 3         # Rotated backup files to keep. Default: 3.
```

These are read best-effort at startup. If `config.yaml` does not include a `logging:` section, the defaults above apply.

---

## `hermes logs` Command

The `hermes logs` command (added in v0.8.0, `hermes_cli/logs.py`) provides viewing, filtering, and following of log files without leaving the terminal.

### Basic Usage

```bash
hermes logs                    # Last 50 lines of agent.log
hermes logs errors             # Last 50 lines of errors.log
hermes logs gateway            # Last 50 lines of gateway.log
hermes logs list               # List available log files with sizes and modification times
```

### Options

| Option | Description |
|--------|-------------|
| `log_name` | Which log to view: `agent` (default), `errors`, `gateway`, or `list` |
| `-n N` / `--lines N` | Number of recent lines to show (default: 50) |
| `-f` / `--follow` | Follow the log in real time (like `tail -f`) |
| `--level LEVEL` | Minimum log level: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` |
| `--session ID` | Filter lines containing this session ID substring |
| `--since TIME` | Show lines from the last N time units: `1h`, `30m`, `2d` |

### Examples

```bash
hermes logs -f                         # Follow agent.log in real time (Ctrl+C to stop)
hermes logs errors                     # Show last 50 lines of errors.log
hermes logs gateway -n 100             # Show last 100 lines of gateway.log
hermes logs --level WARNING            # Only show WARNING and above from agent.log
hermes logs --session abc123           # Filter by session ID substring
hermes logs --since 1h                 # Lines from the last hour
hermes logs --since 30m -f             # Follow, starting from 30 minutes ago
```

### Implementation Notes

- `hermes logs list` shows each `.log` file's size, modification time, and age
- When filters are active (`--level`, `--session`, `--since`), the command reads up to 10x the requested line count (`max(N * 20, 2000)`) and filters down to the requested number
- For files under 1 MB the entire file is read (simple and correct); larger files are read in reverse chunks from the end for efficiency
- Follow mode polls the file every 0.3 seconds; press Ctrl+C to stop

---

## Log Rotation

Both `agent.log` and `errors.log` use Python's `RotatingFileHandler`. When a file reaches its size limit, it is renamed to `agent.log.1`, `agent.log.2`, etc. up to `backup_count` backups. The active file is always `agent.log` / `errors.log`.

Rotated files are not shown by `hermes logs` -- the command always reads the current active file.

---

## Secret Redaction

All log output is processed by `RedactingFormatter` from `agent/redact.py`. This strips API keys, tokens, and other credential patterns before writing to disk. The patterns matched include known key prefixes (`sk-`, `sk-or-`, `sk-ant-`, OAuth tokens, etc.) and common environment variable patterns. See `agent/redact.py` for the full pattern list.

---

## Logging in Gateway Mode

The gateway calls `setup_logging(mode="gateway")` early in its startup path. This is the same `setup_logging()` call used by the CLI. `setup_logging()` attaches only file handlers (`RotatingFileHandler`) regardless of caller — it never adds a `StreamHandler` for any code path. Console output is only enabled when `setup_verbose_logging()` is called, which happens when the CLI receives `--verbose` / `-v`. Gateway mode never calls `setup_verbose_logging()`, so all output goes to `agent.log` and `errors.log` only. Check `journalctl -u hermes-agent` for systemd service output.

---

## Source Files

| File | Description |
|------|-------------|
| `hermes_logging.py` | `setup_logging()`, `setup_verbose_logging()`, `_NOISY_LOGGERS` |
| `hermes_cli/logs.py` | `tail_log()`, `list_logs()`, `hermes logs` command implementation |
| `agent/redact.py` | `RedactingFormatter` -- secret masking for log output |
