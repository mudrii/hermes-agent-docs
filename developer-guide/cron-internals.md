# Cron Internals

Hermes cron support allows scheduled agent tasks with full tool access, skill injection, and multi-platform delivery.

Primary files: `cron/jobs.py`, `cron/scheduler.py`, `tools/cronjob_tools.py`, `gateway/run.py`, `hermes_cli/cron.py`

## Scheduling Model

Hermes supports:

- One-shot delays
- Intervals
- Cron expressions
- Explicit timestamps

The model-facing surface is a single `cronjob` tool with action-style operations: `create`, `list`, `update`, `pause`, `resume`, `run`, `remove`.

## Job Storage

Cron jobs are stored in `~/.hermes/cron/jobs.json` with atomic write semantics.

Each job can carry:

- Prompt
- Schedule metadata
- Repeat counters
- Delivery target
- Lifecycle state (`scheduled`, `paused`, `completed`, etc.)
- Zero, one, or multiple attached skills

Backward compatibility is preserved for older jobs that only stored a legacy single `skill` field or none of the newer lifecycle fields.

## Runtime Behavior

The scheduler:

1. Loads jobs
2. Computes due work
3. Executes jobs in fresh agent sessions
4. Optionally injects one or more skills before the prompt
5. Handles repeat counters
6. Updates next-run metadata and state

In gateway mode, cron ticking is integrated into the long-running gateway loop with a 60-second tick interval.

## Skill-Backed Jobs

A cron job may attach multiple skills. At runtime, Hermes loads those skills in order and then appends the job prompt as the task instruction. This gives scheduled jobs reusable guidance without requiring the user to paste full skill bodies into the cron prompt.

## Recursion Guard

Cron-run sessions disable the `cronjob` toolset. This prevents a scheduled job from recursively creating or mutating more cron jobs, which could cause token usage or scheduler load to explode.

## Delivery Model

Cron jobs can deliver to:

- Origin chat
- Local files
- Platform home channels
- Explicit platform/chat IDs

## Locking

Hermes uses lock-based protections so overlapping scheduler ticks do not execute the same due-job batch twice.
