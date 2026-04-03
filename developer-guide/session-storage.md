# Session Storage

Hermes Agent uses a SQLite database (`~/.hermes/state.db`) to persist session metadata, full message history, and model configuration across CLI and gateway sessions.

Source file: `hermes_state.py`

## Architecture Overview

```
~/.hermes/state.db (SQLite, WAL mode)
+-- sessions          Session metadata, token counts, billing
+-- messages          Full message history per session
+-- messages_fts      FTS5 virtual table for full-text search
+-- schema_version    Single-row table tracking migration state
```

Key design decisions:
- **WAL mode** for concurrent readers + one writer (gateway multi-platform)
- **FTS5 virtual table** for fast text search across all session messages
- **Session lineage** via `parent_session_id` chains (compression-triggered splits)
- **Source tagging** (`cli`, `telegram`, `discord`, etc.) for platform filtering
- Batch runner and RL trajectories are NOT stored here (separate systems)

## Schema (v6)

The `SessionDB` maintains schema version 6. Migrations run automatically on startup, stepping through each version sequentially.

### Sessions Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | TEXT PRIMARY KEY | Session UUID |
| `source` | TEXT NOT NULL | Platform origin: `cli`, `telegram`, `discord`, `slack`, etc. |
| `user_id` | TEXT | Platform user identifier |
| `model` | TEXT | Active model slug |
| `model_config` | TEXT | JSON serialized model configuration |
| `system_prompt` | TEXT | Snapshot of the assembled system prompt |
| `parent_session_id` | TEXT | Foreign key to parent session after compression split |
| `started_at` | REAL NOT NULL | Unix timestamp of session creation |
| `ended_at` | REAL | Unix timestamp when session ended |
| `end_reason` | TEXT | Why the session ended |
| `message_count` | INTEGER DEFAULT 0 | Total message count |
| `tool_call_count` | INTEGER DEFAULT 0 | Total tool call count |
| `input_tokens` | INTEGER DEFAULT 0 | Cumulative input tokens |
| `output_tokens` | INTEGER DEFAULT 0 | Cumulative output tokens |
| `cache_read_tokens` | INTEGER DEFAULT 0 | Cumulative cache-read tokens |
| `cache_write_tokens` | INTEGER DEFAULT 0 | Cumulative cache-write tokens |
| `reasoning_tokens` | INTEGER DEFAULT 0 | Cumulative reasoning tokens |
| `billing_provider` | TEXT | Provider used for billing |
| `billing_base_url` | TEXT | Endpoint used for billing |
| `billing_mode` | TEXT | Billing mode |
| `estimated_cost_usd` | REAL | Estimated cost in USD |
| `actual_cost_usd` | REAL | Actual cost in USD from provider |
| `cost_status` | TEXT | `actual`, `estimated`, `included`, `unknown` |
| `cost_source` | TEXT | Source of pricing data |
| `pricing_version` | TEXT | Pricing snapshot version identifier |
| `title` | TEXT | Human-readable session title (unique when non-NULL, max 100 chars) |

### Messages Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Row ID |
| `session_id` | TEXT NOT NULL | Foreign key to sessions |
| `role` | TEXT NOT NULL | `user`, `assistant`, `tool` |
| `content` | TEXT | Message content |
| `tool_call_id` | TEXT | Tool call identifier for tool results |
| `tool_calls` | TEXT | JSON serialized tool call list |
| `tool_name` | TEXT | Tool name for tool result messages |
| `timestamp` | REAL NOT NULL | Unix timestamp |
| `token_count` | INTEGER | Per-message token estimate |
| `finish_reason` | TEXT | Model finish reason |
| `reasoning` | TEXT | Raw reasoning text extracted from `<think>` blocks or native reasoning |
| `reasoning_details` | TEXT | JSON serialized structured reasoning detail objects |
| `codex_reasoning_items` | TEXT | JSON serialized Codex reasoning items for Responses API sessions |

### FTS5 Full-Text Search

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);
```

The FTS5 table is kept in sync via triggers that fire on INSERT, UPDATE, and DELETE of the `messages` table.

### Indexes

```sql
idx_sessions_source        ON sessions(source)
idx_sessions_parent        ON sessions(parent_session_id)
idx_sessions_started       ON sessions(started_at DESC)
idx_sessions_title_unique  ON sessions(title) WHERE title IS NOT NULL
idx_messages_session       ON messages(session_id, timestamp)
```

## Schema Migrations

| Version | Changes |
|---------|---------|
| v1 | Initial schema (sessions, messages, FTS5) |
| v2 | Added `finish_reason` column to `messages` |
| v3 | Added `title` column to `sessions` |
| v4 | Added unique index on `title` (NULLs allowed) |
| v5 | Added billing columns (`cache_read_tokens`, `cache_write_tokens`, `reasoning_tokens`, `billing_provider`, `billing_base_url`, `billing_mode`) and cost columns (`estimated_cost_usd`, `actual_cost_usd`, `cost_status`, `cost_source`, `pricing_version`) |
| v6 | Added `reasoning`, `reasoning_details`, `codex_reasoning_items` columns to `messages` to persist reasoning across gateway turns |

Each migration uses `ALTER TABLE ADD COLUMN` wrapped in try/except to handle the column-already-exists case (idempotent).

## Write Contention Handling

Multiple hermes processes (gateway + CLI sessions + worktree agents) share one `state.db`. The `SessionDB` class handles write contention with:

- **Short SQLite timeout** (1 second) instead of the default 30s
- **Application-level retry** with random jitter (20-150ms, up to 15 retries)
- **BEGIN IMMEDIATE** transactions to surface lock contention at transaction start
- **Periodic WAL checkpoints** every 50 successful writes (PASSIVE mode)

```
_WRITE_MAX_RETRIES = 15
_WRITE_RETRY_MIN_S = 0.020   # 20ms
_WRITE_RETRY_MAX_S = 0.150   # 150ms
_CHECKPOINT_EVERY_N_WRITES = 50
```

## Common Operations

### Initialize

```python
from hermes_state import SessionDB

db = SessionDB()                              # Default: ~/.hermes/state.db
db = SessionDB(db_path=Path("/tmp/test.db"))  # Custom path
```

### Create and Manage Sessions

```python
db.create_session(
    session_id="sess_abc123",
    source="cli",
    model="anthropic/claude-sonnet-4.6",
    user_id="user_1",
    parent_session_id=None,
)

db.end_session("sess_abc123", end_reason="user_exit")
db.reopen_session("sess_abc123")
```

### Store Messages

```python
msg_id = db.append_message(
    session_id="sess_abc123",
    role="assistant",
    content="Here's the answer...",
    tool_calls=[{"id": "call_1", "function": {"name": "terminal", "arguments": "{}"}}],
    token_count=150,
    finish_reason="stop",
    reasoning="Let me think about this...",
)
```

### Retrieve Messages

```python
messages = db.get_messages("sess_abc123")
conversation = db.get_messages_as_conversation("sess_abc123")
```

## Full-Text Search

The `search_messages()` method supports FTS5 query syntax with automatic sanitization of user input.

### FTS5 Query Syntax

| Syntax | Example | Meaning |
|--------|---------|---------|
| Keywords | `docker deployment` | Both terms (implicit AND) |
| Quoted phrase | `"exact phrase"` | Exact phrase match |
| Boolean OR | `docker OR kubernetes` | Either term |
| Boolean NOT | `python NOT java` | Exclude term |
| Prefix | `deploy*` | Prefix match |

### Query Sanitization

`_sanitize_fts5_query()` handles edge cases:
- Strips unmatched quotes and special characters
- Wraps hyphenated terms in quotes (`chat-send` -> `"chat-send"`)
- Removes dangling boolean operators (`hello AND` -> `hello`)

Results include a snippet (40 tokens, delimited by `>>>` and `<<<`), surrounding context (1 message before and after), and session metadata.

## Session Titles and Lineage

- `MAX_TITLE_LENGTH = 100` characters
- `sanitize_title()` strips control characters, invisible Unicode, collapses whitespace
- `set_session_title()` validates uniqueness across sessions
- `resolve_session_by_title()` follows numbered lineage variants (e.g. `"my session"` -> `"my session #2"`)
- Auto-titling: after the first user-assistant exchange, `maybe_auto_title()` fires a background thread that calls `call_llm(task="compression", max_tokens=30, temperature=0.3)` to generate a 3-7 word title

## Session Lineage

Sessions form chains via `parent_session_id`. This happens when context compression triggers a session split in the gateway.

## Schema v6: Reasoning Persistence

Schema v6 was introduced in v0.5.0 ([#2974](https://github.com/NousResearch/hermes-agent/pull/2974)) to persist reasoning data across gateway session turns.

### New Columns

| Column | Type | Purpose |
|--------|------|---------|
| `reasoning` | TEXT | Raw reasoning text extracted from `<think>` blocks or native reasoning fields |
| `reasoning_details` | TEXT | JSON serialized structured reasoning detail objects (e.g. Anthropic's `thinking` content blocks) |
| `codex_reasoning_items` | TEXT | JSON serialized Codex reasoning items for Responses API sessions |

### Why These Columns Exist

Providers that replay reasoning across turns (OpenRouter, OpenAI Responses API, Nous Portal) need the reasoning content to be available when the conversation is restored. Prior to v6, reasoning was only available during the current turn's streaming callbacks and was discarded before persistence. The gateway's session restore path now reads these columns back, allowing multi-turn sessions to retain and replay reasoning context across connection resets.

### Write Contention Details

Multiple hermes processes (gateway + CLI sessions + worktree agents) share one `state.db`. The `SessionDB` class handles write contention with:

- **BEGIN IMMEDIATE** transactions to surface lock contention at transaction start rather than at commit time
- **Application-level retry** with random jitter (20-150ms, up to 15 retries)
- **Short SQLite timeout** (1 second) instead of the default 30s, so retries happen quickly
- **PASSIVE WAL checkpoint** every 50 successful writes (`_CHECKPOINT_EVERY_N_WRITES = 50`), keeping the WAL file from growing unbounded

```
_WRITE_MAX_RETRIES = 15
_WRITE_RETRY_MIN_S = 0.020   # 20ms
_WRITE_RETRY_MAX_S = 0.150   # 150ms
_CHECKPOINT_EVERY_N_WRITES = 50
```

## Database Location

Default path: `~/.hermes/state.db`

Derived from `hermes_constants.get_hermes_home()` which resolves to `~/.hermes/` by default, or the value of `HERMES_HOME` environment variable.
