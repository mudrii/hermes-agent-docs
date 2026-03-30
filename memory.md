# Memory and Persistence

Hermes Agent has two memory systems that can work independently or together. The built-in memory system uses local files and SQLite for persistent facts and session history. The optional Honcho integration adds a cloud-backed AI-native user modeling layer. This document covers both systems in full.

---

## Built-in Memory: Overview

Hermes maintains two local memory files that persist across sessions:

| File | Purpose | Character Limit |
|------|---------|----------------|
| **MEMORY.md** | Agent's personal notes -- environment facts, conventions, lessons learned | 2,200 chars (~800 tokens) |
| **USER.md** | User profile -- preferences, communication style, expectations | 1,375 chars (~500 tokens) |

Both files are stored in `~/.hermes/memories/` and are injected into the system prompt as a frozen snapshot at session start. The agent manages its own memory via the `memory` tool.

The character limits are defined in `tools/memory_tool.py` as defaults to `MemoryStore.__init__()`:

```python
def __init__(self, memory_char_limit: int = 2200, user_char_limit: int = 1375):
```

The entry delimiter between entries is `\n§\n` (section sign on its own line).

---

## How Memory Appears in the System Prompt

At session start, memory entries are loaded from disk and rendered into the system prompt as a frozen block:

```
══════════════════════════════════════════════
MEMORY (your personal notes) [67% -- 1,474/2,200 chars]
══════════════════════════════════════════════
User's project is a Rust web service at ~/code/myapi using Axum + SQLx
§
This machine runs Ubuntu 22.04, has Docker and Podman installed
§
User prefers concise responses, dislikes verbose explanations
```

The format includes:
- A separator line of `═` characters (46 characters)
- A header with the store name (`MEMORY` or `USER PROFILE`) and usage stats
- Another separator line
- Individual entries separated by `§` (section sign) delimiters on their own lines
- Entries can span multiple lines

### Frozen Snapshot Pattern

The system prompt injection is captured once at session start via `load_from_disk()` and stored in `_system_prompt_snapshot`. This snapshot never changes mid-session. This is intentional -- it preserves the LLM's prefix cache for performance.

When the agent adds, removes, or replaces memory entries during a session, the changes are persisted to disk immediately (under file lock, with atomic rename via `os.replace()`), but they do not appear in the system prompt until the next session starts. Tool call responses always reflect the live state via `_entries_for()`.

---

## Memory Tool Actions

The agent uses the `memory` tool with these actions:

| Action | Description |
|--------|-------------|
| `add` | Add a new memory entry |
| `replace` | Replace an existing entry using substring matching |
| `remove` | Remove an entry using substring matching |

There is no `read` action -- memory content is automatically injected into the system prompt. The agent sees its memories as part of its conversation context at the start of each session.

### Substring Matching

The `replace` and `remove` actions use short unique substring matching. The `old_text` parameter just needs to be a unique substring that identifies exactly one entry:

```python
# If memory contains "User prefers dark mode in all editors"
memory(action="replace", target="memory",
       old_text="dark mode",
       content="User prefers light mode in VS Code, dark mode in terminal")
```

If the substring matches multiple entries with different content, an error is returned asking for a more specific match. If all matching entries are identical (exact duplicates), the operation applies to the first one.

---

## Two Memory Targets

### `memory` -- Agent's Personal Notes

For information the agent needs to remember about the environment, workflows, and lessons learned:

- Environment facts (OS, installed tools, project structure)
- Project conventions and configuration
- Tool quirks and workarounds discovered
- Skills and techniques that worked

### `user` -- User Profile

For information about the user's identity, preferences, and communication style:

- Name, role, timezone
- Communication preferences (concise vs. detailed, format preferences)
- Pet peeves and things to avoid
- Workflow habits
- Technical skill level

---

## What to Save vs. Skip

The agent saves information proactively -- you do not need to ask. It saves when it learns:

| Category | Examples | Target |
|----------|---------|--------|
| User preferences | "I prefer TypeScript over JavaScript" | `user` |
| Environment facts | "This server runs Debian 12 with PostgreSQL 16" | `memory` |
| Corrections | "Don't use `sudo` for Docker commands, user is in docker group" | `memory` |
| Conventions | "Project uses tabs, 120-char line width, Google-style docstrings" | `memory` |
| Explicit requests | "Remember that my API key rotation happens monthly" | `memory` |

Skip:
- Trivial or obvious information: "User asked about Python" (too vague)
- Easily re-discovered facts: "Python 3.12 supports f-string nesting" (can search this)
- Raw data dumps: large code blocks, log files, data tables (too big)
- Session-specific ephemera: temporary file paths, one-off debugging context
- Information already in context files: `SOUL.md` and `AGENTS.md` content
- Task progress, session outcomes, completed-work logs, or temporary TODO state -- use `session_search` for those
- New procedures or solutions that could be useful later -- save those as skills with the skill tool

---

## Capacity Management

| Store | Limit | Typical entries |
|-------|-------|----------------|
| memory | 2,200 chars | 8-15 entries |
| user | 1,375 chars | 5-10 entries |

When an addition would exceed the limit, the tool returns an error:

```json
{
  "success": false,
  "error": "Memory at 2,100/2,200 chars. Adding this entry (250 chars) would exceed the limit. Replace or remove existing entries first.",
  "current_entries": ["..."],
  "usage": "2,100/2,200"
}
```

The agent should read current entries from the error response, identify entries to consolidate or remove, use `replace` to merge related entries, then add the new entry.

Best practice: when memory is above 80% capacity (visible in the system prompt header), consolidate entries proactively.

### Duplicate Prevention

The memory system automatically rejects exact duplicate entries. If you try to add content that already exists, it returns success with a "no duplicate added" message.

### File Locking and Atomic Writes

Memory operations use a two-layer safety mechanism:

1. **File locking** (`fcntl.flock`) via a separate `.lock` file for read-modify-write safety across concurrent sessions
2. **Atomic writes** via `tempfile.mkstemp()` + `os.replace()` so readers always see either the complete old file or the complete new file, never a partial write

The `_reload_target()` method re-reads from disk under the lock before every mutation to pick up writes from other concurrent sessions.

---

## Security Scanning

Memory entries are scanned for injection and exfiltration patterns before being accepted, because they are injected into the system prompt. The `_scan_memory_content()` function in `tools/memory_tool.py` checks for:

- Prompt injection patterns ("ignore previous instructions", "you are now", "system prompt override")
- Credential exfiltration via curl/wget with secret variable references
- Attempts to read secret files (.env, credentials, .netrc, .pgpass, .npmrc, .pypirc)
- SSH backdoor attempts (authorized_keys, .ssh access)
- Hermes env file access patterns
- Invisible Unicode characters (zero-width spaces, bidirectional overrides, word joiners, byte order marks)

Content matching any pattern is blocked with an error message identifying the threat pattern.

---

## Session Search

Beyond `MEMORY.md` and `USER.md`, the agent can search past conversations using the `session_search` tool:

- All CLI and messaging sessions are stored in SQLite at `~/.hermes/state.db` with FTS5 full-text search indexing
- Search queries return relevant past conversations with summarization
- The agent can find things discussed weeks ago, even if they are not in the active memory files

```bash
hermes sessions list    # Browse past sessions
```

### Recent Sessions Mode (v0.5.0 — PR #2533)

Calling `session_search` with **no query** returns recent sessions instead of performing a keyword search. Each result includes the session title, a short preview, and a timestamp. This gives the agent a compact overview of recent work without requiring a specific search term.

```python
session_search()              # Returns recent sessions with titles, previews, timestamps
session_search(query="auth")  # Full-text search across all sessions
```

### Session Search Fallback Preview (v0.5.0 — PR #3478)

When LLM-based summarization fails (e.g., model unavailable), `session_search` falls back to a plain-text excerpt from the raw transcript instead of returning an empty result. The preview is truncated at a safe token limit.

### Memory vs. Session Search

| Feature | Persistent Memory | Session Search |
|---------|------------------|----------------|
| Capacity | ~1,300 tokens total | Unlimited (all sessions) |
| Speed | Instant (already in system prompt) | Requires search and LLM summarization |
| Use case | Key facts always available | Finding specific past conversations |
| Management | Manually curated by agent | Automatic -- all sessions stored |
| Token cost | Fixed per session (~1,300 tokens) | On-demand when searched |

Memory is for critical facts that should always be in context. Session search is for "did we discuss X last week?" queries.

---

## Session Lifecycle Commands (v0.5.0)

### `/resume` Command (PR #3315)

The `/resume [name]` slash command resumes a previously-named session. In v0.5.0 the CLI handler was added alongside a `reopen_session` API on the `SessionDB` class. This allows the session transcript to be reopened and continued, flushing the current session context and loading the target session's history.

```
/resume my-project        # Resume the session titled "my-project"
/resume                   # Open session picker
```

When a session is resumed:
1. The current session's memories are flushed.
2. The resumed session's conversation history is loaded.
3. The session ID and title are restored.
4. Honcho context for the resumed session loads (if Honcho is enabled).

### Session Config Surfacing on `/new`, `/reset`, Auto-Reset (PR #3321)

In v0.5.0, when a new session starts — via `/new`, `/reset`, or automatic idle/daily reset — Hermes displays the active session configuration to the user. This includes the current model, provider, toolsets, and key settings, so users can confirm the session started with the expected configuration.

---

## Configuration

Control built-in memory in `~/.hermes/config.yaml`:

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200   # ~800 tokens
  user_char_limit: 1375     # ~500 tokens
```

Setting `memory_enabled: false` disables `MEMORY.md` injection. Setting `user_profile_enabled: false` disables `USER.md` injection. These are independent toggles.

---

## How to Disable Memory

Set both flags to false in `~/.hermes/config.yaml`:

```yaml
memory:
  memory_enabled: false
  user_profile_enabled: false
```

Or via CLI:

```bash
hermes config set memory.memory_enabled false
hermes config set memory.user_profile_enabled false
```

The memory files remain on disk. Re-enabling memory loads them again in the next session.

---

## Honcho Integration: Cross-Session User Modeling

Honcho is an AI-native memory system that gives Hermes persistent, cross-session understanding of users. While Hermes has built-in memory (`MEMORY.md` and `USER.md`), Honcho adds a deeper layer of user modeling -- learning preferences, goals, and communication style across conversations via a dual-peer architecture where both the user and the AI build representations over time.

### Built-in Memory vs. Honcho Memory

| Feature | Built-in Memory | Honcho Memory |
|---------|----------------|---------------|
| Storage | Local files (`~/.hermes/memories/`) | Cloud-hosted Honcho API |
| Scope | Agent-level notes and user profile | Deep user modeling via dialectic reasoning |
| Persistence | Across sessions on same machine | Across sessions, machines, and platforms |
| Query | Injected into system prompt automatically | Prefetched plus on-demand via tools |
| Content | Manually curated by the agent | Automatically learned from conversations |
| Write surface | `memory` tool (add/replace/remove) | `honcho_conclude` tool (persist facts) |

In `hybrid` mode (the default when Honcho is enabled), both run side by side.

---

## Honcho Setup

### Interactive Setup

```bash
hermes honcho setup
```

The wizard walks through API key, peer names, workspace, memory mode, write frequency, recall mode, and session strategy. It offers to install `honcho-ai` if missing.

### Manual Setup

Install the client library:

```bash
pip install 'honcho-ai>=2.0.1'
```

Or install with the extra:

```bash
uv pip install -e '.[honcho]'
```

Get an API key from [app.honcho.dev](https://app.honcho.dev) under Settings > API Keys.

Configure in `~/.honcho/config.json` (shared across all Honcho-enabled applications):

```json
{
  "apiKey": "your-honcho-api-key",
  "hosts": {
    "hermes": {
      "workspace": "hermes",
      "peerName": "your-name",
      "aiPeer": "hermes",
      "memoryMode": "hybrid",
      "writeFrequency": "async",
      "recallMode": "hybrid",
      "sessionStrategy": "per-session",
      "enabled": true
    }
  }
}
```

Or set the API key via environment variable:

```bash
hermes config set HONCHO_API_KEY your-key
```

When an API key is present (either in `~/.honcho/config.json` or as `HONCHO_API_KEY`), Honcho auto-enables unless explicitly set to `"enabled": false`.

The global config path is `~/.honcho/config.json` (defined as `GLOBAL_CONFIG_PATH` in `honcho_integration/client.py`). The host key for Hermes is `"hermes"` (defined as `HOST = "hermes"`).

---

## Honcho Configuration Reference

### Global Config (`~/.honcho/config.json`)

Settings are scoped to `hosts.hermes`. Root-level keys are shared across all Honcho-enabled tools. Hermes only writes to its own host block (except `apiKey`, which is a shared credential at the root).

**Resolution order:** host block field > root-level field > default.

**Root-level (shared)**

| Field | Default | Description |
|-------|---------|-------------|
| `apiKey` | -- | Honcho API key (required, shared across all hosts) |
| `sessions` | `{}` | Manual session name overrides per directory |

**Host-level (`hosts.hermes`)**

| Field | Default | Description |
|-------|---------|-------------|
| `workspace` | `"hermes"` | Workspace identifier |
| `peerName` | *(derived from system)* | Your identity name for user modeling |
| `aiPeer` | `"hermes"` | AI assistant identity name |
| `environment` | `"production"` | Honcho environment |
| `enabled` | *(auto when API key present)* | Enable/disable Honcho |
| `saveMessages` | `true` | Whether to sync messages to Honcho |
| `memoryMode` | `"hybrid"` | Memory mode: `hybrid` or `honcho` |
| `writeFrequency` | `"async"` | When to write: `async`, `turn`, `session`, or integer N |
| `recallMode` | `"hybrid"` | Retrieval strategy: `hybrid`, `context`, or `tools` |
| `sessionStrategy` | `"per-session"` | How sessions are scoped |
| `sessionPeerPrefix` | `false` | Prefix session names with peer name |
| `contextTokens` | *(Honcho default)* | Max tokens for auto-injected context |
| `dialecticReasoningLevel` | `"low"` | Floor for dialectic reasoning: `minimal`, `low`, `medium`, `high`, `max` |
| `dialecticMaxChars` | `600` | Character cap on dialectic results injected into system prompt |
| `linkedHosts` | `[]` | Other host keys whose workspaces to cross-reference |
| `baseUrl` | *(none)* | Override for self-hosted Honcho deployments (also configurable via `honcho.base_url` in `~/.hermes/config.yaml`) |

---

## Memory Modes

| Mode | Effect |
|------|--------|
| `hybrid` | Write to both Honcho and local files (default) |
| `honcho` | Honcho only -- skip local file writes |

Memory mode can be set globally or per-peer:

```json
{
  "memoryMode": {
    "default": "hybrid",
    "hermes": "honcho"
  }
}
```

The `_resolve_memory_mode()` function in `honcho_integration/client.py` handles both string and object forms. The object form uses `"default"` as the fallback and other keys as per-peer overrides.

To disable Honcho entirely, set `enabled: false` or remove the API key.

---

## Recall Modes

Controls how Honcho context reaches the agent:

| Mode | Behavior |
|------|----------|
| `hybrid` | Auto-injected context plus Honcho tools available (default) |
| `context` | Auto-injected context only -- Honcho tools hidden |
| `tools` | Honcho tools only -- no auto-injected context |

Legacy alias: `"auto"` is normalized to `"hybrid"` by `_normalize_recall_mode()`.

---

## Write Frequency

| Setting | Behavior |
|---------|----------|
| `async` | Background thread writes (zero blocking, default) |
| `turn` | Synchronous write after each conversation turn |
| `session` | Batched write at session end |
| *integer N* | Write every N turns |

The default `async` mode has zero blocking impact on response latency. The background writer thread runs independently and flushes automatically at session end.

---

## Session Strategies

| Strategy | Session key | Use case |
|----------|-------------|----------|
| `per-session` | Unique per run (Hermes session_id) | Default -- fresh session every time |
| `per-directory` | CWD basename | Each project gets its own session |
| `per-repo` | Git repo root name | Groups subdirectories under one session |
| `global` | Workspace name | Single cross-project session |

Resolution order (from `resolve_session_name()`): manual directory map > session title > strategy-derived key > platform key.

---

## How Async Context Works

Honcho context is fetched asynchronously to avoid blocking the response path:

```
User message
  -> Consume cached Honcho context from the previous turn
  -> Inject user, AI, and dialectic context into the system prompt
  -> LLM call
  -> Assistant response
  -> Start background fetch for Turn N+1:
       - Fetch context
       - Fetch dialectic
       - Cache for the next turn
```

Turn 1 is a cold start (no cache). All subsequent turns consume cached results with zero HTTP latency on the response path. The system prompt on Turn 1 uses only static context to preserve prefix cache hits at the LLM provider.

---

## Dual-Peer Architecture

Both the user and the AI have peer representations in Honcho:

- **User peer**: observed from user messages. Honcho learns preferences, goals, communication style.
- **AI peer**: observed from assistant messages (`observe_me=True`). Honcho builds a representation of the agent's knowledge and behavior.

Both representations are injected into the system prompt when available.

### Dynamic Reasoning Level

Dialectic queries scale reasoning effort with message complexity:

| Message length | Reasoning level |
|----------------|-----------------|
| Less than 120 chars | Config default (typically `low`) |
| 120-400 chars | One level above default (cap: `high`) |
| More than 400 chars | Two levels above default (cap: `high`) |

`max` is never selected automatically.

---

## Honcho Tools

When Honcho is active, four tools become available to the agent (registered in `tools/honcho_tools.py`). They are invisible when Honcho is disabled (the `_check_honcho_available()` function returns `False`).

### `honcho_profile`

Fast peer card retrieval (no LLM call). Returns a curated list of key facts about the user. Good at conversation start or for a quick factual snapshot.

### `honcho_search`

Semantic search over memory (no LLM call). Returns raw excerpts ranked by relevance. Cheaper and faster than `honcho_context` -- good for factual lookups.

Parameters:
- `query` (string, required) -- search query
- `max_tokens` (integer, optional) -- result token budget (default 800, max 2000)

### `honcho_context`

Dialectic Q&A powered by Honcho's LLM. Synthesizes an answer from accumulated conversation history. Higher cost than profile or search.

Parameters:
- `query` (string, required) -- natural language question
- `peer` (string, optional) -- `"user"` (default) or `"ai"` (queries the assistant's own history and identity)

### `honcho_conclude`

Writes a factual statement to Honcho memory. Use when the user explicitly states a preference, correction, or project context worth remembering. Feeds into the user's peer card and representation.

Parameters:
- `conclusion` (string, required) -- the fact to persist

---

## Multi-User Isolation in Gateway Mode

Each gateway session (e.g., a Telegram chat, a Discord channel) gets its own Honcho session context. The session key -- derived from the platform and chat ID -- is threaded through the entire tool dispatch chain via `set_session_context()` so that Honcho tool calls always execute against the correct session, even when multiple users are messaging concurrently.

This means:
- `honcho_profile`, `honcho_search`, `honcho_context`, and `honcho_conclude` all resolve the correct session at call time via `_resolve_session_context()`, not at startup
- Background memory flushes (triggered by `/reset`, `/resume`, or session expiry) preserve the original session key so they write to the correct Honcho session
- Synthetic flush turns skip Honcho sync to avoid polluting conversation history with internal bookkeeping

### Gateway Session Lifecycle

| Event | What happens to Honcho |
|-------|------------------------|
| New message arrives | Agent inherits the gateway's Honcho manager and session key |
| `/reset` | Memory flush fires with the old session key, then Honcho manager shuts down |
| `/resume` | Current session is flushed, then the resumed session's Honcho context loads (v0.5.0 — PR #3315) |
| Session expiry | Automatic flush and shutdown after the configured idle timeout |
| Gateway stop | All active Honcho managers are flushed and shut down gracefully |

**`reopen_session` API (v0.5.0 — PR #3315):** The `SessionDB` class exposes a `reopen_session(session_id)` method that re-attaches the database to an existing session, loading its transcript and metadata. This is the internal mechanism used by `/resume` and by gateway session restoration after a restart.

Honcho managers are owned at the gateway session layer so they persist across requests within the same session and flush at real session boundaries.

---

## Honcho CLI Commands

```bash
hermes honcho setup                        # Interactive setup wizard
hermes honcho status                       # Show config and connection status
hermes honcho sessions                     # List directory -> session name mappings
hermes honcho map <name>                   # Map current directory to a session name
hermes honcho peer                         # Show peer names and dialectic settings
hermes honcho peer --user NAME             # Set user peer name
hermes honcho peer --ai NAME               # Set AI peer name
hermes honcho peer --reasoning LEVEL       # Set dialectic reasoning level
hermes honcho mode                         # Show current memory mode
hermes honcho mode [hybrid|honcho|local]   # Set memory mode
hermes honcho tokens                       # Show token budget settings
hermes honcho tokens --context N           # Set context token cap
hermes honcho tokens --dialectic N         # Set dialectic char cap
hermes honcho identity                     # Show AI peer identity
hermes honcho identity <file>              # Seed AI peer identity from file (SOUL.md, etc.)
hermes honcho migrate                      # Migration guide: OpenClaw -> Hermes + Honcho
```

`hermes doctor` includes a Honcho section that validates config, API key, and connection status, showing workspace, mode, and write frequency.

---

## Multi-Host Configuration

Multiple Honcho-enabled tools share `~/.honcho/config.json`. Each tool writes only to its own host block and reads its host block first, falling back to root-level globals:

```json
{
  "apiKey": "your-key",
  "peerName": "eri",
  "hosts": {
    "hermes": {
      "workspace": "my-workspace",
      "aiPeer": "hermes-assistant",
      "memoryMode": "honcho",
      "linkedHosts": ["claude-code"],
      "contextTokens": 2000,
      "dialecticReasoningLevel": "medium"
    },
    "claude-code": {
      "workspace": "my-workspace",
      "aiPeer": "clawd"
    }
  }
}
```

Resolution: `hosts.<tool>` field > root-level field > default. Both tools share the root `apiKey` and `peerName`, but each has its own `aiPeer` and can have separate workspace settings.

The `get_linked_workspaces()` method resolves linked host keys to workspace names for cross-tool context sharing.

---

## AI Peer Identity

Honcho builds a representation of the AI assistant over time via `observe_me=True`. You can also seed the AI peer explicitly:

```bash
hermes honcho identity ~/.hermes/SOUL.md
```

This uploads the file content through Honcho's observation pipeline. The AI peer representation is then injected into the system prompt alongside the user's, giving the agent awareness of its own accumulated identity.

```bash
hermes honcho identity --show    # Show the current AI peer representation from Honcho
```

---

## Migration

### From Local Memory to Honcho

When Honcho activates on an instance with existing local history, migration runs automatically:

1. Prior conversation messages are uploaded as an XML transcript file
2. Existing `MEMORY.md`, `USER.md`, and `SOUL.md` are uploaded for context

### From OpenClaw to Hermes + Honcho

```bash
hermes honcho migrate
```

This walks through converting an OpenClaw native Honcho setup to the shared `~/.honcho/config.json` format.

---

## Honcho Reliability

Honcho is fully opt-in -- zero behavior change when disabled or unconfigured. All Honcho calls are non-fatal: if the service is unreachable, the agent continues normally using only local memory and session history.
