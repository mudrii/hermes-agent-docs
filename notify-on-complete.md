# Notify on Complete

The `notify_on_complete` parameter for the `terminal()` tool lets the agent request an automatic notification when a background process exits. When set, the agent does not need to poll for completion — it is triggered automatically with the process results.

## Basic usage

```python
terminal(
    command="python train.py --epochs 50",
    background=True,
    notify_on_complete=True,
)
```

`notify_on_complete` only has effect when `background=True`. When the process exits (success or failure), a new agent turn is triggered automatically with the process output and exit status.

Without `notify_on_complete`, the agent must poll manually:

```python
# Manual polling — check periodically
process(action="poll", session_id="proc_abc123")

# Or block until done
process(action="wait", session_id="proc_abc123", timeout=3600)
```

With `notify_on_complete`, neither of those calls is needed. The agent is notified automatically.

## How it works

### ProcessRegistry and completion_queue

Every background process is tracked by `ProcessRegistry` in `tools/process_registry.py`. When `notify_on_complete=True` is set on a `ProcessSession`, the registry pushes the session ID onto `ProcessRegistry.completion_queue` when the process exits.

After each agent turn, the CLI event loop and the gateway drain `completion_queue`. Any pending completions trigger a new agent turn with the process output attached — effectively a background callback.

### ProcessSession fields

```python
@dataclass
class ProcessSession:
    notify_on_complete: bool = False  # Queue agent notification on exit
```

The field defaults to `False`. The terminal tool sets it to `True` only when the caller passes `notify_on_complete=True` AND `background=True`.

### Output buffer

Each background process keeps a rolling output buffer capped at 200 KB (`MAX_OUTPUT_CHARS = 200_000`). Output beyond 200 KB is dropped from the head, keeping the most recent output. The completion notification includes whatever is in the buffer at the time the process exits.

### Process tracking limits

| Constant | Value | Meaning |
|----------|-------|---------|
| `MAX_OUTPUT_CHARS` | 200,000 | Rolling output buffer per process |
| `FINISHED_TTL_SECONDS` | 1800 | Finished processes kept for 30 minutes |
| `MAX_PROCESSES` | 64 | Max concurrent tracked processes (LRU pruning) |

## Crash recovery

`ProcessRegistry` persists a checkpoint to `~/.hermes/processes.json` on every status change. On restart, Hermes reads the checkpoint file and restores the process list, including `notify_on_complete` state. If a process was still running at shutdown and completed while Hermes was down, the completion is queued on the next startup.

## Execution environment

Background processes run through the active terminal backend — they are not special-cased. Which environment a background process runs in depends on the configured `terminal.backend`:

| Backend | Where the process runs |
|---------|----------------------|
| `local` | Directly on the host machine |
| `docker` | Inside the persistent Docker container |
| `ssh` | On the remote SSH host |
| `singularity` | Inside the Singularity container |
| `modal` | In a Modal serverless cloud sandbox |
| `daytona` | In a Daytona cloud workspace |

Only `TERMINAL_ENV=local` (equivalent to `terminal.backend: local`) runs processes directly on the host. All other backends route commands through the environment interface — background processes live inside the sandbox, not on the machine running Hermes.

This matters for file access: a background training job in a Docker container writes its checkpoints inside the container filesystem. If you need those files on the host, configure a bind mount or use the appropriate backend file-transfer mechanism.

## Use cases

### AI training runs

```python
terminal(
    command="python train.py --model llama-3b --dataset /data/train.jsonl --output /runs/exp1",
    background=True,
    notify_on_complete=True,
)
# Agent is notified when training finishes or crashes, with the last 200KB of output
```

### Test suites

```python
terminal(
    command="pytest tests/ -v --tb=short 2>&1 | tee /tmp/test-results.txt",
    background=True,
    notify_on_complete=True,
)
```

### Build pipelines

```python
terminal(
    command="make release 2>&1",
    background=True,
    notify_on_complete=True,
)
```

### Deployments

```python
terminal(
    command="./deploy.sh production",
    background=True,
    notify_on_complete=True,
)
```

## Managing background processes

While a background process runs, you can check on it or interact with it:

```python
process(action="list")                          # All tracked processes
process(action="poll", session_id="proc_abc")   # Latest status + new output
process(action="log", session_id="proc_abc")    # Full output with pagination
process(action="write", session_id="proc_abc", data="y\n")  # Send stdin
process(action="kill", session_id="proc_abc")   # Terminate
```

`notify_on_complete` and manual `process()` calls are compatible — you can still poll while also having a completion callback set.

## Session-scoped tracking

Each Hermes session has its own process namespace. Processes started in one session are not visible from another session's `process(action="list")`. This isolation prevents sessions from accidentally interfering with each other's background work.

The checkpoint file (`~/.hermes/processes.json`) persists process metadata across restarts but does not share process state between concurrent sessions.
