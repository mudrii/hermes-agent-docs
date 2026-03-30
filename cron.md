# Scheduled Tasks (Cron)

Hermes provides a built-in cron system for scheduling tasks to run automatically. Jobs run in fresh agent sessions and can deliver results back to messaging platforms, local files, or specific channels.

## Overview

The cron system consists of:

- **Job storage** — `~/.hermes/cron/jobs.json` stores all scheduled jobs
- **Scheduler** — `cron/scheduler.py` provides the `tick()` function that checks for due jobs
- **Gateway integration** — the gateway daemon calls `tick()` every 60 seconds
- **Output storage** — `~/.hermes/cron/output/{job_id}/{timestamp}.md` stores job run output

The gateway must be running for scheduled tasks to execute. Run it with:

```bash
hermes gateway           # run in foreground
hermes gateway install   # install as a user service (runs at login)
sudo hermes gateway install --system   # Linux: install as boot-time system service
```

## Creating Scheduled Tasks

### In chat with /cron

```bash
/cron add 30m "Remind me to check the build"
/cron add "every 2h" "Check server status"
/cron add "every 1h" "Summarize new feed items" --skill blogwatcher
/cron add "every 1h" "Use both skills and combine the result" --skill blogwatcher --skill find-nearby
```

### From the standalone CLI

```bash
hermes cron create "every 2h" "Check server status"
hermes cron create "every 1h" "Summarize new feed items" --skill blogwatcher
hermes cron create "every 1h" "Use both skills and combine the result" \
  --skill blogwatcher \
  --skill find-nearby \
  --name "Skill combo"
```

### Through natural conversation

```text
Every morning at 9am, check Hacker News for AI news and send me a summary on Telegram.
```

Hermes uses the unified `cronjob` tool internally.

## Schedule Formats

### Relative delays (one-shot)

```text
30m     → Run once in 30 minutes
2h      → Run once in 2 hours
1d      → Run once in 1 day
```

### Intervals (recurring)

```text
every 30m    → Every 30 minutes
every 2h     → Every 2 hours
every 1d     → Every day
```

### Cron expressions

```text
0 9 * * *       → Daily at 9:00 AM
0 9 * * 1-5     → Weekdays at 9:00 AM
0 */6 * * *     → Every 6 hours
30 8 1 * *      → First of every month at 8:30 AM
0 0 * * 0       → Every Sunday at midnight
```

Cron expressions require the `croniter` package: `pip install croniter`

### ISO timestamps (one-shot at specific time)

```text
2026-03-15T09:00:00    → One-time at March 15, 2026 9:00 AM
```

## Repeat Behavior

| Schedule type | Default repeat | Behavior |
|--------------|----------------|----------|
| One-shot (`30m`, timestamp) | 1 | Runs once |
| Interval (`every 2h`) | forever | Runs until removed |
| Cron expression | forever | Runs until removed |

Override with an explicit `repeat` count:

```python
cronjob(
    action="create",
    prompt="...",
    schedule="every 2h",
    repeat=5,          # run exactly 5 times then auto-delete
)
```

## Skill-Backed Cron Jobs

A cron job can load one or more skills before running the prompt. Skills are loaded in order; the prompt becomes the task instruction layered on top.

### Single skill

```python
cronjob(
    action="create",
    skill="blogwatcher",
    prompt="Check the configured feeds and summarize anything new.",
    schedule="0 9 * * *",
    name="Morning feeds",
)
```

### Multiple skills

```python
cronjob(
    action="create",
    skills=["blogwatcher", "find-nearby"],
    prompt="Look for new local events and interesting nearby places, then combine them into one short brief.",
    schedule="every 6h",
    name="Local brief",
)
```

## Delivery Options

| Option | Description | Example |
|--------|-------------|---------|
| `"origin"` | Back to where the job was created | Default on messaging platforms |
| `"local"` | Save to local files only (`~/.hermes/cron/output/`) | Default on CLI |
| `"telegram"` | Telegram home channel | Uses `TELEGRAM_HOME_CHANNEL` |
| `"discord"` | Discord home channel | Uses `DISCORD_HOME_CHANNEL` |
| `"telegram:123456"` | Specific Telegram chat by ID | Direct delivery |
| `"discord:987654"` | Specific Discord channel by ID | Direct delivery |

The agent's final response is automatically delivered. You do not need to call `send_message` in the cron prompt for the same destination. Use `send_message` only for additional or different targets.

If the agent's response starts with `[SILENT]`, delivery is suppressed. Output is still saved locally. This allows cron agents to opt out of delivery when there is nothing new to report.

## Job Lifecycle Actions

### Chat

```bash
/cron list
/cron pause <job_id>
/cron resume <job_id>
/cron run <job_id>
/cron remove <job_id>
```

### Standalone CLI

```bash
hermes cron list
hermes cron pause <job_id>
hermes cron resume <job_id>
hermes cron run <job_id>
hermes cron remove <job_id>
hermes cron status
hermes cron tick
```

- `pause` — keep the job but stop scheduling it; sets `state: "paused"` and `enabled: false`; records `paused_at` timestamp and optional `paused_reason`
- `resume` — re-enable the job and compute the next future run from now; clears pause metadata
- `run` — trigger the job on the next scheduler tick by setting `next_run_at` to now; also clears pause state if the job was paused
- `remove` — delete the job entirely

## Editing Jobs

Jobs can be edited without deleting and recreating them.

### Chat

```bash
/cron edit <job_id> --schedule "every 4h"
/cron edit <job_id> --prompt "Use the revised task"
/cron edit <job_id> --skill blogwatcher --skill find-nearby
/cron edit <job_id> --remove-skill blogwatcher
/cron edit <job_id> --clear-skills
```

### Standalone CLI

```bash
hermes cron edit <job_id> --schedule "every 4h"
hermes cron edit <job_id> --prompt "Use the revised task"
hermes cron edit <job_id> --skill blogwatcher --skill find-nearby
hermes cron edit <job_id> --add-skill find-nearby
hermes cron edit <job_id> --remove-skill blogwatcher
hermes cron edit <job_id> --clear-skills
```

Skill edit notes:
- Repeated `--skill` replaces the job's entire attached skill list
- `--add-skill` appends to the existing list without replacing
- `--remove-skill` removes a specific skill
- `--clear-skills` removes all attached skills

## Programmatic API (cronjob tool)

The agent-facing API uses one tool with action-style operations:

```python
cronjob(action="create", ...)
cronjob(action="list")
cronjob(action="update", job_id="...")
cronjob(action="pause", job_id="...")
cronjob(action="resume", job_id="...")
cronjob(action="run", job_id="...")
cronjob(action="remove", job_id="...")
```

For `update`, pass `skills=[]` to remove all attached skills.

## How Jobs Are Executed

The gateway ticks the scheduler every 60 seconds via `cron.scheduler.tick()`. On each tick:

1. Loads jobs from `~/.hermes/cron/jobs.json`
2. Checks `next_run_at` against the current time
3. For recurring jobs that are more than 2 minutes stale (gateway was down), fast-forwards to the next future occurrence instead of firing a missed run
4. Starts a fresh `AIAgent` session for each due job with `disabled_toolsets=["cronjob"]` (cron agents cannot create more cron jobs)
5. Optionally injects one or more attached skills into the session prompt
6. Runs the prompt to completion
7. Delivers the final response to the configured target
8. Updates `last_run_at`, `last_status`, and `next_run_at`
9. Auto-deletes one-shot jobs after they complete their `repeat` count

A file lock at `~/.hermes/cron/.tick.lock` prevents overlapping ticks from double-running the same job batch. The lock uses `fcntl` on Unix and `msvcrt` on Windows.

One-shot jobs get a 120-second grace period (`ONESHOT_GRACE_SECONDS`). If a job is created a few seconds after its requested time, it still fires on the next tick as long as the scheduled time is within 2 minutes of now.

The cron scheduler re-reads `~/.hermes/.env` and `config.yaml` fresh on every run, so provider/key changes take effect without a gateway restart.

Each cron job session initializes its own SQLite session store so messages are persisted and discoverable via `session_search`.

## Job Storage Format

Jobs are stored in `~/.hermes/cron/jobs.json` as a JSON object with a `jobs` array. Each job contains:

```json
{
  "id": "abc123def456",
  "name": "Morning feeds",
  "prompt": "Check feeds and summarize",
  "skills": ["blogwatcher"],
  "skill": "blogwatcher",
  "model": null,
  "provider": null,
  "base_url": null,
  "schedule": {"kind": "cron", "expr": "0 9 * * *", "display": "0 9 * * *"},
  "schedule_display": "0 9 * * *",
  "repeat": {"times": null, "completed": 0},
  "enabled": true,
  "state": "scheduled",
  "paused_at": null,
  "paused_reason": null,
  "created_at": "2026-03-19T10:00:00",
  "next_run_at": "2026-03-20T09:00:00",
  "last_run_at": null,
  "last_status": null,
  "last_error": null,
  "deliver": "origin",
  "origin": {"platform": "telegram", "chat_id": "123456"}
}
```

Storage uses atomic writes (`tempfile.mkstemp` + `os.replace`) so interrupted writes never leave a partially written jobs file. Cron directories and files are created with owner-only permissions (`0700` for directories, `0600` for files).

## Output Files

Job run output is saved to `~/.hermes/cron/output/{job_id}/{timestamp}.md`. Each output file is a Markdown document containing:

- Job name and ID
- Run time and schedule
- The prompt used (including skill content if applicable)
- The agent's response (or error traceback)

## Per-Job Model Override

A cron job can use a different model than the gateway's default:

```python
cronjob(
    action="create",
    prompt="...",
    schedule="every 2h",
    model="google/gemini-flash-2.0",
    provider="openrouter",
)
```

## Self-Contained Prompts

Cron jobs run in a completely fresh agent session. The prompt must contain everything the agent needs that is not already provided by attached skills.

**BAD:** `"Check on that server issue"`

**GOOD:** `"SSH into server 192.168.1.100 as user 'deploy', check if nginx is running with 'systemctl status nginx', and verify https://example.com returns HTTP 200."`

## Security

Scheduled task prompts are scanned for prompt-injection and credential-exfiltration patterns at creation and update time. Prompts containing invisible Unicode tricks, SSH backdoor attempts, or obvious secret-exfiltration payloads are blocked.

Cron agents run with `disabled_toolsets=["cronjob"]` — they cannot recursively create more cron jobs, preventing runaway scheduling loops.

## REST API for Cron Management

The gateway's API server (when enabled) exposes a REST API for cron job management at `/api/jobs`. This provides programmatic create, read, update, delete, pause, resume, and trigger operations over HTTP using the same Bearer token auth as the OpenAI endpoints.

See [api-server.md](api-server.md#cron-jobs-rest-api) for the full endpoint reference.

## What's New

### v0.4.0

- `/api/jobs` REST API introduced via the API server platform adapter (PR #1756, #2450). Provides full CRUD + pause/resume/run operations over HTTP.

### v0.5.0

- **Prevent recurring job re-fire on gateway crash/restart loop** (PR #3396) — For recurring cron jobs (interval and cron-expression schedules), `next_run_at` is now advanced to the next future occurrence _before_ the job executes. If the gateway crashes mid-run, the job will not re-fire immediately on restart. One-shot jobs are left unchanged so they can retry on restart.
- **Mark cron session as ended after job completes** (PR #2998) — Cron job sessions are now properly closed in the session database after execution.
- **Auto-repair `jobs.json` with invalid control characters** (PR #3537) — If `jobs.json` contains bare control characters (from a crash or corruption), the scheduler automatically repairs and rewrites the file on the next load.
