# Batch Processing

Batch processing lets you run the Hermes agent across hundreds or thousands of prompts in parallel, generating structured trajectory data. The primary use case is **training data generation** — producing ShareGPT-format trajectories with tool usage statistics for fine-tuning or evaluation.

## Overview

The batch runner (`batch_runner.py`) processes a JSONL dataset of prompts, running each through a full agent session with tool access. Each prompt gets its own isolated environment. The output is structured trajectory data with full conversation history, tool call statistics, and reasoning coverage metrics.

## Quick Start

```bash
# Basic batch run
python batch_runner.py \
    --dataset_file=data/prompts.jsonl \
    --batch_size=10 \
    --run_name=my_first_run \
    --model=anthropic/claude-sonnet-4-20250514 \
    --num_workers=4

# Resume an interrupted run
python batch_runner.py \
    --dataset_file=data/prompts.jsonl \
    --batch_size=10 \
    --run_name=my_first_run \
    --resume

# List available toolset distributions
python batch_runner.py --list_distributions
```

## Dataset Format

The input dataset is a JSONL file (one JSON object per line). Each entry must have a `prompt` field:

```jsonl
{"prompt": "Write a Python function that finds the longest palindromic substring"}
{"prompt": "Create a REST API endpoint for user authentication using Flask"}
{"prompt": "Debug this error: TypeError: cannot unpack non-iterable NoneType object"}
```

Optional fields:

- `image` or `docker_image`: A container image to use for this prompt's sandbox (works with Docker, Modal, and Singularity backends)
- `cwd`: Working directory override for the task's terminal session

```jsonl
{"prompt": "Install numpy and compute eigenvalues of a 3x3 matrix", "image": "python:3.11-slim"}
{"prompt": "Compile this Rust program and run it", "image": "rust:1.75"}
{"prompt": "Set up a Node.js Express server", "image": "node:20-alpine", "cwd": "/app"}
```

When a `docker_image` field is present, the batch runner verifies the Docker image is accessible before spending tokens on the agent loop. If the image is not in the local cache, it attempts `docker pull`.

## Configuration Options

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--dataset_file` | (required) | Path to JSONL dataset |
| `--batch_size` | (required) | Prompts per batch |
| `--run_name` | (required) | Name for this run (used for output dir and checkpointing) |
| `--distribution` | `"default"` | Toolset distribution to sample from |
| `--model` | `anthropic/claude-sonnet-4.6` | Model to use |
| `--base_url` | `https://openrouter.ai/api/v1` | API base URL |
| `--api_key` | (env var) | API key for model |
| `--max_turns` | `10` | Maximum tool-calling iterations per prompt |
| `--num_workers` | `4` | Parallel worker processes |
| `--resume` | `false` | Resume from checkpoint |
| `--verbose` | `false` | Enable verbose logging |
| `--max_samples` | all | Only process first N samples from dataset |
| `--max_tokens` | model default | Maximum tokens per model response |

### Provider Routing (OpenRouter)

| Parameter | Description |
|-----------|-------------|
| `--providers_allowed` | Comma-separated providers to allow (e.g., `"anthropic,openai"`) |
| `--providers_ignored` | Comma-separated providers to ignore (e.g., `"together,deepinfra"`) |
| `--providers_order` | Comma-separated preferred provider order |
| `--provider_sort` | Sort by `"price"`, `"throughput"`, or `"latency"` |

### Reasoning Control

| Parameter | Description |
|-----------|-------------|
| `--reasoning_effort` | Effort level: `xhigh`, `high`, `medium`, `low`, `minimal`, `none` |
| `--reasoning_disabled` | Completely disable reasoning/thinking tokens |

### Advanced Options

| Parameter | Description |
|-----------|-------------|
| `--ephemeral_system_prompt` | System prompt used during execution but NOT saved to trajectories |
| `--log_prefix_chars` | Characters to show in log previews (default: 100) |
| `--prefill_messages_file` | Path to JSON file with prefill messages for few-shot priming |

## Toolset Distributions

Each prompt gets a randomly sampled set of toolsets from a **distribution**. This ensures training data covers diverse tool combinations. Use `--list_distributions` to see all available distributions.

Distributions assign a probability to each individual toolset. The sampler flips each toolset independently, then guarantees that at least one toolset is enabled. This is different from a hand-authored table of prebuilt combinations.

## Output Format

All output goes to `data/<run_name>/`:

```text
data/my_run/
├── trajectories.jsonl    # Combined final output (all batches merged)
├── batch_0.jsonl         # Individual batch results
├── batch_1.jsonl
├── ...
├── checkpoint.json       # Resume checkpoint
└── statistics.json       # Aggregate tool usage stats
```

### Trajectory Format

Each line in `trajectories.jsonl` is a JSON object:

```json
{
  "prompt_index": 42,
  "conversations": [
    {"from": "human", "value": "Write a function..."},
    {"from": "gpt", "value": "I'll create that function...",
     "tool_calls": [...]},
    {"from": "tool", "value": "..."},
    {"from": "gpt", "value": "Here's the completed function..."}
  ],
  "metadata": {
    "batch_num": 2,
    "timestamp": "2026-01-15T10:30:00",
    "model": "anthropic/claude-sonnet-4-20250514"
  },
  "completed": true,
  "partial": false,
  "api_calls": 3,
  "toolsets_used": ["terminal", "file"],
  "tool_stats": {
    "terminal": {"count": 2, "success": 2, "failure": 0},
    "read_file": {"count": 1, "success": 1, "failure": 0}
  },
  "tool_error_counts": {
    "terminal": 0,
    "read_file": 0
  }
}
```

The `conversations` field uses ShareGPT-like format with `from` and `value` fields. Tool stats are normalized to include all possible tools with zero defaults, ensuring consistent schema across entries for HuggingFace datasets compatibility.

## Checkpointing

The batch runner has robust checkpointing for fault tolerance:

- **Checkpoint file:** Saved after each batch completes, tracking which prompt indices are done. Incremental writes happen after each batch finishes, so a crash mid-run loses at most one batch.
- **Content-based resume:** On `--resume`, the runner scans existing batch files and matches completed prompts by their actual text content (not just indices), enabling recovery even if the dataset order changes
- **Failed prompts:** Only successfully completed prompts are marked as done — failed prompts will be retried on resume
- **Batch merging:** On completion, all batch files (including from previous runs) are merged into a single `trajectories.jsonl`

### How Resume Works

1. Scan all `batch_*.jsonl` files for completed prompts (by content matching on the human turn text)
2. Filter the dataset to exclude already-completed prompts
3. Re-batch the remaining prompts
4. Process only the remaining prompts
5. Merge all batch files (old and new) into final output

## Quality Filtering

The batch runner applies automatic quality filtering during the final merge step:

- **No-reasoning filter:** Samples where zero assistant turns contain reasoning (no `<REASONING_SCRATCHPAD>` or native thinking tokens) are discarded. These are logged with a count at the end.
- **Corrupted entry filter:** Entries with hallucinated tool names (not in the valid tool list derived from `model_tools.TOOL_TO_TOOLSET_MAP`) are filtered out
- **Reasoning statistics:** Tracks percentage of turns with/without reasoning across the entire run

## Statistics

After completion, the runner prints comprehensive statistics and saves them to `statistics.json`:

- **Tool usage:** Call counts, success/failure rates per tool (sorted by count descending)
- **Reasoning coverage:** Percentage of assistant turns with and without reasoning
- **Samples discarded:** Count of samples filtered for lacking reasoning
- **Duration:** Total processing time

Example statistics output:

```
Tool Usage Statistics:
----------------------------------------------------------------------
Tool Name                 Count      Success    Failure    Success Rate
----------------------------------------------------------------------
terminal                  1523       1489       34         97.8%
read_file                 876        876        0          100.0%
web_search                234        221        13         94.4%

Reasoning Coverage:
----------------------------------------------------------------------
   Total assistant turns:    4821
   With reasoning:           4756 (98.7%)
   Without reasoning:        65 (1.3%)
   Samples discarded (zero reasoning): 12
```

## Use Cases

### Training Data Generation

```bash
python batch_runner.py \
    --dataset_file=data/coding_prompts.jsonl \
    --batch_size=20 \
    --run_name=coding_v1 \
    --model=anthropic/claude-sonnet-4-20250514 \
    --num_workers=8 \
    --distribution=default \
    --max_turns=15
```

### Model Evaluation

```bash
python batch_runner.py \
    --dataset_file=data/eval_suite.jsonl \
    --batch_size=10 \
    --run_name=eval_gpt4 \
    --model=openai/gpt-4o \
    --num_workers=4 \
    --max_turns=10
```

### With Disabled Reasoning

```bash
python batch_runner.py \
    --dataset_file=data/prompts.jsonl \
    --batch_size=10 \
    --run_name=no_reasoning \
    --reasoning_disabled \
    --max_tokens=128000
```

### With Few-Shot Prefill Messages

```bash
python batch_runner.py \
    --dataset_file=data/prompts.jsonl \
    --batch_size=10 \
    --run_name=my_run \
    --prefill_messages_file=configs/prefill_opus.json
```

The prefill messages file must contain a JSON array of `{role, content}` dicts.

## Implementation Details

The `BatchRunner` class initializes with all configuration, loads the dataset, creates batches, and manages checkpoints. Processing is parallelized using Python's `multiprocessing.Pool` with `imap_unordered` for efficient parallel execution. Each batch processes prompts sequentially within a worker process. A `multiprocessing.Lock` protects checkpoint writes from concurrent worker updates.

Context files (`SOUL.md`, `AGENTS.md`) are skipped during batch runs (`skip_context_files=True`) to prevent local project context from contaminating trajectories. Persistent memory is also skipped (`skip_memory=True`).
