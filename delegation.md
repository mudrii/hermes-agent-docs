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
| `delegation` | No recursive delegation by default; opt in by raising `delegation.max_spawn_depth` (and optionally toggling `delegation.orchestrator_enabled`) |
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

## Depth Limit and the Orchestrator Role (v0.11.0)

Delegation depth is **configurable** through `delegation.max_spawn_depth` (range `1`-`3`, default `1`). Depth `1` means a parent (depth 0) can spawn children (depth 1) but children cannot delegate further — this is the historical default and matches the pre-v0.11.0 behaviour. Depth `2` and `3` allow a parent to spawn grandchildren and great-grandchildren respectively. The cap is enforced in `tools/delegate_tool.py`; attempting to delegate beyond `max_spawn_depth` returns an error.

v0.11.0 also introduces an explicit **orchestrator role** (`delegation.orchestrator_enabled`). When enabled, the parent is told it is an orchestrator and that its children may themselves delegate, and Hermes coordinates **cross-agent file state** so that children writing to the same shared workspace see each other's writes via the session lineage. Combined with `max_spawn_depth > 1`, this turns `delegate_task` into a true multi-tier coordination primitive while preserving context isolation at each tier.

| Setting | Range | Default | Effect |
|---------|-------|---------|--------|
| `delegation.max_spawn_depth` | 1-3 | 1 | Caps how deep recursive delegation can go |
| `delegation.orchestrator_enabled` | bool | true | Kill switch for the orchestrator role (orchestrator framing + cross-agent file coordination). Set `false` to force every parent into a leaf, even with `max_spawn_depth > 1`. |

## Key Properties

- Each subagent gets its **own terminal session** (separate from the parent's session and from other subagents)
- **Nested delegation is opt-in (v0.11.0)** — by default children cannot spawn grandchildren; raise `delegation.max_spawn_depth` to permit it. `delegation.orchestrator_enabled` defaults to `true` and only needs flipping to `false` if you want to force-disable orchestrator framing entirely.
- **Interrupt propagation** — interrupting the parent interrupts all active children via the `_active_children` registry
- Only the final summary enters the parent's context, keeping token usage efficient
- Subagents inherit the parent's **API key and provider configuration** unless overridden in delegation config
- **v0.5.0 (PR #3004):** Subagents now have **independent iteration budgets**. Each subagent gets its own `max_iterations` counter (default: 50 per `delegation.max_iterations`). Previously, subagents shared the parent's budget, causing them to hit the limit prematurely when the parent had already consumed many turns.
- **v0.8.0 (PR #5309):** Subagent sessions are **linked to the parent session** via `parent_session_id` and **hidden from the session list** by default. `hermes sessions list` and `session_search` exclude child sessions unless you pass `include_children=True` to the `SessionDB.list_sessions()` API.

## Configuration

```yaml
# In ~/.hermes/config.yaml
delegation:
  max_iterations: 50                        # Max turns per child (default: 50). v0.5.0: each subagent gets its own independent budget.
  max_spawn_depth: 1                        # v0.11.0: how deep recursive delegation can go (1-3, default 1)
  orchestrator_enabled: true                # v0.11.0: orchestrator role (default true; set false to force every parent into a leaf)
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

---

## Third-Party Session Isolation (v0.5.0 — PR #3255)

The `--source` flag on `hermes chat` isolates sessions by their origin label. When subagents or tools spawn child processes with `--source <label>`, their sessions are stored separately and do not mix with interactive sessions in `hermes sessions list` or `session_search`.

```bash
hermes chat --source my-automation -q "Run the nightly checks"
```

This is useful when scripts or CI pipelines invoke Hermes alongside interactive use — the automated sessions stay isolated from the interactive history.

---

## @ Context References as Delegation Input (v0.4.0 — PR #2343, #2482)

You can use `@file`, `@url`, `@diff`, and `@folder` references in the `context` field passed to `delegate_task`. The references are expanded before the subagent receives them:

```python
delegate_task(
    goal="Review the recent changes and write a summary",
    context="Here is the current diff:\n@diff\n\nMain file: @file:src/main.py"
)
```

The expansion happens in the parent before the subagent starts, so the subagent receives the injected content directly — no `@` syntax reaches the subagent.

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
