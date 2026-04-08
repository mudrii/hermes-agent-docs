# Trajectory Format

Hermes Agent saves conversation trajectories in ShareGPT-compatible JSONL format for use as training data, debugging artifacts, and reinforcement learning datasets.

Source files: `agent/trajectory.py`, `run_agent.py`, `batch_runner.py`, `trajectory_compressor.py`

## File Naming Convention

| File | When |
|------|------|
| `trajectory_samples.jsonl` | Conversations that completed successfully (`completed=True`) |
| `failed_trajectories.jsonl` | Conversations that failed or were interrupted (`completed=False`) |

The batch runner writes to a custom output file per batch (e.g. `batch_001_output.jsonl`) with additional metadata fields.

## JSONL Entry Format

Each line in the file is a self-contained JSON object.

### CLI/Interactive Format

```json
{
  "conversations": [ ... ],
  "timestamp": "2026-03-30T14:22:31.456789",
  "model": "anthropic/claude-sonnet-4.6",
  "completed": true
}
```

### Batch Runner Format

```json
{
  "prompt_index": 42,
  "conversations": [ ... ],
  "metadata": { "prompt_source": "gsm8k", "difficulty": "hard" },
  "completed": true,
  "partial": false,
  "api_calls": 7,
  "toolsets_used": ["code_tools", "file_tools"],
  "tool_stats": {
    "terminal": {"count": 3, "success": 3, "failure": 0},
    "read_file": {"count": 2, "success": 2, "failure": 0}
  },
  "tool_error_counts": {
    "terminal": 0,
    "read_file": 0
  }
}
```

The `tool_stats` and `tool_error_counts` dictionaries are normalized to include ALL possible tools (from `model_tools.TOOL_TO_TOOLSET_MAP`) with zero defaults, ensuring consistent schema across entries for HuggingFace dataset loading.

## Conversations Array (ShareGPT Format)

The `conversations` array uses ShareGPT role conventions:

| API Role | ShareGPT `from` |
|----------|-----------------|
| system | `"system"` |
| user | `"human"` |
| assistant | `"gpt"` |
| tool | `"tool"` |

## Normalization Rules

### Reasoning Content Markup

The trajectory converter normalizes ALL reasoning into `<think>` tags, regardless of how the model originally produced it:

1. **Native thinking tokens** (from providers like Anthropic, OpenAI o-series): Wrapped as `<think>\n{reasoning}\n</think>\n` and prepended before the content
2. **REASONING_SCRATCHPAD XML** (when native thinking is disabled): `<REASONING_SCRATCHPAD>` tags are converted to `<think>` via `convert_scratchpad_to_think()`
3. **Empty think blocks**: Every `gpt` turn is guaranteed to have a `<think>` block. If no reasoning was produced, an empty block is inserted -- this ensures consistent format for training data

### Tool Call Normalization

Tool calls from the API format are converted to XML-wrapped JSON:

```
<tool_call>
{"name": "terminal", "arguments": {"command": "ls -la"}}
</tool_call>
```

Arguments are parsed from JSON strings back to objects (not double-encoded). Multiple tool calls in one assistant turn produce multiple `<tool_call>` blocks in a single `gpt` message.

### Tool Response Normalization

All tool results following an assistant message are grouped into a single `tool` turn with XML-wrapped JSON responses:

```
<tool_response>
{"tool_call_id": "call_abc123", "name": "terminal", "content": "output here"}
</tool_response>
```

If tool content looks like JSON, it is parsed so the content field contains a JSON object/array rather than a string. Multiple tool results are joined with newlines.

### System Message

The system message is generated at save time (not taken from the conversation). It follows the Hermes function-calling prompt template with a preamble, `<tools>` XML block containing the JSON tool definitions, schema reference for `FunctionCall` objects, and a `<tool_call>` example.

## Persistence Boundaries

Trajectory files do not blindly mirror all runtime prompt state. The `ephemeral_system_prompt` is intentionally excluded so datasets are cleaner and less environment-specific.

## Loading Trajectories

```python
import json

def load_trajectories(path: str):
    entries = []
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                entries.append(json.loads(line))
    return entries

successful = [e for e in load_trajectories("trajectory_samples.jsonl")
              if e.get("completed")]

training_data = [e["conversations"] for e in successful]
```

### Loading for HuggingFace Datasets

```python
from datasets import load_dataset

ds = load_dataset("json", data_files="trajectory_samples.jsonl")
```

## Controlling Trajectory Saving

```yaml
# config.yaml
agent:
  save_trajectories: true  # default: false
```

Or via the `--save-trajectories` flag. When the agent initializes with `save_trajectories=True`, the `_save_trajectory()` method is called at the end of each conversation turn.

The batch runner always saves trajectories. Samples with zero reasoning across all turns are automatically discarded to avoid polluting training data with non-reasoning examples.

## Version History

**v0.8.0 (v2026.4.8):** The source reference for `run_agent.py` was updated to use function name search (`_save_trajectory`) rather than stale line numbers. No format changes to trajectory entries.
