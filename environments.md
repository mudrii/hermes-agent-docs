# Execution Environments

Hermes Agent includes a full environment framework that connects its tool-calling capabilities to the [Atropos](https://github.com/NousResearch/atropos) RL training framework. This enables three workflows:

1. **RL Training** -- train language models on multi-turn agentic tasks with GRPO
2. **Benchmarks** -- evaluate models on standardised agentic benchmarks
3. **Data Generation** -- generate SFT training data from agent rollouts

All three share the same core: an **environment** class that defines tasks, runs an agent loop, and scores the output.

The environment framework lives under the repo's `environments/` directory. This is the implementation-level API for Hermes/Atropos integration, separate from the user-facing [RL training tools](./rl-training.md) which operate as an orchestration surface for remote RL training workflows.

## Terminal Backends

The `terminal` tool can execute commands in different environments. Configure in `~/.hermes/config.yaml`:

```yaml
terminal:
  backend: local    # or: docker, ssh, singularity, modal, daytona
  cwd: "."
  timeout: 180
```

| Backend | Description | Use Case | Isolation |
|---------|-------------|----------|-----------|
| `local` | Run on your machine (default) | Development, trusted tasks | None |
| `ssh` | Remote server | Sandboxing, keep agent away from its own code | Remote host |
| `docker` | Isolated containers | Security, reproducibility | Container (namespaces, cgroups) |
| `singularity` | HPC containers (Apptainer) | Cluster computing, rootless environments | Container (rootless) |
| `modal` | Cloud execution | Serverless, scale | Cloud sandbox |
| `daytona` | Cloud sandbox workspace | Persistent remote dev environments | Persistent cloud workspace |

### Docker Backend

```yaml
terminal:
  backend: docker
  docker_image: python:3.11-slim
  container_cpu: 1
  container_memory: 5120        # MB (default: 5GB)
  container_disk: 51200         # MB (default: 50GB)
  container_persistent: true
```

Docker containers run with security hardening by default. The following security arguments are applied:

```
--cap-drop ALL
--security-opt no-new-privileges
--pids-limit 256
--read-only
--tmpfs /tmp:rw,noexec,nosuid,size=512m
--tmpfs /var/tmp:rw,noexec,nosuid,size=256m
```

This drops all Linux capabilities, prevents privilege escalation, limits the container to 256 processes, makes the root filesystem read-only, and provides writable tmpfs mounts for temporary files. Persistent workspace is provided via volumes.

#### Docker Environment Variable Filtering

By default, no host environment variables are passed into Docker containers. The container environment starts empty for security. Only variables explicitly listed in the terminal config's `env_passthrough` list or in a skill's `required_environment_variables` are injected. This prevents accidental leakage of API keys, tokens, and credentials into the sandbox.

### SSH Backend

Recommended for security -- the agent cannot modify its own code:

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
apptainer build ~/python.sif docker://python:3.11-slim
hermes config set terminal.backend singularity
hermes config set terminal.singularity_image ~/python.sif
```

### Modal (Serverless Cloud)

```bash
uv pip install "swe-rex[modal]"
modal setup
hermes config set terminal.backend modal
```

### Daytona

Cloud sandbox workspace with persistent remote dev environments.

### Skills Directory Mounting (v0.6.0)

The `~/.hermes/skills/` directory is automatically mounted (live-synced) to all remote terminal backends: Docker, Modal, SSH, Singularity, and Daytona. This ensures the agent has access to the same skills regardless of which backend is executing commands. The sync uses mtime+size caching to avoid redundant transfers -- only changed files are pushed to the remote environment.

### Credential File Mounting

Credential files (`~/.hermes/.env` and skill-declared environment variables) are automatically passed to Docker and Modal containers. The runtime reads the `.env` file and injects the relevant variables into the container environment so that tool calls inside the sandbox can authenticate with external services.

## Architecture

The environment system is built on a three-layer inheritance chain:

### BaseEnv (Atropos)

The foundation from `atroposlib`. Provides:
- **Server management** -- connects to OpenAI-compatible APIs (VLLM, SGLang, OpenRouter)
- **Worker scheduling** -- parallel rollout coordination
- **Wandb integration** -- metrics logging and rollout visualisation
- **CLI interface** -- three subcommands: `serve`, `process`, `evaluate`
- **Eval logging** -- `evaluate_log()` saves results to JSON + JSONL

### HermesAgentBaseEnv

The hermes-agent layer (`environments/hermes_base_env.py`). Adds:
- **Terminal backend configuration** -- sets `TERMINAL_ENV` for sandboxed execution
- **Tool resolution** -- `_resolve_tools_for_group()` calls `get_tool_definitions()` to get the right tool schemas
- **Agent loop integration** -- `collect_trajectory()` runs `HermesAgentLoop` and scores the result
- **Two-phase operation** -- Phase 1 (OpenAI server) for eval/SFT, Phase 2 (VLLM ManagedServer) for full RL with logprobs
- **Async safety patches** -- handles Modal backend's event loop conflicts

### Concrete Environments

Your environment inherits from `HermesAgentBaseEnv` and implements five methods:

| Method | Purpose |
|--------|---------|
| `setup()` | Load dataset, initialise state |
| `get_next_item()` | Return the next item for rollout |
| `format_prompt(item)` | Convert an item into the user message |
| `compute_reward(item, result, ctx)` | Score the rollout (0.0-1.0) |
| `evaluate()` | Periodic evaluation logic |

## Agent Loop

`HermesAgentLoop` (`environments/agent_loop.py`) is the reusable multi-turn agent engine. It runs the same tool-calling pattern as hermes-agent's main loop:

1. Send messages + tool schemas to the API via `server.chat_completion()`
2. If the response contains `tool_calls`, dispatch each via `handle_function_call()`
3. Append tool results to the conversation, go back to step 1
4. If no `tool_calls`, the agent is done

Tool calls execute in a thread pool (`ThreadPoolExecutor(128)`) so that async backends (Modal, Docker) don't deadlock inside Atropos's event loop.

Returns an `AgentResult` containing the full conversation history, turn count, reasoning content per turn, tool errors, and optional ManagedServer state.

## Tool Context

`ToolContext` (`environments/tool_context.py`) gives reward functions direct access to the **same sandbox** the model used during its rollout. The `task_id` scoping means all state (files, processes, browser tabs) is preserved.

```python
async def compute_reward(self, item, result, ctx: ToolContext):
    test = ctx.terminal("pytest -v")
    if test["exit_code"] == 0:
        return 1.0

    content = ctx.read_file("/workspace/solution.py")
    if content.get("content"):
        return 0.5

    ctx.download_file("/remote/output.bin", "/local/output.bin")
    return 0.0
```

Available methods:

| Category | Methods |
|----------|---------|
| **Terminal** | `terminal(command, timeout)` |
| **Files** | `read_file(path)`, `write_file(path, content)`, `search(query, path)` |
| **Transfers** | `upload_file()`, `upload_dir()`, `download_file()`, `download_dir()` |
| **Web** | `web_search(query)`, `web_extract(urls)` |
| **Browser** | `browser_navigate(url)`, `browser_snapshot()` |
| **Generic** | `call_tool(name, args)` -- escape hatch for any hermes-agent tool |
| **Cleanup** | `cleanup()` -- release all resources |

## Tool Call Parsers

For **Phase 2** (VLLM ManagedServer), the server returns raw text without structured tool calls. Client-side parsers in `environments/tool_call_parsers/` extract `tool_calls` from raw output:

```python
from environments.tool_call_parsers import get_parser

parser = get_parser("hermes")
content, tool_calls = parser.parse(raw_model_output)
```

Available parsers: `hermes`, `mistral`, `llama3_json`, `qwen`, `qwen3_coder`, `deepseek_v3`, `deepseek_v3_1`, `kimi_k2`, `longcat`, `glm45`, `glm47`.

In Phase 1 (OpenAI server type), parsers are not needed -- the server handles tool call parsing natively.

## Available Benchmarks

### TerminalBench2

89 challenging terminal tasks with per-task Docker sandbox environments.

- **Scoring:** Binary pass/fail (test suite verification)
- **Sandbox:** Modal cloud sandboxes (per-task Docker images)
- **Tools:** `terminal` + `file`
- **Cost:** ~$50-200 for full eval
- **Time:** ~2-4 hours

```bash
python environments/benchmarks/terminalbench_2/terminalbench2_env.py evaluate \
    --config environments/benchmarks/terminalbench_2/default.yaml
```

Dataset: [NousResearch/terminal-bench-2](https://huggingface.co/datasets/NousResearch/terminal-bench-2).

### TBLite (OpenThoughts Terminal Bench Lite)

100 difficulty-calibrated tasks -- a faster proxy for TerminalBench2.

- **Tasks:** Easy (40), Medium (26), Hard (26), Extreme (8)
- **Correlation:** r=0.911 with full TB2
- **Speed:** 2.6-8x faster than TB2

```bash
python environments/benchmarks/tblite/tblite_env.py evaluate \
    --config environments/benchmarks/tblite/default.yaml
```

Dataset: [NousResearch/openthoughts-tblite](https://huggingface.co/datasets/NousResearch/openthoughts-tblite).

### YC-Bench

Long-horizon strategic benchmark -- the agent plays CEO of an AI startup.

- **Scoring:** Composite: `0.5 x survival + 0.5 x normalised_funds`
- **Sandbox:** Local terminal (no Modal needed)
- **Runs:** 9 default (3 presets x 3 seeds), sequential
- **Cost:** ~$50-200 for full eval
- **Time:** ~3-6 hours

```bash
pip install "hermes-agent[yc-bench]"
python environments/benchmarks/yc_bench/yc_bench_env.py evaluate \
    --config environments/benchmarks/yc_bench/default.yaml
```

Uses [collinear-ai/yc-bench](https://github.com/collinear-ai/yc-bench) -- a deterministic simulation with 4 skill domains, prestige system, employee management, and financial pressure.

## Training Environments

### TerminalTestEnv

A minimal self-contained environment with inline tasks (no external dataset). Used for validating the full stack end-to-end. Each task asks the model to create a file at a known path; the verifier checks the content.

```bash
python environments/terminal_test_env/terminal_test_env.py process \
    --env.data_path_to_save_groups terminal_test_output.jsonl
```

### HermesSweEnv

SWE-bench style training environment. The model gets a coding task, uses terminal + file + web tools to solve it, and the reward function runs tests in the same Modal sandbox.

```bash
python environments/hermes_swe_env/hermes_swe_env.py serve \
    --openai.model_name YourModel \
    --env.dataset_name bigcode/humanevalpack \
    --env.terminal_backend modal
```

## Running Environments

Every environment is a standalone Python script with three CLI subcommands:

### `evaluate` -- Run a benchmark

For eval-only environments. Runs all items, computes metrics, logs to wandb. No training server or `run-api` needed.

```bash
python my_env.py evaluate \
    --config my_config.yaml \
    --openai.model_name anthropic/claude-sonnet-4.6
```

### `process` -- Generate SFT data

Runs rollouts and saves scored trajectories to JSONL. Useful for generating training data without a full RL loop.

### `serve` -- Connect to Atropos for RL training

Connects the environment to a running Atropos API server. The environment receives items from Atropos, runs agent rollouts, computes rewards, and sends scored trajectories back for training.

## Two-Phase Operation

### Phase 1: OpenAI Server (Eval / SFT)

Uses `server.chat_completion()` with `tools=` parameter. The server handles tool call parsing natively. Returns `ChatCompletion` objects with structured `tool_calls`.

- **Use for:** evaluation, SFT data generation, benchmarks, testing
- Placeholder tokens are created for the Atropos pipeline

### Phase 2: VLLM ManagedServer (Full RL)

Uses ManagedServer for exact token IDs + logprobs via `/generate`. A client-side tool call parser reconstructs structured `tool_calls` from raw output.

- **Use for:** full RL training with GRPO/PPO
- Real tokens, masks, and logprobs flow through the pipeline
- Set `tool_call_parser` in config to match your model's format

## Configuration Reference

### HermesAgentEnvConfig Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled_toolsets` | `List[str]` | `None` (all) | Which hermes toolsets to enable |
| `disabled_toolsets` | `List[str]` | `None` | Toolsets to filter out |
| `distribution` | `str` | `None` | Probabilistic toolset distribution name |
| `max_agent_turns` | `int` | `30` | Max LLM calls per rollout |
| `agent_temperature` | `float` | `1.0` | Sampling temperature |
| `system_prompt` | `str` | `None` | System message for the agent |
| `terminal_backend` | `str` | `"local"` | `local`, `docker`, `modal`, `daytona`, `ssh`, `singularity` |
| `terminal_timeout` | `int` | `120` | Seconds per terminal command |
| `terminal_lifetime` | `int` | `3600` | Max sandbox lifetime |
| `dataset_name` | `str` | `None` | HuggingFace dataset identifier |
| `tool_pool_size` | `int` | `128` | Thread pool size for tool execution |
| `tool_call_parser` | `str` | `"hermes"` | Parser for Phase 2 raw output |
| `extra_body` | `Dict` | `None` | Extra params for OpenAI API |
| `eval_handling` | `Enum` | `STOP_TRAIN` | `STOP_TRAIN`, `LIMIT_TRAIN`, `NONE` |

### YAML Configuration

```yaml
env:
  enabled_toolsets: ["terminal", "file"]
  max_agent_turns: 60
  max_token_length: 32000
  agent_temperature: 0.8
  terminal_backend: "modal"
  terminal_timeout: 300
  dataset_name: "NousResearch/terminal-bench-2"
  tokenizer_name: "NousResearch/Hermes-3-Llama-3.1-8B"
  use_wandb: true
  wandb_name: "my-benchmark"

openai:
  base_url: "https://openrouter.ai/api/v1"
  model_name: "anthropic/claude-sonnet-4.6"
  server_type: "openai"
  health_check: false
```

YAML values override `config_init()` defaults. CLI arguments override YAML values.

## Prerequisites

### For all environments

- Python >= 3.11
- `atroposlib`: `pip install git+https://github.com/NousResearch/atropos.git`
- An LLM API key (OpenRouter, OpenAI, or self-hosted VLLM/SGLang)

### For Modal-sandboxed benchmarks (TB2, TBLite)

- [Modal](https://modal.com) account and CLI: `pip install "hermes-agent[modal]"`
- `MODAL_TOKEN_ID` and `MODAL_TOKEN_SECRET` environment variables

### For YC-Bench

- `pip install "hermes-agent[yc-bench]"` (installs the yc-bench CLI + SQLAlchemy)
- No Modal needed -- runs with local terminal backend

### For RL training

- `TINKER_API_KEY` -- API key for the [Tinker](https://tinker.computer) training service
- `WANDB_API_KEY` -- for Weights and Biases metrics tracking
- The `tinker-atropos` submodule

## Directory Structure

```
environments/
  hermes_base_env.py          # Abstract base class (HermesAgentBaseEnv)
  agent_loop.py               # Multi-turn agent engine (HermesAgentLoop)
  tool_context.py             # Per-rollout tool access for reward functions
  patches.py                  # Async-safety patches (now a no-op, kept for imports)

  tool_call_parsers/          # Phase 2 client-side parsers
    hermes_parser.py
    mistral_parser.py
    llama_parser.py
    qwen_parser.py
    qwen3_coder_parser.py
    deepseek_v3_parser.py
    deepseek_v3_1_parser.py
    kimi_k2_parser.py
    longcat_parser.py
    glm45_parser.py
    glm47_parser.py

  terminal_test_env/          # Stack validation (inline tasks)
  hermes_swe_env/             # SWE-bench training environment

  benchmarks/
    terminalbench_2/          # 89 terminal tasks, Modal sandboxes
    tblite/                   # 100 calibrated tasks (fast TB2 proxy)
    yc_bench/                 # Long-horizon strategic benchmark
```
