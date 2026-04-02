# Code Execution

The `execute_code` tool lets the agent write Python scripts that call Hermes tools programmatically, collapsing multi-step workflows into a single LLM turn. The script runs in a sandboxed child process on the agent host, communicating via Unix domain socket RPC.

Added in v0.3.0. Available on Linux and macOS only (requires Unix domain sockets). Automatically disabled on Windows.

## How It Works

1. The agent writes a Python script using `from hermes_tools import ...`
2. Hermes generates a `hermes_tools.py` stub module with RPC functions for each allowed tool
3. Hermes opens a Unix domain socket and starts an RPC listener thread
4. The script runs in a child process -- tool calls travel over the socket back to Hermes
5. Only the script's `print()` output is returned to the LLM; intermediate tool results never enter the context window

```python
from hermes_tools import web_search, web_extract

results = web_search("Python 3.13 features", limit=5)
for r in results["data"]["web"]:
    content = web_extract([r["url"]])
    # ... filter and process ...
print(summary)
```

## Available Tools in Sandbox

The sandbox exposes a fixed set of 7 tools. The intersection of this list and the session's enabled tools determines which stubs are generated:

- `web_search`
- `web_extract`
- `read_file`
- `write_file`
- `search_files`
- `patch`
- `terminal` (foreground only -- no `background`, `pty`, or `check_interval` parameters)

Tool calls inside scripts behave identically to normal tool calls -- same rate limits, same error handling, same capabilities. The only restriction is that `terminal()` is foreground-only.

## When the Agent Uses This

The agent uses `execute_code` when there are:

- **3+ tool calls** with processing logic between them
- Bulk data filtering or conditional branching
- Loops over results

The key benefit: intermediate tool results never enter the context window -- only the final `print()` output comes back, dramatically reducing token usage.

## Practical Examples

### Data Processing Pipeline

```python
from hermes_tools import search_files, read_file
import json

matches = search_files("database", path=".", file_glob="*.yaml", limit=20)
configs = []
for match in matches.get("matches", []):
    content = read_file(match["path"])
    configs.append({"file": match["path"], "preview": content["content"][:200]})

print(json.dumps(configs, indent=2))
```

### Multi-Step Web Research

```python
from hermes_tools import web_search, web_extract
import json

results = web_search("Rust async runtime comparison 2025", limit=5)
summaries = []
for r in results["data"]["web"]:
    page = web_extract([r["url"]])
    for p in page.get("results", []):
        if p.get("content"):
            summaries.append({
                "title": r["title"],
                "url": r["url"],
                "excerpt": p["content"][:500]
            })

print(json.dumps(summaries, indent=2))
```

### Bulk File Refactoring

```python
from hermes_tools import search_files, read_file, patch

matches = search_files("old_api_call", path="src/", file_glob="*.py")
fixed = 0
for match in matches.get("matches", []):
    result = patch(
        path=match["path"],
        old_string="old_api_call(",
        new_string="new_api_call(",
        replace_all=True
    )
    if "error" not in str(result):
        fixed += 1

print(f"Fixed {fixed} files out of {len(matches.get('matches', []))} matches")
```

### Build and Test Pipeline

```python
from hermes_tools import terminal, read_file
import json

result = terminal("cd /project && python -m pytest --tb=short -q 2>&1", timeout=120)
output = result.get("output", "")

passed = output.count(" passed")
failed = output.count(" failed")
errors = output.count(" error")

report = {
    "passed": passed,
    "failed": failed,
    "errors": errors,
    "exit_code": result.get("exit_code", -1),
    "summary": output[-500:] if len(output) > 500 else output
}

print(json.dumps(report, indent=2))
```

## Resource Limits

| Resource | Limit | Notes |
|----------|-------|-------|
| **Timeout** | 5 minutes (300s) | Script is killed with SIGTERM, then SIGKILL after 5s grace |
| **Stdout** | 50 KB | Output truncated with `[output truncated at 50KB]` notice |
| **Stderr** | 10 KB | Included in output on non-zero exit for debugging |
| **Tool calls** | 50 per execution | Error returned when limit reached |

All limits are configurable via `config.yaml`:

```yaml
code_execution:
  timeout: 300       # Max seconds per script (default: 300)
  max_tool_calls: 50 # Max tool calls per execution (default: 50)
```

## Error Handling

When a script fails, the agent receives structured error information:

- **Non-zero exit code** -- stderr is included in the output so the agent sees the full traceback
- **Timeout** -- script is killed and the agent sees `"Script timed out after 300s and was killed."`
- **Interruption** -- if the user sends a new message during execution, the script is terminated and the agent sees `[execution interrupted -- user sent a new message]`
- **Tool call limit** -- when the 50-call limit is hit, subsequent tool calls return an error message

The response always includes `status` (success/error/timeout/interrupted), `output`, `tool_calls_made`, and `duration_seconds`.

## Security

The child process runs with a **minimal environment**. API keys, tokens, and credentials are stripped by default. The script accesses tools exclusively via the RPC channel -- it cannot read secrets from environment variables unless explicitly allowed.

Environment variables containing `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `CREDENTIAL`, `PASSWD`, or `AUTH` in their names are excluded. Only safe system variables (`PATH`, `HOME`, `LANG`, `SHELL`, `PYTHONPATH`, `VIRTUAL_ENV`, etc.) are passed through.

The script runs in a temporary directory that is cleaned up after execution. The child process runs in its own process group so it can be cleanly killed on timeout or interruption.

### Skill Environment Variable Passthrough

When a skill declares `required_environment_variables` in its frontmatter, those variables are automatically passed through to both `execute_code` and `terminal` sandboxes after the skill is loaded. This lets skills use their declared API keys without weakening the security posture for arbitrary code.

For non-skill use cases, you can explicitly allowlist variables in `config.yaml`:

```yaml
terminal:
  env_passthrough:
    - MY_CUSTOM_KEY
    - ANOTHER_TOKEN
```

## execute_code vs terminal

| Use Case | execute_code | terminal |
|----------|-------------|----------|
| Multi-step workflows with tool calls between | Yes | No |
| Simple shell command | No | Yes |
| Filtering/processing large tool outputs | Yes | No |
| Running a build or test suite | No | Yes |
| Looping over search results | Yes | No |
| Interactive/background processes | No | Yes |
| Needs API keys in environment | Only via passthrough config | Yes (most pass through) |

**Rule of thumb:** Use `execute_code` when you need to call Hermes tools programmatically with logic between calls. Use `terminal` for running shell commands, builds, and processes.
