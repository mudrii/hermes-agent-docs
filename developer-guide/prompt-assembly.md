# Prompt Assembly

Hermes deliberately separates the **cached system prompt** (stable across turns) from **ephemeral API-call-time additions** (not persisted). This is one of the most important design choices in the project because it affects token usage, prompt caching effectiveness, session continuity, and memory correctness.

Primary files: `run_agent.py`, `agent/prompt_builder.py`, `tools/memory_tool.py`

## Cached System Prompt Layers (in order)

1. **Agent identity** -- `SOUL.md` loaded from `HERMES_HOME` via `load_soul_md()` when available; falls back to `DEFAULT_AGENT_IDENTITY` in `prompt_builder.py`
2. **Tool-aware behavior guidance** -- `MEMORY_GUIDANCE`, `SESSION_SEARCH_GUIDANCE`, `SKILLS_GUIDANCE` constants
3. **Honcho static block** -- when Honcho integration is active
4. **Optional system message** -- user-provided system message override
5. **Frozen MEMORY snapshot** -- memory tool contents at session start
6. **Frozen USER profile snapshot** -- user profile at session start
7. **Skills index** -- built by `build_skills_system_prompt()`, scanning `~/.hermes/skills/` for `SKILL.md` files grouped by category; filtered by platform compatibility and conditional activation rules
8. **Context files** -- discovered by `build_context_files_prompt()`: `AGENTS.md` (hierarchical, recursive), `.cursorrules`, `.cursor/rules/*.mdc`, `.hermes.md`/`HERMES.md` (walks to git root); `SOUL.md` is excluded here when already loaded as identity in step 1
9. **Timestamp and optional session ID** -- from `hermes_time.now()` (timezone-aware, configured via `HERMES_TIMEZONE` env var or `timezone` key in `config.yaml`)
10. **Platform hint** -- from `PLATFORM_HINTS` dict, keyed by platform name

When `skip_context_files` is set (for subagent delegation), `SOUL.md` is not loaded and the hardcoded `DEFAULT_AGENT_IDENTITY` is used instead.

## How SOUL.md Appears in the Prompt

`SOUL.md` lives at `~/.hermes/SOUL.md` and serves as the agent's identity -- the very first section of the system prompt:

```python
def load_soul_md() -> Optional[str]:
    soul_path = get_hermes_home() / "SOUL.md"
    if not soul_path.exists():
        return None
    content = soul_path.read_text(encoding="utf-8").strip()
    content = _scan_context_content(content, "SOUL.md")  # Security scan
    content = _truncate_content(content, "SOUL.md")       # Cap at 20k chars
    return content
```

When `load_soul_md()` returns content, it replaces the hardcoded `DEFAULT_AGENT_IDENTITY`. The `build_context_files_prompt()` function is then called with `skip_soul=True` to prevent SOUL.md from appearing twice.

If `SOUL.md` doesn't exist, the system falls back to the default identity string.

## How Context Files Are Injected

`build_context_files_prompt()` uses a priority system -- only one project context type is loaded (first match wins):

| Priority | Files | Search scope | Notes |
|----------|-------|-------------|-------|
| 1 | `.hermes.md`, `HERMES.md` | CWD up to git root | Hermes-native project config |
| 2 | `AGENTS.md` | CWD only | Common agent instruction file |
| 3 | `CLAUDE.md` | CWD only | Claude Code compatibility |
| 4 | `.cursorrules`, `.cursor/rules/*.mdc` | CWD only | Cursor compatibility |

All context files are:
- **Security scanned** -- checked for prompt injection patterns (invisible unicode, "ignore previous instructions", credential exfiltration attempts)
- **Truncated** -- capped at 20,000 characters using 70/20 head/tail ratio with a truncation marker
- **YAML frontmatter stripped** -- `.hermes.md` frontmatter is removed (reserved for future config overrides)

## Context File Security Scanning

`build_context_files_prompt()` and `load_soul_md()` run `_scan_context_content()` on every context file before injection. The scanner checks for:

- Invisible Unicode characters (zero-width `U+200B`-`U+200F`, `U+FEFF`, directional overrides `U+202A`-`U+202E`, `U+2060`-`U+2069`)
- Prompt injection patterns (`"ignore previous instructions"`, `"system prompt override"`, `"act as if you have no restrictions"`, `"disregard your rules"`, etc.)
- Data exfiltration commands (`curl ... $KEY`, `cat .env`, `cat credentials`)
- Hidden HTML elements (`<div style="display:none"`)

Files that match threat patterns are blocked and replaced with a `[BLOCKED: ...]` warning message.

## Skills Index Format

`build_skills_system_prompt()` produces a compact index in the format:

```
## Skills (mandatory)
<available_skills>
  category: description
    - skill-name: description
    - skill-name
  category:
    - skill-name: description
</available_skills>
```

Skills are filtered by:
- Platform compatibility (`skill_matches_platform()`)
- Disabled skills list
- Conditional activation: `fallback_for_toolsets`, `requires_toolsets`, `fallback_for_tools`, `requires_tools` from skill frontmatter

## API-Call-Time-Only Layers

These are intentionally excluded from the cached system prompt:

- `ephemeral_system_prompt` -- injected per-call
- Prefill messages
- Gateway-derived session context overlays
- Honcho recall injected into the current-turn user message via `_inject_honcho_turn_context()` (kept out of the cached prefix to preserve cache stability)

This separation keeps the stable prefix stable for caching.

## Memory Snapshots

Local memory and user profile data are injected as frozen snapshots at session start. Mid-session writes update disk state but do not mutate the already-built system prompt until a new session or forced rebuild occurs.

## System Prompt Layer Order

The cached system prompt is composed of exactly 10 layers, assembled in this order:

| Layer | Source | Content |
|-------|--------|---------|
| 1 | `load_soul_md()` / `DEFAULT_AGENT_IDENTITY` | Agent identity |
| 2 | `MEMORY_GUIDANCE`, `SESSION_SEARCH_GUIDANCE`, `SKILLS_GUIDANCE` | Tool-aware behavior guidance |
| 3 | Honcho integration | Static cross-session user modeling block |
| 4 | User-provided override | Optional system message |
| 5 | `memory_tool.py` | Frozen MEMORY snapshot at session start |
| 6 | User profile store | Frozen USER profile snapshot at session start |
| 7 | `build_skills_system_prompt()` | Skills index (filtered by platform, conditions) |
| 8 | `build_context_files_prompt()` | Context files (AGENTS.md, .cursorrules, .hermes.md, etc.) |
| 9 | `hermes_time.now()` | Timestamp and optional session ID |
| 10 | `PLATFORM_HINTS` dict | Platform-specific guidance |

## What Is NOT in the Cached Prompt

The following are intentionally excluded from the cached system prompt and injected only at API call time:

- **`ephemeral_system_prompt`** -- per-call injection, not persisted
- **Prefill messages** -- added after the message list is built
- **Honcho turn-context** -- injected into the current-turn user message via `_inject_honcho_turn_context()`, kept out of the cached prefix to preserve cache stability
- **Plugin turn context** -- plugin hooks that add per-turn context
- **Gateway-derived session context overlays** -- session metadata injected by platform adapters

This separation is the reason prompt caching works: the stable prefix stays identical across turns, and only the ephemeral tail changes.

## Skill Conditions Evaluation

Skills are filtered through multiple condition gates before inclusion in the skills index:

- **Platform gates**: `skill_matches_platform()` checks whether the skill's `platforms` frontmatter list includes the current platform (e.g. a skill marked `platforms: [cli]` is excluded from Telegram sessions)
- **Tool dependencies**: `requires_tools` in frontmatter lists tool names that must be available; `requires_toolsets` lists toolset names. The skill is excluded if any dependency is missing
- **Fallback activation**: `fallback_for_tools` and `fallback_for_toolsets` mark a skill as a fallback -- it is only included when the specified tools/toolsets are NOT available (e.g. a "manual file editing" skill that activates only when `terminal` is disabled)
- **Deprecated marking**: skills with `deprecated: true` in frontmatter are excluded from the index
- **Disabled list**: skills listed in the user's `disabled_skills` config are excluded

## Self-Optimized GPT/Codex Tool-Use Guidance (v0.8.0)

**New in v0.8.0** (PR [#6120](https://github.com/NousResearch/hermes-agent/pull/6120), [#5414](https://github.com/NousResearch/hermes-agent/pull/5414)): Hermes automatically injects structured execution guidance for GPT and Codex models (and now also `gemini`, `gemma`, and `grok` names). The constant `OPENAI_MODEL_EXECUTION_GUIDANCE` in `agent/prompt_builder.py` addresses 5 failure modes the agent self-diagnosed through automated behavioral benchmarking:

1. **Tool persistence** (`<tool_persistence>`) -- models abandoning work on partial results or stopping before verification
2. **Mandatory tool use** (`<mandatory_tool_use>`) -- models hallucinating answers to arithmetic, system state, or file content instead of calling tools
3. **Act-don't-ask** (`<act_dont_ask>`) -- models asking clarifying questions when an obvious default interpretation exists
4. **Prerequisite checks** (`<prerequisite_checks>`) -- models skipping prerequisite discovery/lookup before acting
5. **Verification** (`<verification>`) -- models declaring "done" without confirming correctness and grounding

The set of models this fires for is controlled by the `TOOL_USE_ENFORCEMENT_MODELS` tuple: `("gpt", "codex", "gemini", "gemma", "grok")`. This list was expanded in v0.8.0 from the earlier `("gpt", "codex")` set.

## Thinking-Only Prefill Continuation (v0.8.0)

**New in v0.8.0** (PR [#5931](https://github.com/NousResearch/hermes-agent/pull/5931)): When a model returns structured reasoning (via `reasoning`, `reasoning_content`, or `reasoning_details` API fields) but no visible text content, the system prompt layer is not mutated. Instead, the agent appends the incomplete assistant message (tagged with `_thinking_prefill = True`) to the conversation and makes another API call. The model sees its own reasoning on the next turn and typically produces the text portion. This "thinking-only prefill continuation" is retried up to 2 times before falling back to an `"(empty)"` response terminal. The `_thinking_prefill` marker is stripped from messages before they are sent to the API.

## Tool-Use Enforcement Details

The `agent.tool_use_enforcement` config controls injection of `TOOL_USE_ENFORCEMENT_GUIDANCE` into the system prompt:

| Value | Behavior |
|-------|----------|
| `"auto"` (default) | Inject for models matching `TOOL_USE_ENFORCEMENT_MODELS` (names containing `"gpt"` or `"codex"`) |
| `true` / `"always"` | Always inject, regardless of model |
| `false` / `"never"` | Never inject |
| list of strings | Inject when the model name contains any listed substring |

This is enabled by default for GPT and Codex models because they have a tendency to describe intended tool use in prose rather than actually emitting tool calls.

## Supported prompt customization surfaces

Most users should treat `agent/prompt_builder.py` as implementation code, not a configuration surface. The supported customization path is to change the prompt inputs Hermes already loads, rather than editing Python templates in place.

### Use these surfaces first

- `~/.hermes/SOUL.md` — replace the built-in default identity block with your own agent persona and standing behavior.
- `~/.hermes/MEMORY.md` and `~/.hermes/USER.md` — provide durable cross-session facts and user profile data that should be snapshotted into new sessions.
- Project context files such as `.hermes.md`, `HERMES.md`, `AGENTS.md`, `CLAUDE.md`, or `.cursorrules` — inject repo-specific working rules.
- Skills — package reusable workflows and references without editing core prompt code.
- Optional system prompt config / API overrides — add deployment-specific instruction text without forking Hermes.
- Ephemeral overlays such as `HERMES_EPHEMERAL_SYSTEM_PROMPT` or prefill messages — add turn-scoped guidance that should not become part of the cached prompt prefix.

### When to edit code instead

Edit `agent/prompt_builder.py` only if you are intentionally maintaining a fork or contributing upstream behavior changes. That file assembles the prompt plumbing, cache boundaries, and injection order for every session. Direct edits there are global product changes, not per-user prompt customization.

In other words:

- if you want a different assistant identity, edit `SOUL.md`
- if you want different repo rules, edit project context files
- if you want reusable operating procedures, add or modify skills
- if you want to change how Hermes assembles prompts for everyone, change Python and treat it as a code contribution

## Why Prompt Assembly Is Split This Way

The architecture is intentionally optimized to:

- Preserve provider-side prompt caching
- Avoid mutating history unnecessarily
- Keep memory semantics understandable
- Let gateway/ACP/CLI add context without poisoning persistent prompt state
