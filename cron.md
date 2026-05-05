
# Scheduled Tasks (Cron)

Schedule tasks to run automatically with natural language or cron expressions. Hermes exposes cron management through a single `cronjob` tool with action-style operations instead of separate schedule/list/remove tools.

## What cron can do now

Cron jobs can:

- schedule one-shot or recurring tasks
- pause, resume, edit, trigger, and remove jobs
- attach zero, one, or multiple skills to a job
- deliver results back to the origin chat, local files, or configured platform targets
- run in fresh agent sessions with the normal static tool list
- restrict each job to a specific subset of toolsets via per-job `enabled_toolsets` (v0.11.0)
- skip the LLM entirely when a pre-run script returns `{"wakeAgent": false}` (v0.11.0)

:::warning
Cron-run sessions cannot recursively create more cron jobs. Hermes disables cron management tools inside cron executions to prevent runaway scheduling loops.
:::

## Creating scheduled tasks

### In chat with `/cron`

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

Ask Hermes normally:

```text
Every morning at 9am, check Hacker News for AI news and send me a summary on Telegram.
```

Hermes will use the unified `cronjob` tool internally.

## Skill-backed cron jobs

A cron job can load one or more skills before it runs the prompt.

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

Skills are loaded in order. The prompt becomes the task instruction layered on top of those skills.

```python
cronjob(
    action="create",
    skills=["blogwatcher", "find-nearby"],
    prompt="Look for new local events and interesting nearby places, then combine them into one short brief.",
    schedule="every 6h",
    name="Local brief",
)
```

This is useful when you want a scheduled agent to inherit reusable workflows without stuffing the full skill text into the cron prompt itself.

## Editing jobs

You do not need to delete and recreate jobs just to change them.

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

Notes:

- repeated `--skill` replaces the job's attached skill list
- `--add-skill` appends to the existing list without replacing it
- `--remove-skill` removes specific attached skills
- `--clear-skills` removes all attached skills

## Lifecycle actions

Cron jobs now have a fuller lifecycle than just create/remove.

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

What they do:

- `pause` — keep the job but stop scheduling it
- `resume` — re-enable the job and compute the next future run
- `run` — trigger the job on the next scheduler tick
- `remove` — delete it entirely

## How it works

**Cron execution is handled by the gateway daemon.** The gateway ticks the scheduler every 60 seconds, running any due jobs in isolated agent sessions.

```bash
hermes gateway install     # Install as a user service
sudo hermes gateway install --system   # Linux: boot-time system service for servers
hermes gateway             # Or run in foreground

hermes cron list
hermes cron status
```

### Gateway scheduler behavior

On each tick Hermes:

1. loads jobs from `~/.hermes/cron/jobs.json`
2. checks `next_run_at` against the current time
3. starts a fresh `AIAgent` session for each due job
4. optionally injects one or more attached skills into that fresh session
5. runs the prompt to completion
6. delivers the final response
7. updates run metadata and the next scheduled time

A file lock at `~/.hermes/cron/.tick.lock` prevents overlapping scheduler ticks from double-running the same job batch.

## Delivery options

When scheduling jobs, you specify where the output goes:

| Option | Description | Example |
|--------|-------------|---------|
| `"origin"` | Back to where the job was created | Default on messaging platforms |
| `"local"` | Save to local files only (`~/.hermes/cron/output/`) | Default on CLI |
| `"telegram"` | Telegram home channel | Uses `TELEGRAM_HOME_CHANNEL` |
| `"telegram:123456"` | Specific Telegram chat by ID | Direct delivery |
| `"telegram:-100123:17585"` | Specific Telegram topic | `chat_id:thread_id` format |
| `"discord"` | Discord home channel | Uses `DISCORD_HOME_CHANNEL` |
| `"discord:#engineering"` | Specific Discord channel | By channel name |
| `"slack"` | Slack home channel | |
| `"whatsapp"` | WhatsApp home | |
| `"signal"` | Signal | |
| `"matrix"` | Matrix home room | |
| `"mattermost"` | Mattermost home channel | |
| `"email"` | Email | |
| `"sms"` | SMS via Twilio | |
| `"homeassistant"` | Home Assistant | |
| `"dingtalk"` | DingTalk | |
| `"feishu"` | Feishu/Lark | |
| `"wecom"` | WeCom | |
| `"weixin"` | Weixin (WeChat) | |
| `"bluebubbles"` | BlueBubbles (iMessage) | |

The agent's final response is automatically delivered. You do not need to call `send_message` in the cron prompt.

### Response wrapping

By default, delivered cron output is wrapped with a header and footer so the recipient knows it came from a scheduled task:

```
Cronjob Response: Morning feeds
-------------

<agent output here>

Note: The agent cannot see this message, and therefore cannot respond to it.
```

To deliver the raw agent output without the wrapper, set `cron.wrap_response` to `false`:

```yaml
# ~/.hermes/config.yaml
cron:
  wrap_response: false
```

### Silent suppression

If the agent's final response starts with `[SILENT]`, delivery is suppressed entirely. The output is still saved locally for audit (in `~/.hermes/cron/output/`), but no message is sent to the delivery target.

This is useful for monitoring jobs that should only report when something is wrong:

```text
Check if nginx is running. If everything is healthy, respond with only [SILENT].
Otherwise, report the issue.
```

Failed jobs always deliver regardless of the `[SILENT]` marker — only successful runs can be silenced.

## Script timeout

Pre-run scripts (attached via the `script` parameter) have a default timeout of 120 seconds. If your scripts need longer — for example, to include randomized delays that avoid bot-like timing patterns — you can increase this:

```yaml
# ~/.hermes/config.yaml
cron:
  script_timeout_seconds: 300   # 5 minutes
```

Or set the `HERMES_CRON_SCRIPT_TIMEOUT` environment variable. The resolution order is: env var → config.yaml → 120s default.

## Skipping the agent with `wakeAgent` (v0.11.0)

A cron job's pre-run `script` can decide whether the LLM runs at all. If the **last non-empty stdout line** of the script is JSON in the form `{"wakeAgent": false}`, the scheduler skips the agent entirely — no LLM call, no tool calls, no delivery (just a local audit log entry).

Any other output — non-JSON, missing flag, or `{"wakeAgent": true}` — wakes the agent normally and the script's stdout is injected into the prompt as `## Script Output` context.

This is useful for high-frequency monitoring jobs that should only consume an LLM turn when something actually changed:

```bash
#!/usr/bin/env bash
# precheck.sh — only wake the agent if there are unread alerts.
set -euo pipefail

unread=$(curl -fsS https://example.com/api/alerts/unread | jq '.count')
if [ "$unread" -eq 0 ]; then
  echo '{"wakeAgent": false}'
  exit 0
fi

# Otherwise, emit the data the agent should reason over and wake it.
curl -fsS https://example.com/api/alerts/unread > /dev/stdout
echo '{"wakeAgent": true}'
```

Then attach the script to the job:

```python
cronjob(
    action="create",
    prompt="Summarize the unread alerts and post them to #ops.",
    schedule="every 5m",
    script="precheck.sh",
    delivery="slack:#ops",
)
```

This collapses cost on idle ticks while keeping the same prompt/skill/delivery wiring for the cases that do need the agent.

## Per-job toolsets (v0.11.0)

By default, cron jobs inherit the toolset list configured for the **`cron` platform** in `hermes tools` (which already trims a few default-off toolsets like `moa`, `homeassistant`, and `rl` to keep idle costs down). You can override that list per job via `enabled_toolsets` to cap the system-prompt size and tool surface for individual jobs:

```python
cronjob(
    action="create",
    prompt="Pull the latest hourly metrics and post them to #ops.",
    schedule="every 1h",
    enabled_toolsets=["core", "web_research", "messaging"],
    delivery="slack:#ops",
)
```

Resolution precedence used by the scheduler:

1. **Per-job `enabled_toolsets`** (set via `cronjob` create/update) — highest priority.
2. **Per-platform `hermes tools` config for the `cron` platform** — applies to all jobs without an explicit override.
3. **Full default toolset** — fallback only if both lookups fail.

Use this to keep token overhead and per-tick cost predictable on jobs that only need a small tool surface (e.g. a metrics fetcher that just needs `core` + one messaging toolset).

## Provider recovery

Cron jobs inherit your configured fallback providers and credential pool rotation. If the primary API key is rate-limited or the provider returns an error, the cron agent can:

- **Fall back to an alternate provider** if you have `fallback_providers` (or the legacy `fallback_model`) configured in `config.yaml`
- **Rotate to the next credential** in your [credential pool](/docs/user-guide/configuration#credential-pool-strategies) for the same provider

This means cron jobs that run at high frequency or during peak hours are more resilient — a single rate-limited key won't fail the entire run.

## Schedule formats

The agent's final response is automatically delivered — you do **not** need to include `send_message` in the cron prompt for that same destination. If a cron run calls `send_message` to the exact target the scheduler will already deliver to, Hermes skips that duplicate send and tells the model to put the user-facing content in the final response instead. Use `send_message` only for additional or different targets.

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

### ISO timestamps

```text
2026-03-15T09:00:00    → One-time at March 15, 2026 9:00 AM
```

## Repeat behavior

| Schedule type | Default repeat | Behavior |
|--------------|----------------|----------|
| One-shot (`30m`, timestamp) | 1 | Runs once |
| Interval (`every 2h`) | forever | Runs until removed |
| Cron expression | forever | Runs until removed |

You can override it:

```python
cronjob(
    action="create",
    prompt="...",
    schedule="every 2h",
    repeat=5,
)
```

## Managing jobs programmatically

The agent-facing API is one tool:

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

### Full create/update signature (v0.11.0)

In addition to `prompt`, `schedule`, `name`, `skill(s)`, `script`, and `delivery`, `cronjob` accepts:

```python
cronjob(
    action="create",
    prompt="Summarize new feed items.",
    schedule="every 1h",
    name="Hourly digest",
    skills=["blogwatcher"],
    script="precheck.sh",             # Relative to ~/.hermes/scripts/
    enabled_toolsets=[                # v0.11.0: restrict this job to a subset
        "core",
        "web_research",
        "messaging",
    ],
    repeat=24,                         # Optional: cap recurrences
    delivery="telegram",
)
```

Pass the same arguments to `action="update"` to change them on an existing job. Pass `enabled_toolsets=[]` to clear the per-job override and fall back to the global cron toolset config.

## Job storage

Jobs are stored in `~/.hermes/cron/jobs.json`. Output from job runs is saved to `~/.hermes/cron/output/{job_id}/{timestamp}.md`.

The storage uses atomic file writes so interrupted writes do not leave a partially written job file behind.

## Self-contained prompts still matter

:::warning Important
Cron jobs run in a completely fresh agent session. The prompt must contain everything the agent needs that is not already provided by attached skills.
:::

**BAD:** `"Check on that server issue"`

**GOOD:** `"SSH into server 192.168.1.100 as user 'deploy', check if nginx is running with 'systemctl status nginx', and verify https://example.com returns HTTP 200."`

## Security

Scheduled task prompts are scanned for prompt-injection and credential-exfiltration patterns at creation and update time. Prompts containing invisible Unicode tricks, SSH backdoor attempts, or obvious secret-exfiltration payloads are blocked.
