# Memory & Persistence

Hermes implements a multi-tier memory architecture that persists knowledge across sessions, devices, and restarts.

---

## Memory Architecture

| Tier | Storage | Scope | Purpose |
|------|---------|-------|---------|
| **Inference memory** | Context window | Current turn | Standard LLM context |
| **Procedural skills** | `~/.hermes/skills/` | All sessions | Step-by-step procedures the agent creates |
| **Session history** | SQLite (FTS5) | All sessions on device | Full conversation search |
| **User model** | Honcho API | Cross-device, cross-session | Persistent user understanding |

---

## Session Storage (SQLite)

**File:** `~/.hermes/state.db`

All sessions, messages, and token counts are persisted immediately to SQLite with WAL mode for concurrent access.

### Session Database Schema

```sql
sessions(
    id, source, user_id, model, system_prompt,
    parent_session_id,      -- lineage chain for compressed sessions
    started_at, ended_at, end_reason,
    message_count, tool_call_count,
    input_tokens, output_tokens, title
)

messages(
    id, session_id, role, content,
    tool_call_id, tool_calls, tool_name,
    timestamp, token_count, finish_reason
)

messages_fts    -- FTS5 virtual table, auto-indexed
```

### Searching Past Sessions

```bash
# Via CLI
/session_search "how did I fix that docker issue"

# Via tool (called by agent)
session_search(query="docker networking", max_results=5, context_messages=3)
```

Returns summarized excerpts from matching conversations.

### Session Commands

```bash
hermes sessions list               # List recent sessions
hermes sessions browse             # Interactive browser
hermes sessions view <id>          # View messages
hermes sessions export <id>        # Export to JSONL
hermes sessions export-all         # Export everything
hermes sessions prune --days 90    # Delete old sessions
```

### Session Resume

```bash
hermes --session abc123            # Resume by ID prefix
hermes --session "my project"      # Resume by title
```

---

## Procedural Memory (Skills)

The most important memory tier for Hermes's self-improvement loop.

When the agent solves a problem for the first time, it can create a skill document capturing the procedure. On future encounters, it loads the skill and applies the known solution.

### How Skills Are Created

The agent uses `skill_manage` to create, update, and delete its own skills:

```
User: "Help me optimize my PyTorch training loop"
Agent: (solves the problem)
       (creates skill: pytorch-training-optimization)
       "I've saved this as a skill for future use."
```

Next session, `pytorch-training-optimization` appears in the skills index in the system prompt.

### Skill Storage

```
~/.hermes/skills/
└── <category>/
    └── <skill-name>/
        ├── SKILL.md           # Main instructions
        ├── references/        # Linked docs
        ├── templates/         # Output templates
        └── scripts/           # Helper scripts
```

See [Skills](skills.md) for full documentation.

---

## Persistent Key-Value Memory

The `memory` tool provides simple persistent storage that appears in the system prompt at session start:

```python
# Save
memory(action="save", key="user_preferences", value="prefers Python 3.12, uses pytest")

# Read
memory(action="read", key="user_preferences")

# Delete
memory(action="delete", key="user_preferences")
```

**Storage:** `~/.hermes/memories/`

Memories are small facts, preferences, and context that should always be available without taking up conversation space.

---

## Honcho Integration (Cross-Session User Modeling)

[Honcho](https://app.honcho.dev) is an AI-native user memory platform. Hermes optionally uses it to build persistent understanding of users across sessions and devices.

### Setup

```bash
# Install Honcho extras
pip install -e ".[honcho]"

# Configure
hermes honcho setup
```

Set `HONCHO_API_KEY` in `~/.hermes/.env`.

### How Honcho Works

Each conversation turn:
1. User message → Honcho session updated
2. Honcho prefetches relevant user context (optionally injected into system prompt)
3. Agent responds
4. Assistant message → Honcho session updated (per write_frequency)

Over time, Honcho builds a model of the user's preferences, working style, and history.

### Memory Modes

| Mode | Description |
|------|-------------|
| `hybrid` | Auto-inject recent context + Honcho tools available (recommended) |
| `honcho` | Only Honcho tools — explicit retrieval required |
| `context` | Only auto-injected context — no tools |

Configure:
```bash
hermes honcho mode hybrid
```

### Recall Modes

| Mode | Description |
|------|-------------|
| `hybrid` | Auto-inject + tools available |
| `context` | Auto-inject only |
| `tools` | Tools only (no auto-inject) |

### Honcho Tools

When Honcho is enabled, 4 tools become available:

| Tool | Description |
|------|-------------|
| `honcho_profile` | Get user peer card (curated facts, fast) |
| `honcho_search` | Semantic search over user context |
| `honcho_context` | Ask natural language question about user (uses Honcho LLM) |
| `honcho_conclude` | Write a conclusion about the user to Honcho (builds long-term model) |

### Session Strategy

Control how directory/project maps to Honcho sessions:

```yaml
honcho:
  session_strategy: "per-repo"  # per-session | per-repo | per-directory | global

  # Manual overrides
  sessions:
    /home/user/my-project: "main-project"
    /home/user/client-work: "client"
```

### Write Frequency

```yaml
honcho:
  write_frequency: "async"    # async | turn | session | N (int)
```

- `async` — Background thread, batch writes (lowest latency)
- `turn` — Sync per turn (guarantees persistence)
- `session` — Flush on session end
- `N` — Every N turns

### Dialectic Reasoning

Honcho's dialectic feature uses an LLM to reason about the user before responding:

```yaml
honcho:
  dialectic_reasoning_level: "medium"   # minimal | low | medium | high | max
  dialectic_max_chars: 600              # Chars of reasoning to inject
```

Higher levels provide more nuanced user understanding but consume more tokens.

### Management Commands

```bash
hermes honcho status                    # Show current settings
hermes honcho sessions                  # List session mappings
hermes honcho map my-project-session   # Map current dir to session name
hermes honcho peer                      # Configure peer names
hermes honcho tokens --context 800     # Set context token budget
hermes honcho identity SOUL.md         # Seed AI identity
```

---

## Cron Scheduler

The cron system runs agent tasks on a schedule and delivers results to messaging platforms.

### How Cron Works

1. Cron daemon ticks every 60 seconds (file-locked, single writer)
2. Due jobs execute in a separate thread with a fresh AIAgent
3. Output saved to `~/.hermes/cron/output/{job_id}/{timestamp}.md`
4. Result delivered to configured target

### Schedule Formats

| Format | Example | Description |
|--------|---------|-------------|
| Cron expression | `0 9 * * *` | Standard 5-field cron |
| Interval | `every 30m` | Recurring every N duration |
| One-time | `2026-12-25T09:00:00` | ISO 8601 datetime |
| Natural language | `daily 9am` | Parsed to cron |

### Creating Jobs

```bash
# Via CLI
hermes cron add "daily 9am"
# → Interactive prompts for name, prompt, delivery target

# Via tool (agent creates the job)
cronjob(
    action="create",
    name="daily-brief",
    prompt="Give me a morning briefing: today's weather, calendar events, and top 3 priorities",
    schedule="0 9 * * *",
    deliver="telegram",
    skills=["google-workspace"]
)
```

### Delivery Targets

| Target | Description |
|--------|-------------|
| `"origin"` | Back to the chat that created the job |
| `"local"` | Save to `~/.hermes/cron/output/` only |
| `"telegram"` | Telegram home channel |
| `"telegram:123456"` | Specific Telegram chat |
| `"discord"` | Discord home channel |
| `"slack"` | Slack home channel |
| `"email"` | Email home address |

### Managing Jobs

```bash
hermes cron list          # Show all jobs
hermes cron status        # Check if running
hermes cron pause <id>    # Pause
hermes cron resume <id>   # Resume
hermes cron run <id>      # Run now
hermes cron remove <id>   # Delete
```

### Per-Job Model Override

Jobs can use a different (cheaper) model than the default:

```json
{
  "model": "google/gemini-3-flash-preview",
  "provider": "openrouter"
}
```

### Job Storage

```
~/.hermes/cron/jobs.json                            # Job definitions
~/.hermes/cron/output/{job_id}/{timestamp}.md       # Job output
~/.hermes/cron/.tick.lock                           # Prevents concurrent ticks
```

---

## Context Compression

When the conversation gets too long, Hermes automatically compresses old turns to free up context.

### Compression Trigger

- **Default threshold:** 50% of model's context window
- **Emergency:** 70% and 90% triggers more aggressive trimming

### How It Works

1. Token count approaches threshold
2. Auxiliary LLM (default: Gemini 3 Flash) summarizes old turns
3. Summary replaces verbose conversation history
4. Original messages preserved in SQLite (not lost)

### Configuration

```yaml
compression:
  enabled: true
  threshold: 0.50           # Trigger at 50% of context window
  summary_model: "google/gemini-3-flash-preview"
```

Manual trigger:
```
/compress
```

### Session Lineage

When a session is compressed into a new session, the original session is preserved with `parent_session_id` pointing to the new one. This creates a lineage chain allowing full history reconstruction.

---

## Filesystem Checkpoints

Hermes can take periodic snapshots of the filesystem before making changes:

```yaml
checkpoints:
  enabled: true
  max_snapshots: 50
```

Restore:
```
/rollback
```

Shows a list of snapshots with timestamps and changed files. Select one to restore.

**Storage:** `~/.hermes/snapshots/`
