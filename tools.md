# Tools

Tools are Python functions exposed to the AI as callable capabilities. Each tool has a name, a JSON schema describing its parameters, a handler function that executes the logic, and an optional check function that determines whether the tool is available in the current environment.

## What Tools Are

Tools are the mechanism through which Hermes Agent interacts with the world. Unlike skills (which are instruction documents the agent reads), tools are code that runs when the agent calls them. The agent receives tool definitions as OpenAI-format function schemas and calls them by name during reasoning.

Every tool:
- Has a unique name (e.g., `web_search`, `terminal`, `browser_navigate`)
- Belongs to exactly one toolset (e.g., `web`, `terminal`, `browser`)
- Has a JSON schema describing its parameters
- Returns a JSON string — always, including on error (`{"error": "message"}`)
- Has an optional `check_fn` that returns `False` to silently exclude the tool when dependencies are missing

## Tool Discovery and Registration

Tools self-register at import time via `tools/registry.py`. The registration flow is:

1. `model_tools.py` imports each tool module in `_discover_tools()`
2. Each module calls `registry.register(...)` at the bottom of the file
3. After built-in tools, `discover_mcp_tools()` connects to configured MCP servers
4. After MCP tools, `discover_plugins()` loads any installed plugin tools
5. `get_tool_definitions()` filters the registered tools through toolset resolution and `check_fn` evaluation

The full list of tool modules imported by `model_tools.py`:

```
tools.web_tools
tools.terminal_tool
tools.file_tools
tools.vision_tools
tools.mixture_of_agents_tool
tools.image_generation_tool
tools.skills_tool
tools.skill_manager_tool
tools.browser_tool
tools.cronjob_tools
tools.rl_training_tool
tools.tts_tool
tools.todo_tool
tools.memory_tool
tools.session_search_tool
tools.clarify_tool
tools.code_execution_tool
tools.delegate_tool
tools.process_registry
tools.send_message_tool
tools.honcho_tools
tools.homeassistant_tool
```

## Tool Execution Environment

Tools are dispatched through `handle_function_call()` in `model_tools.py`, which routes each call through `registry.dispatch()`. Four tools (`todo`, `memory`, `session_search`, `delegate_task`) are intercepted by the agent loop in `run_agent.py` before reaching the registry because they need access to per-session agent state.

The registry handles async bridging transparently: if a handler is registered with `is_async=True`, the registry calls `_run_async()` which runs the coroutine in a thread pool if an event loop is already running.

Tool calls may execute sequentially or concurrently depending on the tool mix and the model's interaction requirements.

## All Built-in Tools by Toolset

### `browser` toolset

Browser automation for web interaction. Requires browser automation infrastructure to be available.

| Tool | Description |
|------|-------------|
| `browser_back` | Navigate back to the previous page in browser history. Requires `browser_navigate` to be called first. |
| `browser_click` | Click on an element identified by its ref ID from the snapshot (e.g., `@e5`). Ref IDs are shown in square brackets in snapshot output. Requires `browser_navigate` and `browser_snapshot` first. |
| `browser_close` | Close the browser session and release resources. Call when done with browser tasks to free up session quota. |
| `browser_console` | Get browser console output and JavaScript errors from the current page. Returns console.log/warn/error/info messages and uncaught JS exceptions. Useful for detecting silent JavaScript errors and failed API calls. |
| `browser_get_images` | Get a list of all images on the current page with their URLs and alt text. Useful for finding images to analyze with the vision tool. Requires `browser_navigate` first. |
| `browser_navigate` | Navigate to a URL in the browser. Initializes the session and loads the page. Must be called before other browser tools. For simple information retrieval, prefer `web_search` or `web_extract`. |
| `browser_press` | Press a keyboard key. Useful for submitting forms (Enter), navigating (Tab), or keyboard shortcuts. Requires `browser_navigate` first. |
| `browser_scroll` | Scroll the page in a direction. Use to reveal more content below or above the current viewport. Requires `browser_navigate` first. |
| `browser_snapshot` | Get a text-based snapshot of the current page's accessibility tree. Returns interactive elements with ref IDs (like `@e1`, `@e2`) for `browser_click` and `browser_type`. `full=false` (default) gives a compact view; `full=true` gives a comprehensive view. |
| `browser_type` | Type text into an input field identified by its ref ID. Clears the field first, then types the new text. Requires `browser_navigate` and `browser_snapshot` first. |
| `browser_vision` | Take a screenshot of the current page and analyze it with vision AI. Use when you need to visually understand page content — especially for CAPTCHAs, visual verification challenges, and complex layouts. |

The `browser` toolset also includes `web_search` so the agent can find URLs before navigating.

### `clarify` toolset

| Tool | Description |
|------|-------------|
| `clarify` | Ask the user a question when clarification, feedback, or a decision is needed before proceeding. Supports two modes: (1) Multiple choice — provide up to 4 choices, user picks one or types their own via a 5th "Other" option; (2) Open-ended question. |

### `code_execution` toolset

| Tool | Description |
|------|-------------|
| `execute_code` | Run a Python script that can call Hermes tools programmatically. Use when you need 3+ tool calls with processing logic between them, need to filter/reduce large tool outputs before they enter context, or need conditional branching. The sandbox lists which tools are available to the script. |

### `cronjob` toolset

| Tool | Description |
|------|-------------|
| `cronjob` | Unified scheduled-task manager. Use `action="create"`, `"list"`, `"update"`, `"pause"`, `"resume"`, `"run"`, or `"remove"` to manage jobs. Supports skill-backed jobs with one or more attached skills. `skills=[]` on update clears attached skills. Cron runs happen in fresh sessions with no current-chat context. |

### `delegation` toolset

| Tool | Description |
|------|-------------|
| `delegate_task` | Spawn one or more subagents to work on tasks in isolated contexts. Each subagent gets its own conversation, terminal session, and toolset. Only the final summary is returned — intermediate tool results never enter your context window. |

### `file` toolset

| Tool | Description |
|------|-------------|
| `patch` | Targeted find-and-replace edits in files. Use instead of sed/awk in terminal. Uses fuzzy matching (9 strategies) so minor whitespace/indentation differences do not break it. Returns a unified diff. Runs syntax checks automatically after editing Python and JavaScript files. |
| `read_file` | Read a text file with line numbers and pagination. Use instead of cat/head/tail in terminal. Output format: `LINE_NUM|CONTENT`. Suggests similar filenames if not found. Use `offset` and `limit` for large files. Cannot read images or binary files. |
| `search_files` | Search file contents or find files by name. Use instead of grep/rg/find/ls in terminal. Ripgrep-backed, faster than shell equivalents. Content search (`target='content'`): regex search inside files. File search: find by name pattern. |
| `write_file` | Write content to a file, completely replacing existing content. Use instead of echo/cat heredoc in terminal. Creates parent directories automatically. Overwrites the entire file — use `patch` for targeted edits. |

### `homeassistant` toolset

Requires `HASS_TOKEN` environment variable. Gated via check function.

| Tool | Description |
|------|-------------|
| `ha_call_service` | Call a Home Assistant service to control a device. Use `ha_list_services` first to discover available services and their parameters for each domain. |
| `ha_get_state` | Get the detailed state of a single Home Assistant entity, including all attributes (brightness, color, temperature setpoint, sensor readings). |
| `ha_list_entities` | List Home Assistant entities. Optionally filter by domain (light, switch, climate, sensor, binary_sensor, cover, fan) or by area name (living room, kitchen, bedroom). |
| `ha_list_services` | List available Home Assistant services (actions) for device control. Shows what actions can be performed on each device type and what parameters they accept. |

### `honcho` toolset

Honcho AI-native memory for persistent cross-session user modeling. Gated via check function when Honcho is configured.

| Tool | Description |
|------|-------------|
| `honcho_conclude` | Write a conclusion about the user back to Honcho's memory. Conclusions are persistent facts that build the user's profile — preferences, corrections, clarifications, project context. |
| `honcho_context` | Ask Honcho a natural language question and get a synthesized answer using Honcho's LLM (dialectic reasoning). Higher cost than `honcho_profile` or `honcho_search`. Can query about any peer: the user (default), the AI assistant, or any named person. |
| `honcho_profile` | Retrieve the user's peer card from Honcho — a curated list of key facts about them (name, role, preferences, communication style, patterns). Fast, no LLM reasoning, minimal cost. |
| `honcho_search` | Semantic search over Honcho's stored context about the user. Returns raw excerpts ranked by relevance — no LLM synthesis. Cheaper and faster than `honcho_context`. |

### `image_gen` toolset

| Tool | Description | Requires |
|------|-------------|---------|
| `image_generate` | Generate high-quality images from text prompts using FLUX 2 Pro model with automatic 2x upscaling. Returns a single upscaled image URL. | `FAL_KEY` |

### `memory` toolset

| Tool | Description |
|------|-------------|
| `memory` | Save important information to persistent memory that survives across sessions. Memory appears in the system prompt at session start. Intercepted by the agent loop to access the session's MemoryStore. |

### `messaging` toolset

Gated via check function when gateway is running.

| Tool | Description |
|------|-------------|
| `send_message` | Send a message to a connected messaging platform, or list available targets. Call `send_message(action='list')` first when targeting a specific channel or person to see available targets. |

### `moa` toolset

| Tool | Description | Requires |
|------|-------------|---------|
| `mixture_of_agents` | Route a hard problem through multiple frontier LLMs collaboratively. Makes 5 API calls (4 reference models + 1 aggregator) with maximum reasoning effort. Use sparingly for genuinely difficult problems: complex math, advanced algorithms, multi-step reasoning. | `OPENROUTER_API_KEY` |

### `rl` toolset

RL training tools for running reinforcement learning on Tinker-Atropos. Requires both `TINKER_API_KEY` and `WANDB_API_KEY`.

| Tool | Description |
|------|-------------|
| `rl_check_status` | Get status and metrics for a training run. Rate limited: enforces 30-minute minimum between checks for the same run. Returns WandB metrics: step, state, reward_mean, loss, percent_correct. |
| `rl_edit_config` | Update a configuration field. Use `rl_get_current_config()` first to see all available fields for the selected environment. Infrastructure settings (tokenizer, URLs, lora_rank, learning_rate) are fixed and cannot be changed. |
| `rl_get_current_config` | Get the current environment configuration. Returns only fields that can be modified: group_size, max_token_length, total_steps, steps_per_eval, use_wandb, wandb_name, max_num_workers. |
| `rl_get_results` | Get final results and metrics for a completed training run. Returns final metrics and path to trained weights. |
| `rl_list_environments` | List all available RL environments with names, paths, and descriptions. |
| `rl_list_runs` | List all training runs (active and completed) with their status. |
| `rl_select_environment` | Select an RL environment for training. Loads the environment's default configuration. After selecting, use `rl_get_current_config()` to see settings. |
| `rl_start_training` | Start a new RL training run with the current environment and config. Most training parameters are fixed. Use `rl_edit_config()` to set group_size, batch_size, wandb_project before starting. |
| `rl_stop_training` | Stop a running training job. |
| `rl_test_inference` | Quick inference test for any environment. Runs inference + scoring steps using OpenRouter. Default: 3 steps x 16 completions = 48 rollouts per model, testing 3 models = 144 total. |

### `session_search` toolset

| Tool | Description |
|------|-------------|
| `session_search` | Search your long-term memory of past conversations. Every past session is searchable and this tool summarizes what happened. Use proactively when the user says "we did this before", "remember when", or "last time". Intercepted by the agent loop. |

### `skills` toolset

| Tool | Description |
|------|-------------|
| `skill_manage` | Manage skills (create, update, delete). Skills are procedural memory — reusable approaches for recurring task types. New skills go to `~/.hermes/skills/`. Actions: `create` (full SKILL.md), `patch` (targeted fixes, preferred), `edit` (full replacement), `delete`, `write_file`, `remove_file`. |
| `skill_view` | Load a skill's full content or access its linked files (references, templates, scripts). First call returns SKILL.md content plus a `linked_files` dict showing available references/templates/scripts. Call again with `file_path` parameter to access those. |
| `skills_list` | List available skills (name + description). Use `skill_view(name)` to load full content. Accepts optional `category` filter. |

### `terminal` toolset

| Tool | Description |
|------|-------------|
| `process` | Manage background processes started with `terminal(background=true)`. Actions: `list` (show all), `poll` (check status + new output), `log` (full output with pagination), `wait` (block until done or timeout), `kill` (terminate), `write` (send input to stdin). |
| `terminal` | Execute shell commands. Filesystem persists between calls. Do not use cat/head/tail — use `read_file`. Do not use grep/rg/find — use `search_files`. Supports `background=true` for async execution and `pty=true` for interactive CLI tools. |

### `todo` toolset

| Tool | Description |
|------|-------------|
| `todo` | Manage the task list for the current session. Use for complex tasks with 3+ steps or when the user provides multiple tasks. Call with no parameters to read the current list. Intercepted by the agent loop to access the session's TodoStore. |

### `tts` toolset

| Tool | Description |
|------|-------------|
| `text_to_speech` | Convert text to speech audio. Supports four providers: Edge TTS (default, free), ElevenLabs (premium), OpenAI TTS, and NeuTTS (local, free). Returns a MEDIA: path that the platform delivers as a voice message. On Telegram it plays as a voice bubble; on Discord/WhatsApp as an audio attachment; in CLI mode, saves to `~/voice-memos/`. Voice and provider are configurable. |

### `vision` toolset

| Tool | Description |
|------|-------------|
| `vision_analyze` | Analyze images using AI vision. Provides a comprehensive description and answers a specific question about the image content. |

### `web` toolset

| Tool | Description | Requires |
|------|-------------|---------|
| `web_extract` | Extract content from web page URLs. Returns page content in markdown format. Also works with PDF URLs — pass the PDF link directly and it converts to markdown text. Pages under 5000 chars return full markdown; larger pages are LLM-summarized. | `PARALLEL_API_KEY` or `FIRECRAWL_API_KEY` or `TAVILY_API_KEY` (any one) |
| `web_search` | Search the web for information on any topic. Returns up to 5 relevant results with titles, URLs, and descriptions. | `PARALLEL_API_KEY` or `FIRECRAWL_API_KEY` or `TAVILY_API_KEY` (any one) |

## Terminal Backends

The `terminal` tool can execute commands in different environments:

| Backend | Description | Use Case |
|---------|-------------|----------|
| `local` | Run on your machine (default) | Development, trusted tasks |
| `docker` | Isolated containers | Security, reproducibility |
| `ssh` | Remote server | Sandboxing, keep agent away from its own code |
| `singularity` | HPC containers | Cluster computing, rootless environments |
| `modal` | Cloud execution | Serverless, scale |
| `daytona` | Cloud sandbox workspace | Persistent remote dev environments |

Configure in `~/.hermes/config.yaml`:

```yaml
terminal:
  backend: local    # or: docker, ssh, singularity, modal, daytona
  cwd: "."          # Working directory
  timeout: 180      # Command timeout in seconds
```

### Docker Backend

```yaml
terminal:
  backend: docker
  docker_image: python:3.11-slim
  container_cpu: 1              # CPU cores (default: 1)
  container_memory: 5120        # Memory in MB (default: 5GB)
  container_disk: 51200         # Disk in MB (default: 50GB)
  container_persistent: true    # Persist filesystem across sessions (default: true)
```

Docker containers run with security hardening: read-only root filesystem, all Linux capabilities dropped, no privilege escalation, 256-process PID limit, and full namespace isolation. Persistent workspace is provided via volumes, not a writable root layer.

### SSH Backend

Recommended for security — the agent cannot modify its own code:

```yaml
terminal:
  backend: ssh
```

Set credentials in `~/.hermes/.env`:

```
TERMINAL_SSH_HOST=my-server.example.com
TERMINAL_SSH_USER=myuser
TERMINAL_SSH_KEY=~/.ssh/id_rsa
```

### Singularity/Apptainer

```bash
# Pre-build SIF for parallel workers
apptainer build ~/python.sif docker://python:3.11-slim

# Configure
hermes config set terminal.backend singularity
hermes config set terminal.singularity_image ~/python.sif
```

### Modal (Serverless Cloud)

```bash
uv pip install "swe-rex[modal]"
modal setup
hermes config set terminal.backend modal
```

## Background Process Management

Start background processes and manage them:

```python
terminal(command="pytest -v tests/", background=true)
# Returns: {"session_id": "proc_abc123", "pid": 12345}

process(action="list")
process(action="poll", session_id="proc_abc123")
process(action="wait", session_id="proc_abc123")
process(action="log", session_id="proc_abc123")
process(action="kill", session_id="proc_abc123")
process(action="write", session_id="proc_abc123", data="y")
```

PTY mode (`pty=true`) enables interactive CLI tools like Codex and Claude Code.

## Sudo Support

If a command needs sudo, you will be prompted for your password (cached for the session). Or set `SUDO_PASSWORD` in `~/.hermes/.env`. On messaging platforms, if sudo fails, the output includes a tip to add `SUDO_PASSWORD` to `~/.hermes/.env`.

## Security and Safety Notes

### Agent-Loop Intercepted Tools

`todo`, `memory`, `session_search`, and `delegate_task` need access to per-session agent state. These are intercepted by `run_agent.py` before reaching the registry. The registry still holds their schemas, but `dispatch()` returns a fallback error if the intercept is bypassed.

### Check Functions

Each tool can define a `check_fn` that returns `True` or `False`. When `check_fn` returns `False`, the tool is silently excluded from the session's tool definitions. This is how tools gate on environment variables:
- `image_generate` gates on `FAL_KEY`
- `web_search` / `web_extract` gate on `PARALLEL_API_KEY` or `FIRECRAWL_API_KEY` or `TAVILY_API_KEY`
- `mixture_of_agents` gates on `OPENROUTER_API_KEY`
- `ha_*` tools gate on `HASS_TOKEN`
- `rl_*` tools gate on `TINKER_API_KEY` and `WANDB_API_KEY`
- `honcho_*` tools gate on Honcho being active
- `send_message` gates on the gateway running

### Error Handling

Handlers must return a JSON string. Errors must be returned as `{"error": "message"}`, never raised as exceptions. This means the agent always receives a structured response it can reason about.

### Tool Approval

Dangerous terminal commands trigger an approval callback in supported environments. The callback is configured by the platform (CLI, gateway, etc.) and can block or allow the command.

## How to Add a Custom Tool (via Plugin)

Adding a native tool touches three files:

1. **`tools/your_tool.py`** — handler, schema, check function, `registry.register()` call
2. **`toolsets.py`** — add tool name to `_HERMES_CORE_TOOLS` or a specific toolset
3. **`model_tools.py`** — add `"tools.your_tool"` to the `_discover_tools()` list

### Tool file structure

Every tool file follows the same structure:

```python
# tools/weather_tool.py

import json
import os
import logging

logger = logging.getLogger(__name__)


def check_weather_requirements() -> bool:
    return bool(os.getenv("WEATHER_API_KEY"))


def weather_tool(location: str, units: str = "metric") -> str:
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return json.dumps({"error": "WEATHER_API_KEY not configured"})
    try:
        # call weather API...
        return json.dumps({"location": location, "temp": 22, "units": units})
    except Exception as e:
        return json.dumps({"error": str(e)})


WEATHER_SCHEMA = {
    "name": "weather",
    "description": "Get current weather for a location.",
    "parameters": {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "City name or coordinates (e.g. 'London' or '51.5,-0.1')"
            },
            "units": {
                "type": "string",
                "enum": ["metric", "imperial"],
                "description": "Temperature units (default: metric)",
                "default": "metric"
            }
        },
        "required": ["location"]
    }
}


from tools.registry import registry

registry.register(
    name="weather",
    toolset="weather",
    schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool(
        location=args.get("location", ""),
        units=args.get("units", "metric")),
    check_fn=check_weather_requirements,
    requires_env=["WEATHER_API_KEY"],
)
```

### Key rules

- Handlers **must** return a JSON string (via `json.dumps()`), never raw dicts
- Errors **must** be returned as `{"error": "message"}`, never raised as exceptions
- The `check_fn` is called when building tool definitions — if it returns `False`, the tool is silently excluded
- The `handler` receives `(args: dict, **kwargs)` where `args` is the LLM's tool call arguments

### Async handlers

If your handler needs async code, mark it with `is_async=True`:

```python
async def weather_tool_async(location: str) -> str:
    async with aiohttp.ClientSession() as session:
        ...
    return json.dumps(result)

registry.register(
    name="weather",
    toolset="weather",
    schema=WEATHER_SCHEMA,
    handler=lambda args, **kw: weather_tool_async(args.get("location", "")),
    check_fn=check_weather_requirements,
    is_async=True,  # registry calls _run_async() automatically
)
```

The registry handles async bridging transparently — you never call `asyncio.run()` yourself.

### Handlers that need task_id

Tools that manage per-session state receive `task_id` via `**kwargs`:

```python
def _handle_weather(args, **kw):
    task_id = kw.get("task_id")
    return weather_tool(args.get("location", ""), task_id=task_id)

registry.register(
    name="weather",
    ...
    handler=_handle_weather,
)
```

### Optional: setup wizard integration

If your tool requires an API key, add it to `hermes_cli/config.py`:

```python
OPTIONAL_ENV_VARS = {
    ...
    "WEATHER_API_KEY": {
        "description": "Weather API key for weather lookup",
        "prompt": "Weather API key",
        "url": "https://weatherapi.com/",
        "tools": ["weather"],
        "password": True,
    },
}
```

### Checklist

- [ ] Tool file created with handler, schema, check function, and registration
- [ ] Added to appropriate toolset in `toolsets.py`
- [ ] Discovery import added to `model_tools.py`
- [ ] Handler returns JSON strings; errors returned as `{"error": "..."}`
- [ ] Optional: API key added to `OPTIONAL_ENV_VARS` in `hermes_cli/config.py`
- [ ] Optional: Added to `toolset_distributions.py` for batch processing
- [ ] Tested with `hermes chat -q "Use the weather tool for London"`
