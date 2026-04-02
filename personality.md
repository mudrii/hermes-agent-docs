# Personality and SOUL.md

Hermes Agent's personality is fully customizable. `SOUL.md` is the **primary identity** -- it's the first thing in the system prompt and defines who the agent is.

- `SOUL.md` -- a durable persona file that lives in `HERMES_HOME` and serves as the agent's identity (slot #1 in the system prompt)
- Built-in or custom `/personality` presets -- session-level system-prompt overlays

SOUL.md as primary agent identity was introduced in v0.3.0 ([PR #1922](https://github.com/NousResearch/hermes-agent/pull/1922)). Default SOUL.md auto-seeding was added in v0.3.0 ([PR #1311](https://github.com/NousResearch/hermes-agent/pull/1311)). The `/personality` command was contributed in v0.2.0 ([PR #773](https://github.com/NousResearch/hermes-agent/pull/773)).

## SOUL.md

### Location

```
~/.hermes/SOUL.md
```

If you run Hermes with a custom home directory:

```
$HERMES_HOME/SOUL.md
```

### Behavior

- **SOUL.md is the agent's primary identity.** It occupies slot #1 in the system prompt, replacing the hardcoded default identity.
- Hermes creates a starter `SOUL.md` automatically if one does not exist yet
- Existing user `SOUL.md` files are never overwritten
- Hermes loads `SOUL.md` only from `HERMES_HOME` -- not from the current working directory
- If `SOUL.md` exists but is empty, or cannot be loaded, Hermes falls back to a built-in default identity ("You are Hermes Agent, an intelligent AI assistant created by Nous Research...")
- If `SOUL.md` has content, that content is injected verbatim after security scanning and truncation
- SOUL.md appears only once in the prompt -- it is not duplicated in the context files section

Loading only from `HERMES_HOME` keeps personality predictable. If Hermes loaded `SOUL.md` from whatever directory you happened to launch it in, your personality could change unexpectedly between projects.

### What Should Go in SOUL.md

Use it for durable voice and personality guidance:
- Tone and communication style
- Level of directness
- Default interaction style
- What to avoid stylistically
- How to handle uncertainty, disagreement, or ambiguity

Use it less for:
- One-off project instructions
- File paths, repo conventions
- Temporary workflow details

Those belong in `AGENTS.md`, not `SOUL.md`.

### Example

```markdown
# Personality

You are a pragmatic senior engineer with strong taste.
You optimize for truth, clarity, and usefulness over politeness theater.

## Style
- Be direct without being cold
- Prefer substance over filler
- Push back when something is a bad idea
- Admit uncertainty plainly
- Keep explanations compact unless depth is useful

## What to avoid
- Sycophancy
- Hype language
- Repeating the user's framing if it's wrong
- Overexplaining obvious things

## Technical posture
- Prefer simple systems over clever systems
- Care about operational reality, not idealized architecture
- Treat edge cases as part of the design, not cleanup
```

### Security Scanning

SOUL.md is scanned like other context-bearing files for prompt injection patterns before inclusion. Keep it focused on persona/voice rather than trying to include meta-instructions.

## SOUL.md vs AGENTS.md

| | SOUL.md | AGENTS.md |
|---|---------|-----------|
| **Purpose** | Identity, tone, style, personality | Project architecture, conventions, workflows |
| **Scope** | Follows you everywhere | Belongs to a project |
| **Location** | `~/.hermes/SOUL.md` | Project root |
| **Examples** | Communication style, directness level | Coding conventions, tool preferences, ports, paths |

A useful rule: if it should follow you everywhere, it belongs in `SOUL.md`. If it belongs to a project, it belongs in `AGENTS.md`.

## SOUL.md vs /personality

`SOUL.md` is your durable default personality. `/personality` is a session-level overlay that changes or supplements the current system prompt.

- `SOUL.md` = baseline voice
- `/personality` = temporary mode switch

Examples:
- Keep a pragmatic default SOUL, then use `/personality teacher` for a tutoring conversation
- Keep a concise SOUL, then use `/personality creative` for brainstorming

## Built-in Personalities

Hermes ships with built-in personalities you can switch to with `/personality`:

| Name | Description |
|------|-------------|
| **helpful** | Friendly, general-purpose assistant |
| **concise** | Brief, to-the-point responses |
| **technical** | Detailed, accurate technical expert |
| **creative** | Innovative, outside-the-box thinking |
| **teacher** | Patient educator with clear examples |
| **kawaii** | Cute expressions, sparkles, and enthusiasm |
| **catgirl** | Neko-chan with cat-like expressions |
| **pirate** | Captain Hermes, tech-savvy buccaneer |
| **shakespeare** | Bardic prose with dramatic flair |
| **surfer** | Totally chill bro vibes |
| **noir** | Hard-boiled detective narration |
| **uwu** | Maximum cute with uwu-speak |
| **philosopher** | Deep contemplation on every query |
| **hype** | Maximum energy and enthusiasm |

## Switching Personalities

### CLI

```text
/personality
/personality concise
/personality technical
```

### Messaging Platforms

```text
/personality teacher
```

These are session-level overlays. Your global `SOUL.md` still gives Hermes its persistent default personality unless the overlay meaningfully changes it.

## Custom Personalities in Config

Define named custom personalities in `~/.hermes/config.yaml` under `agent.personalities`:

```yaml
agent:
  personalities:
    codereviewer: >
      You are a meticulous code reviewer. Identify bugs, security issues,
      performance concerns, and unclear design choices. Be precise and constructive.
```

Then switch with:

```text
/personality codereviewer
```

## How Personality Interacts with the Full Prompt

The prompt stack:

1. **SOUL.md** (agent identity -- or built-in fallback if SOUL.md is unavailable)
2. Tool-aware behavior guidance
3. Memory/user context
4. Skills guidance
5. Context files (`AGENTS.md`, `.cursorrules`)
6. Timestamp
7. Platform-specific formatting hints
8. Optional system-prompt overlays such as `/personality`

`SOUL.md` is the foundation -- everything else builds on top of it.

## CLI Appearance vs Conversational Personality

Conversational personality and CLI appearance are separate:

- `SOUL.md`, `agent.system_prompt`, and `/personality` affect how Hermes speaks
- `display.skin` and `/skin` affect how Hermes looks in the terminal

For terminal appearance, see [Skins and Themes](./skins.md).
