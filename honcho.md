# Honcho Memory

[Honcho](https://honcho.dev) is an AI-native memory system that gives Hermes persistent, cross-session understanding of users. While Hermes has built-in memory (`MEMORY.md` and `USER.md`), Honcho adds a deeper layer of **user modeling** -- learning preferences, goals, communication style, and context across conversations via a dual-peer architecture where both the user and the AI build representations over time.

First integrated in v0.2.0 ([PR #38](https://github.com/NousResearch/hermes-agent/pull/38)). Expanded significantly in v0.3.0 with session isolation, dialectic reasoning, and gateway integration.

## Works Alongside Built-in Memory

Hermes has two memory systems that can work together or be configured separately. In `hybrid` mode (the default), both run side by side -- Honcho adds cross-session user modeling while local files handle agent-level notes.

| Feature | Built-in Memory | Honcho Memory |
|---------|----------------|---------------|
| Storage | Local files (`~/.hermes/memories/`) | Cloud-hosted Honcho API |
| Scope | Agent-level notes and user profile | Deep user modeling via dialectic reasoning |
| Persistence | Across sessions on same machine | Across sessions, machines, and platforms |
| Query | Injected into system prompt automatically | Prefetched + on-demand via tools |
| Content | Manually curated by the agent | Automatically learned from conversations |
| Write surface | `memory` tool (add/replace/remove) | `honcho_conclude` tool (persist facts) |

Set `memoryMode` to `honcho` to use Honcho exclusively.

## Self-hosted / Docker

Hermes supports a local Honcho instance (e.g. via Docker) in addition to the hosted API. Point it at your instance using `HONCHO_BASE_URL` -- no API key required.

```bash
hermes config set HONCHO_BASE_URL http://localhost:8000
```

Or via `~/.honcho/config.json`:

```json
{
  "hosts": {
    "hermes": {
      "base_url": "http://localhost:8000",
      "enabled": true
    }
  }
}
```

Hermes auto-enables Honcho when either `apiKey` or `base_url` is present.

## Setup

### Interactive Setup

```bash
hermes honcho setup
```

The setup wizard walks through API key, peer names, workspace, memory mode, write frequency, recall mode, and session strategy. It offers to install `honcho-ai` if missing.

### Manual Setup

1. Install the client library: `pip install 'honcho-ai>=2.0.1'`
2. Get an API key from [app.honcho.dev](https://app.honcho.dev) > Settings > API Keys
3. Configure `~/.honcho/config.json`:

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

`apiKey` lives at the root because it is a shared credential across all Honcho-enabled tools. All other settings are scoped under `hosts.hermes`.

Or set the API key as an environment variable:

```bash
hermes config set HONCHO_API_KEY your-key
```

When an API key is present (either in `~/.honcho/config.json` or as `HONCHO_API_KEY`), Honcho auto-enables unless explicitly set to `"enabled": false`.

## Configuration

### Global Config (`~/.honcho/config.json`)

Settings are scoped to `hosts.hermes` and fall back to root-level globals when the host field is absent.

**Root-level (shared)**

| Field | Default | Description |
|-------|---------|-------------|
| `apiKey` | -- | Honcho API key (required, shared across all hosts) |
| `sessions` | `{}` | Manual session name overrides per directory (shared) |

**Host-level (`hosts.hermes`)**

| Field | Default | Description |
|-------|---------|-------------|
| `workspace` | `"hermes"` | Workspace identifier |
| `peerName` | *(derived)* | Your identity name for user modeling |
| `aiPeer` | `"hermes"` | AI assistant identity name |
| `environment` | `"production"` | Honcho environment |
| `enabled` | *(auto)* | Auto-enables when API key is present |
| `saveMessages` | `true` | Whether to sync messages to Honcho |
| `memoryMode` | `"hybrid"` | Memory mode: `hybrid` or `honcho` |
| `writeFrequency` | `"async"` | When to write: `async`, `turn`, `session`, or integer N |
| `recallMode` | `"hybrid"` | Retrieval strategy: `hybrid`, `context`, or `tools` |
| `sessionStrategy` | `"per-session"` | How sessions are scoped |
| `sessionPeerPrefix` | `false` | Prefix session names with peer name |
| `contextTokens` | *(Honcho default)* | Max tokens for auto-injected context |
| `dialecticReasoningLevel` | `"low"` | Floor for dialectic reasoning: `minimal` / `low` / `medium` / `high` / `max` |
| `dialecticMaxChars` | `600` | Char cap on dialectic results injected into system prompt |
| `linkedHosts` | `[]` | Other host keys whose workspaces to cross-reference |

### Memory Modes

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

### Recall Modes

Controls how Honcho context reaches the agent:

| Mode | Behavior |
|------|----------|
| `hybrid` | Auto-injected context + Honcho tools available (default) |
| `context` | Auto-injected context only -- Honcho tools hidden |
| `tools` | Honcho tools only -- no auto-injected context |

### Write Frequency

| Setting | Behavior |
|---------|----------|
| `async` | Background thread writes (zero blocking, default) |
| `turn` | Synchronous write after each turn |
| `session` | Batched write at session end |
| *integer N* | Write every N turns |

### Session Strategies

| Strategy | Session key | Use case |
|----------|-------------|----------|
| `per-session` | Unique per run | Default. Fresh session every time. |
| `per-directory` | CWD basename | Each project gets its own session. |
| `per-repo` | Git repo root name | Groups subdirectories under one session. |
| `global` | Fixed `"global"` | Single cross-project session. |

Resolution order: manual map > session title > strategy-derived key > platform key.

## How It Works

### Async Context Pipeline

Honcho context is fetched asynchronously to avoid blocking the response path:

1. User message arrives
2. Consume cached Honcho context from the previous turn
3. Inject user, AI, and dialectic context into the system prompt
4. LLM call
5. Assistant response
6. Start background fetch for Turn N+1 (fetch context + fetch dialectic)
7. Cache for the next turn

Turn 1 is a cold start (no cache). All subsequent turns consume cached results with zero HTTP latency on the response path. The system prompt on turn 1 uses only static context to preserve prefix cache hits at the LLM provider.

### Dual-Peer Architecture

Both the user and AI have peer representations in Honcho:

- **User peer** -- observed from user messages. Honcho learns preferences, goals, communication style.
- **AI peer** -- observed from assistant messages (`observe_me=True`). Honcho builds a representation of the agent's knowledge and behavior.

Both representations are injected into the system prompt when available.

### Dynamic Reasoning Level

Dialectic queries scale reasoning effort with message complexity:

| Message length | Reasoning level |
|----------------|-----------------|
| < 120 chars | Config default (typically `low`) |
| 120-400 chars | One level above default (cap: `high`) |
| > 400 chars | Two levels above default (cap: `high`) |

`max` is never selected automatically.

### Gateway Integration

The gateway creates short-lived `AIAgent` instances per request. Honcho managers are owned at the gateway session layer (`_honcho_managers` dict) so they persist across requests within the same session and flush at real session boundaries (reset, resume, expiry, server stop).

Each gateway session (e.g., a Telegram chat, a Discord channel) gets its own Honcho session context. The session key -- derived from the platform and chat ID -- is threaded through the entire tool dispatch chain so that Honcho tool calls always execute against the correct session, even when multiple users are messaging concurrently.

| Event | What happens to Honcho |
|-------|------------------------|
| New message arrives | Agent inherits the gateway's Honcho manager + session key |
| `/reset` | Memory flush fires with the old session key, then Honcho manager shuts down |
| `/resume` | Current session is flushed, then the resumed session's Honcho context loads |
| Session expiry | Automatic flush + shutdown after the configured idle timeout |
| Gateway stop | All active Honcho managers are flushed and shut down gracefully |

## Tools

When Honcho is active, four tools become available. Availability is gated dynamically -- they are invisible when Honcho is disabled.

### `honcho_profile`

Fast peer card retrieval (no LLM). Returns a curated list of key facts about the user.

### `honcho_search`

Semantic search over memory (no LLM). Returns raw excerpts ranked by relevance. Cheaper and faster than `honcho_context` -- good for factual lookups.

Parameters:
- `query` (string) -- search query
- `max_tokens` (integer, optional) -- result token budget

### `honcho_context`

Dialectic Q&A powered by Honcho's LLM. Synthesizes an answer from accumulated conversation history. Higher cost than `honcho_profile` or `honcho_search`.

Parameters:
- `query` (string) -- natural language question
- `peer` (string, optional) -- `"user"` (default) or `"ai"`. Querying `"ai"` asks about the assistant's own history and identity.

### `honcho_conclude`

Writes a fact to Honcho memory. Use when the user explicitly states a preference, correction, or project context worth remembering. Feeds into the user's peer card and representation.

Parameters:
- `conclusion` (string) -- the fact to persist

## CLI Commands

```
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
hermes honcho identity <file>              # Seed AI peer identity from file
hermes honcho migrate                      # Migration guide: OpenClaw -> Hermes + Honcho
```

`hermes doctor` also includes a Honcho section that validates config, API key, and connection status.

## AI Peer Identity

Honcho can build a representation of the AI assistant over time (via `observe_me=True`). You can also seed the AI peer explicitly:

```bash
hermes honcho identity ~/.hermes/SOUL.md
```

This uploads the file content through Honcho's observation pipeline. The AI peer representation is then injected into the system prompt alongside the user's.

## Implementation

The Honcho integration lives in `honcho_integration/` with four modules:

- `__init__.py` -- package exports
- `client.py` -- Honcho API client wrapper
- `session.py` -- session management and lifecycle
- `cli.py` -- CLI command implementations

Honcho is fully opt-in -- zero behavior change when disabled or unconfigured. All Honcho calls are non-fatal; if the service is unreachable, the agent continues normally.
