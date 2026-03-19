# Subagent Delegation

The `delegate_task` tool spawns child AIAgent instances with isolated context, restricted toolsets, and their own terminal sessions. Each child gets a fresh conversation and works independently — only its final summary enters the parent's context.

This enables two powerful patterns:
- **Parallel workstreams** — run multiple independent tasks simultaneously (up to 3 at once)
- **Context isolation** — keep large amounts of intermediate tool output out of the parent's context window

## Single Task

```python
delegate_task(
    goal="Debug why tests fail",
    context="Error: assertion in test_foo.py line 42",
    toolsets=["terminal", "file"]
)
```

## Parallel Batch

Up to 3 concurrent subagents:

```python
delegate_task(tasks=[
    {"goal": "Research topic A", "toolsets": ["web"]},
    {"goal": "Research topic B", "toolsets": ["web"]},
    {"goal": "Fix the build", "toolsets": ["terminal", "file"]}
])
```

## How Subagent Context Works

Subagents start with a **completely fresh conversation**. They have zero knowledge of the parent's conversation history, prior tool calls, or anything discussed before delegation. The subagent's only context comes from the `goal` and `context` fields you provide.

This means you must pass **everything** the subagent needs:

```python
# BAD — subagent has no idea what "the error" is
delegate_task(goal="Fix the error")

# GOOD — subagent has all context it needs
delegate_task(
    goal="Fix the TypeError in api/handlers.py",
    context="""The file api/handlers.py has a TypeError on line 47:
    'NoneType' object has no attribute 'get'.
    The function process_request() receives a dict from parse_body(),
    but parse_body() returns None when Content-Type is missing.
    The project is at /home/user/myproject and uses Python 3.11."""
)
```

The subagent receives a focused system prompt built from your goal and context, instructing it to complete the task and provide a structured summary of what it did, what it found, any files modified, and any issues encountered.

## Practical Examples

### Parallel Research

Research multiple topics simultaneously:

```python
delegate_task(tasks=[
    {
        "goal": "Research the current state of WebAssembly in 2025",
        "context": "Focus on: browser support, non-browser runtimes, language support",
        "toolsets": ["web"]
    },
    {
        "goal": "Research the current state of RISC-V adoption in 2025",
        "context": "Focus on: server chips, embedded systems, software ecosystem",
        "toolsets": ["web"]
    },
    {
        "goal": "Research quantum computing progress in 2025",
        "context": "Focus on: error correction breakthroughs, practical applications, key players",
        "toolsets": ["web"]
    }
])
```

### Code Review and Fix

Delegate a review-and-fix workflow to a fresh context:

```python
delegate_task(
    goal="Review the authentication module for security issues and fix any found",
    context="""Project at /home/user/webapp.
    Auth module files: src/auth/login.py, src/auth/jwt.py, src/auth/middleware.py.
    The project uses Flask, PyJWT, and bcrypt.
    Focus on: SQL injection, JWT validation, password handling, session management.
    Fix any issues found and run the test suite (pytest tests/auth/).""",
    toolsets=["terminal", "file"]
)
```

### Multi-File Refactoring

Delegate a large refactoring task that would flood the parent's context:

```python
delegate_task(
    goal="Refactor all Python files in src/ to replace print() with proper logging",
    context="""Project at /home/user/myproject.
    Use the 'logging' module with logger = logging.getLogger(__name__).
    Replace print() calls with appropriate log levels:
    - print(f"Error: ...") -> logger.error(...)
    - print(f"Warning: ...") -> logger.warning(...)
    - print(f"Debug: ...") -> logger.debug(...)
    - Other prints -> logger.info(...)
    Don't change print() in test files or CLI output.
    Run pytest after to verify nothing broke.""",
    toolsets=["terminal", "file"]
)
```

## Batch Mode Details

When you provide a `tasks` array:

- **Maximum concurrency:** 3 tasks (arrays longer than 3 are truncated to 3)
- **Thread pool:** Uses `ThreadPoolExecutor` with `MAX_CONCURRENT_CHILDREN = 3` workers
- **Progress display:** In CLI mode, a tree-view shows tool calls from each subagent in real-time with per-task completion lines. In gateway mode, progress is batched and relayed to the parent's progress callback
- **Result ordering:** Results are sorted by task index to match input order regardless of completion order
- **Interrupt propagation:** Interrupting the parent (sending a new message) interrupts all active children

Single-task delegation runs directly without thread pool overhead.

## Model Override

Use a different model for subagents — useful for delegating simple tasks to cheaper/faster models:

```python
delegate_task(
    goal="Summarize this README file",
    context="File at /project/README.md",
    toolsets=["file"],
    model="google/gemini-flash-2.0"
)
```

If omitted, subagents use the same model as the parent.

## Toolset Selection

The `toolsets` parameter controls what tools the subagent has access to:

| Toolset Pattern | Use Case |
|----------------|----------|
| `["terminal", "file"]` | Code work, debugging, file editing, builds |
| `["web"]` | Research, fact-checking, documentation lookup |
| `["terminal", "file", "web"]` | Full-stack tasks (default) |
| `["file"]` | Read-only analysis, code review without execution |
| `["terminal"]` | System administration, process management |

Toolsets that are **always blocked** for subagents regardless of what you specify:

| Blocked toolset | Reason |
|----------------|--------|
| `delegation` | No recursive delegation (prevents infinite spawning) |
| `clarify` | Subagents cannot interact with the user |
| `memory` | No writes to shared persistent memory |
| `code_execution` | Children should reason step-by-step |
| `send_message` | No cross-platform side effects |

If no toolsets are specified, the subagent inherits the parent's enabled toolsets (with blocked toolsets removed).

## Max Iterations

Each subagent has an iteration limit (default: 50) controlling how many tool-calling turns it can take:

```python
delegate_task(
    goal="Quick file check",
    context="Check if /etc/nginx/nginx.conf exists and print its first 10 lines",
    max_iterations=10
)
```

## Depth Limit

Delegation has a **depth limit of 2** — a parent (depth 0) can spawn children (depth 1), but children cannot delegate further. Attempting to call `delegate_task` from a subagent returns an error.

Constant in source: `MAX_DEPTH = 2` in `tools/delegate_tool.py`.

## Key Properties

- Each subagent gets its **own terminal session** (separate from the parent's session and from other subagents)
- **No nested delegation** — children cannot spawn grandchildren
- **Interrupt propagation** — interrupting the parent interrupts all active children via the `_active_children` registry
- Only the final summary enters the parent's context, keeping token usage efficient
- Subagents inherit the parent's **API key and provider configuration** unless overridden in delegation config
- Subagents share the parent's **iteration budget** via `shared_budget = parent_agent.iteration_budget`

## Configuration

```yaml
# In ~/.hermes/config.yaml
delegation:
  max_iterations: 50                        # Max turns per child (default: 50)
  default_toolsets: ["terminal", "file", "web"]  # Default toolsets
  model: "google/gemini-3-flash-preview"             # Optional model override
  provider: "openrouter"                             # Optional provider override

# Or use a direct custom endpoint instead of a named provider:
delegation:
  model: "qwen2.5-coder"
  base_url: "http://localhost:1234/v1"
  api_key: "local-key"
```

When `delegation.provider` is configured, subagents resolve the full credential bundle (base_url, api_key, api_mode) through the same runtime provider system used by CLI/gateway startup. Supported provider names: `openrouter`, `nous`, `zai`, `kimi-coding`, `minimax`.

When `delegation.base_url` is set instead of `delegation.provider`, the subagent connects directly to that OpenAI-compatible endpoint. The API key is taken from `delegation.api_key` or falls back to `OPENAI_API_KEY`. Special URL patterns are auto-detected: ChatGPT Codex URLs use the `codex_responses` API mode, and `api.anthropic.com` URLs use `anthropic_messages` mode.

## Delegation vs execute_code

| Factor | delegate_task | execute_code |
|--------|--------------|-------------|
| **Reasoning** | Full LLM reasoning loop | Just Python code execution |
| **Context** | Fresh isolated conversation | No conversation, just script |
| **Tool access** | All non-blocked tools with reasoning | 7 tools via RPC, no reasoning |
| **Parallelism** | Up to 3 concurrent subagents | Single script |
| **Best for** | Complex tasks needing judgment | Mechanical multi-step pipelines |
| **Token cost** | Higher (full LLM loop) | Lower (only stdout returned) |
| **User interaction** | None (subagents can't clarify) | None |

Use `delegate_task` when the subtask requires reasoning, judgment, or multi-step problem solving. Use `execute_code` when you need mechanical data processing or scripted workflows.
