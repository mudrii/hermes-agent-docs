# Tools Reference

Hermes Agent includes 48 built-in tools organized into toolsets. Tools are registered at import time via a central registry and exposed to LLMs as OpenAI-format function definitions.

---

## Web & Research

| Tool | Toolset | Description |
|------|---------|-------------|
| `web_search` | `web` | Search the web for information on any topic |
| `web_extract` | `web` | Extract content from URLs (HTML, PDFs, Markdown) |

### web_search

```json
{
  "name": "web_search",
  "parameters": {
    "query": "string — search query",
    "max_results": "integer — number of results (default: 10)"
  },
  "returns": "Array of {title, url, description}"
}
```

### web_extract

```json
{
  "name": "web_extract",
  "parameters": {
    "url": "string — URL to fetch",
    "mode": "string — content | markdown | brief"
  },
  "returns": "Markdown-formatted page content"
}
```

**Requires:** `FIRECRAWL_API_KEY` (falls back to basic HTTP if not set)

---

## Browser Automation (11 tools)

All browser tools operate on a single browser session per agent run. Use `browser_navigate` first, then `browser_snapshot` to get element reference IDs, then interact.

| Tool | Toolset | Description |
|------|---------|-------------|
| `browser_navigate` | `browser` | Navigate to a URL |
| `browser_snapshot` | `browser` | Get accessibility tree (element refs) |
| `browser_click` | `browser` | Click element by ref ID |
| `browser_type` | `browser` | Type into input by ref ID |
| `browser_scroll` | `browser` | Scroll page |
| `browser_back` | `browser` | Navigate back |
| `browser_press` | `browser` | Press keyboard key |
| `browser_close` | `browser` | Close browser session |
| `browser_console` | `browser` | Get JS console output |
| `browser_get_images` | `browser` | List all images on page |
| `browser_vision` | `browser` | Screenshot + vision AI analysis |

### Interaction Pattern

```
browser_navigate(url)
  → browser_snapshot()          # Returns elements with ref IDs like "@e5"
  → browser_click(ref="@e5")    # Click by ref
  → browser_type(ref="@e12", text="hello")
  → browser_press(key="Enter")
```

**Optional:** `BROWSERBASE_API_KEY` for remote browser sessions

---

## Terminal & Process Management

| Tool | Toolset | Description |
|------|---------|-------------|
| `terminal` | `terminal` | Execute shell commands |
| `process` | `terminal` | Manage background processes |

### terminal

```json
{
  "name": "terminal",
  "parameters": {
    "command": "string — shell command to execute",
    "cwd": "string — working directory (optional)",
    "timeout": "integer — seconds before timeout"
  },
  "returns": "{stdout, stderr, exit_code}"
}
```

**Backends:** local, docker, ssh, modal, daytona, singularity (configured via `terminal.backend`)

### process

```json
{
  "name": "process",
  "parameters": {
    "action": "string — list | poll | log | wait | kill | write",
    "pid": "string — process ID (for non-list actions)",
    "input": "string — stdin input (for write action)"
  },
  "returns": "Process status and output"
}
```

---

## File Operations

| Tool | Toolset | Description |
|------|---------|-------------|
| `read_file` | `file` | Read text file with line numbers |
| `write_file` | `file` | Write content to file (overwrites) |
| `patch` | `file` | Targeted find-and-replace edits |
| `search_files` | `file` | Search file contents or find files |

### read_file

```json
{
  "name": "read_file",
  "parameters": {
    "path": "string — file path",
    "offset": "integer — start line (optional)",
    "limit": "integer — max lines to read (optional)"
  },
  "returns": "File content with line numbers; suggests similar filenames if not found"
}
```

### write_file

```json
{
  "name": "write_file",
  "parameters": {
    "path": "string — file path",
    "content": "string — complete file content"
  },
  "returns": "Success confirmation; creates parent directories automatically"
}
```

### patch

```json
{
  "name": "patch",
  "parameters": {
    "path": "string — file path",
    "old_string": "string — text to find",
    "new_string": "string — replacement text"
  },
  "returns": "Unified diff of changes; uses 9-strategy fuzzy matching"
}
```

### search_files

```json
{
  "name": "search_files",
  "parameters": {
    "query": "string — search pattern (regex supported)",
    "path": "string — directory to search (optional)",
    "mode": "string — content | files | count",
    "glob": "string — file pattern filter (optional)"
  },
  "returns": "Matching file paths or content lines"
}
```

**Backed by ripgrep for performance.**

---

## Vision & Media

| Tool | Toolset | Description |
|------|---------|-------------|
| `vision_analyze` | `vision` | Analyze images with AI vision |
| `image_generate` | `image_gen` | Generate images from text prompts |
| `text_to_speech` | `tts` | Convert text to audio |

### vision_analyze

```json
{
  "name": "vision_analyze",
  "parameters": {
    "image_path": "string — local path or URL",
    "question": "string — what to analyze"
  },
  "returns": "Natural language description or answer"
}
```

### image_generate

```json
{
  "name": "image_generate",
  "parameters": {
    "prompt": "string — image description",
    "model": "string — model identifier (optional)"
  },
  "returns": "Path to generated image"
}
```

**Powered by FLUX 2 Pro with 2x upscaling via `fal.ai`**. Requires `FAL_KEY`.

### text_to_speech

```json
{
  "name": "text_to_speech",
  "parameters": {
    "text": "string — text to speak",
    "voice": "string — voice name (optional)",
    "provider": "string — edge | elevenlabs | openai"
  },
  "returns": "Path to audio file"
}
```

---

## Planning & Task Management

| Tool | Toolset | Description |
|------|---------|-------------|
| `todo` | `todo` | Manage task list for current session |
| `clarify` | `clarify` | Ask user clarifying questions |

### todo

```json
{
  "name": "todo",
  "parameters": {
    "action": "string — create | update | read | merge",
    "items": "array — task items",
    "id": "string — task ID (for update)"
  },
  "returns": "Current task list state"
}
```

Used for complex multi-step tasks to track progress across iterations.

### clarify

```json
{
  "name": "clarify",
  "parameters": {
    "question": "string — question to ask user",
    "choices": "array — up to 4 options (optional)",
    "allow_free_text": "boolean — allow custom answer beyond choices"
  },
  "returns": "User's response"
}
```

**Note:** Clarify forces sequential tool execution mode (pauses other parallel tools until user responds).

---

## Memory & Context

| Tool | Toolset | Description |
|------|---------|-------------|
| `memory` | `memory` | Save information to persistent memory |
| `session_search` | `session_search` | Search past conversation history |

### memory

```json
{
  "name": "memory",
  "parameters": {
    "action": "string — save | read | delete",
    "key": "string — memory key",
    "value": "string — content to save"
  },
  "returns": "Confirmation or retrieved value"
}
```

Saved memories appear in the system prompt at the start of each new session.

### session_search

```json
{
  "name": "session_search",
  "parameters": {
    "query": "string — what to search for",
    "max_results": "integer — number of sessions to include (default: 5)",
    "context_messages": "integer — messages before/after match to include"
  },
  "returns": "Summarized excerpts from matching past conversations"
}
```

Uses SQLite FTS5 for fast full-text search, then summarizes results with an auxiliary LLM.

---

## Agent Orchestration

| Tool | Toolset | Description |
|------|---------|-------------|
| `execute_code` | `code_execution` | Run Python scripts with tool access |
| `delegate_task` | `delegation` | Spawn isolated subagents |
| `mixture_of_agents` | `moa` | Route through multiple frontier LLMs |

### execute_code

```json
{
  "name": "execute_code",
  "parameters": {
    "code": "string — Python code to execute",
    "description": "string — what this code does"
  },
  "returns": "Code output and any side effects"
}
```

Runs in a Python sandbox with helper functions: `json_parse`, `shell_quote`, retry helpers.

### delegate_task

```json
{
  "name": "delegate_task",
  "parameters": {
    "task": "string — task description for subagent",
    "toolset": "string — toolset to give subagent (optional)",
    "model": "string — model for subagent (optional)",
    "context": "string — additional context to pass"
  },
  "returns": "Subagent's final response"
}
```

Each subagent gets its own conversation, terminal session, and toolset. The iteration budget is shared with the parent.

### mixture_of_agents

```json
{
  "name": "mixture_of_agents",
  "parameters": {
    "prompt": "string — question or task",
    "models": "array — list of models to consult (optional)"
  },
  "returns": "Aggregated response from multiple LLMs"
}
```

Makes 5 API calls: 4 reference models + 1 aggregator with max reasoning effort.

---

## Skills Management

| Tool | Toolset | Description |
|------|---------|-------------|
| `skills_list` | `skills` | List available skills |
| `skill_view` | `skills` | Load full skill content |
| `skill_manage` | `skills` | Create, edit, delete skills |

### skills_list

```json
{
  "name": "skills_list",
  "parameters": {
    "category": "string — filter by category (optional)"
  },
  "returns": "List of {name, description} for each skill"
}
```

### skill_view

```json
{
  "name": "skill_view",
  "parameters": {
    "name": "string — skill name"
  },
  "returns": "Full SKILL.md content plus any linked reference files, templates, and scripts"
}
```

### skill_manage

```json
{
  "name": "skill_manage",
  "parameters": {
    "action": "string — create | patch | edit | delete | write_file | remove_file",
    "name": "string — skill name",
    "content": "string — skill content (for create/edit)",
    "old_string": "string — text to find (for patch)",
    "new_string": "string — replacement text (for patch)",
    "category": "string — skill category (for create)",
    "file_path": "string — relative path (for write_file/remove_file)",
    "file_content": "string — file content (for write_file)"
  },
  "returns": "Confirmation"
}
```

---

## Scheduling

| Tool | Toolset | Description |
|------|---------|-------------|
| `cronjob` | `cronjob` | Manage scheduled tasks |

### cronjob

```json
{
  "name": "cronjob",
  "parameters": {
    "action": "string — create | list | update | pause | resume | run | remove",
    "name": "string — job display name (for create)",
    "prompt": "string — agent prompt to run (for create)",
    "schedule": "string — cron expression or natural language (for create)",
    "deliver": "string — origin | local | telegram | discord | ... (for create)",
    "skills": "array — skills to load for the job (optional)",
    "id": "string — job ID (for non-create actions)"
  },
  "returns": "Job status or list"
}
```

---

## Home Assistant

| Tool | Toolset | Description |
|------|---------|-------------|
| `ha_list_entities` | `homeassistant` | List HA entities with filtering |
| `ha_get_state` | `homeassistant` | Get detailed state of an entity |
| `ha_list_services` | `homeassistant` | List available HA services |
| `ha_call_service` | `homeassistant` | Call HA service to control device |

**Requires:** `HASS_TOKEN` and `HASS_URL` environment variables.

---

## Honcho AI Memory

| Tool | Toolset | Description |
|------|---------|-------------|
| `honcho_profile` | `honcho` | Get user peer card (curated facts) |
| `honcho_search` | `honcho` | Semantic search over user context |
| `honcho_context` | `honcho` | Ask natural language question about user |
| `honcho_conclude` | `honcho` | Write conclusion about user to Honcho |

**Requires:** `HONCHO_API_KEY` and Honcho integration configured.

---

## Messaging

| Tool | Toolset | Description |
|------|---------|-------------|
| `send_message` | `messaging` | Send message to a connected platform |

```json
{
  "name": "send_message",
  "parameters": {
    "platform": "string — telegram | discord | slack | ...",
    "target": "string — chat ID or channel name",
    "message": "string — message content"
  },
  "returns": "Delivery confirmation"
}
```

---

## RL Training (10 tools)

Used to manage reinforcement learning training runs via Atropos environments:

| Tool | Description |
|------|-------------|
| `rl_list_environments` | List all available RL environments |
| `rl_select_environment` | Select environment for training |
| `rl_get_current_config` | Get current environment config |
| `rl_edit_config` | Update a config field |
| `rl_start_training` | Start a new training run |
| `rl_check_status` | Get status and metrics |
| `rl_stop_training` | Stop a running job |
| `rl_get_results` | Get final results and metrics |
| `rl_list_runs` | List all runs (active and completed) |
| `rl_test_inference` | Quick inference test |

---

## MCP (Model Context Protocol)

MCP tools are discovered dynamically from configured MCP servers. They appear as:

```
mcp_<server>_<tool_name>
```

For example, a filesystem MCP server named `fs` exposes: `mcp_fs_read_file`, `mcp_fs_write_file`, etc.

See [Configuration](configuration.md) for MCP server setup.

---

## Tool Implementation

All tool files follow the same pattern:

```python
# 1. Availability check
def check_my_tool_requirements() -> bool:
    return bool(os.environ.get("MY_API_KEY"))

# 2. Handler
def my_tool_handler(args: dict, **kwargs) -> str:
    # Must return JSON string; never raise exceptions
    return json.dumps({"result": "..."})

# 3. Schema (OpenAI format)
MY_TOOL_SCHEMA = {
    "name": "my_tool",
    "description": "...",
    "parameters": {
        "type": "object",
        "properties": {...},
        "required": [...]
    }
}

# 4. Registration
from tools.registry import registry
registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema=MY_TOOL_SCHEMA,
    handler=my_tool_handler,
    check_fn=check_my_tool_requirements,
    requires_env=["MY_API_KEY"]
)
```
