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

## Inactivity-Based Timeout (v0.8.0)

Prior to v0.8.0, cron jobs were killed after a fixed wall-clock duration regardless of whether the agent was actively working. As of v0.8.0 (PR #5440), the timeout is inactivity-based:

- The scheduler polls the running agent every few seconds.
- If the agent has made at least one tool call since the last poll, the inactivity counter resets.
- Only when the agent is truly idle (no tool call activity) for longer than `HERMES_CRON_TIMEOUT` seconds (default: 600) is it killed.
- Set `HERMES_CRON_TIMEOUT=0` to disable the timeout entirely.

This allows long-running but actively working tasks to complete naturally.

## Delivery Failure Tracking (v0.8.0)

As of v0.8.0 (PR #6042), `mark_job_run()` accepts a `delivery_error` parameter separate from the agent `error`. Jobs can succeed at the agent level (output was produced) while failing at the delivery level (platform down, invalid chat ID). The `last_delivery_error` field in the job record stores the delivery failure reason and is cleared on the next successful delivery.

## MEDIA Tag Extraction Before Delivery (v0.8.0)

Before cron delivery is sent to a platform, `MEDIA:` tags in the agent's output are extracted via `BasePlatformAdapter.extract_media()`. Media files are forwarded as native platform attachments; the text portion has the tags stripped. This avoids raw `MEDIA:` tag strings appearing in delivered messages.

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
