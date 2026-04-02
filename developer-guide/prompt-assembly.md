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

## Tool-Use Enforcement

GPT-family models have a tendency to describe intended actions without making tool calls. The `TOOL_USE_ENFORCEMENT_GUIDANCE` system prompt block is injected for models whose name contains `"gpt"` or `"codex"` (matched against `TOOL_USE_ENFORCEMENT_MODELS`). It instructs the model to call tools immediately rather than promising future action.

The injection behavior is controlled by `agent.tool_use_enforcement` in `config.yaml` (default: `"auto"`):

| Value | Behavior |
|-------|----------|
| `"auto"` (default) | Inject for models matching `TOOL_USE_ENFORCEMENT_MODELS` |
| `true` / `"always"` | Always inject, regardless of model |
| `false` / `"never"` | Never inject |
| list of strings | Inject when the model name contains any listed substring |

## Why Prompt Assembly Is Split This Way

The architecture is intentionally optimized to:

- Preserve provider-side prompt caching
- Avoid mutating history unnecessarily
- Keep memory semantics understandable
- Let gateway/ACP/CLI add context without poisoning persistent prompt state
