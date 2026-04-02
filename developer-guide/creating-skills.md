# Creating Skills

Skills are the preferred way to add new capabilities to Hermes Agent. They are easier to create than tools, require no code changes to the agent, and can be shared with the community.

## Should It Be a Skill or a Tool?

Make it a **Skill** when:
- The capability can be expressed as instructions + shell commands + existing tools
- It wraps an external CLI or API that the agent can call via `terminal` or `web_extract`
- It doesn't need custom Python integration or API key management baked into the agent
- Examples: arXiv search, git workflows, Docker management, PDF processing, email via CLI tools

Make it a **Tool** when:
- It requires end-to-end integration with API keys, auth flows, or multi-component configuration
- It needs custom processing logic that must execute precisely every time
- It handles binary data, streaming, or real-time events
- Examples: browser automation, TTS, vision analysis

## Skill Directory Structure

Bundled skills live in `skills/` organized by category. Official optional skills use the same structure in `optional-skills/`:

```
skills/
+-- research/
|   +-- arxiv/
|       +-- SKILL.md              Required: main instructions
|       +-- scripts/              Optional: helper scripts
|           +-- search_arxiv.py
+-- productivity/
|   +-- ocr-and-documents/
|       +-- SKILL.md
|       +-- scripts/
|       +-- references/
+-- ...
```

## SKILL.md Format

```markdown
---
name: my-skill
description: Brief description (shown in skill search results)
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [Category, Subcategory, Keywords]
    related_skills: [other-skill-name]
    requires_toolsets: [web]
    requires_tools: [web_search]
    fallback_for_toolsets: [browser]
    fallback_for_tools: [browser_navigate]
required_environment_variables:
  - name: MY_API_KEY
    prompt: "Enter your API key"
    help: "Get one at https://example.com"
    required_for: "API access"
---

# Skill Title

Brief intro.

## When to Use
Trigger conditions.

## Quick Reference
Table of common commands or API calls.

## Procedure
Step-by-step instructions the agent follows.

## Pitfalls
Known failure modes and how to handle them.

## Verification
How the agent confirms it worked.
```

## Platform-Specific Skills

Skills can restrict themselves to specific operating systems using the `platforms` field:

```yaml
platforms: [macos]            # macOS only
platforms: [macos, linux]     # macOS and Linux
platforms: [windows]          # Windows only
```

When set, the skill is automatically hidden from the system prompt, `skills_list()`, and slash commands on incompatible platforms. If omitted, the skill loads on all platforms.

## Conditional Skill Activation

Skills can declare dependencies on specific tools or toolsets:

```yaml
metadata:
  hermes:
    requires_toolsets: [web]
    requires_tools: [web_search]
    fallback_for_toolsets: [browser]
    fallback_for_tools: [browser_navigate]
```

| Field | Behavior |
|-------|----------|
| `requires_toolsets` | Skill is hidden when ANY listed toolset is NOT available |
| `requires_tools` | Skill is hidden when ANY listed tool is NOT available |
| `fallback_for_toolsets` | Skill is hidden when ANY listed toolset IS available |
| `fallback_for_tools` | Skill is hidden when ANY listed tool IS available |

Use `fallback_for_*` to create workaround skills that only show when a primary tool is unavailable. Use `requires_*` for skills that only make sense when certain tools are present.

## Environment Variable Requirements

Skills can declare environment variables they need:

```yaml
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: "Tenor API key"
    help: "Get your key at https://tenor.com"
    required_for: "GIF search functionality"
```

Each entry supports:
- `name` (required) -- the environment variable name
- `prompt` (optional) -- prompt text when asking the user for the value
- `help` (optional) -- help text or URL for obtaining the value
- `required_for` (optional) -- describes which feature needs this variable

Missing values do not hide the skill from discovery. Hermes prompts for them securely when the skill is loaded in the local CLI. Declared variables that are set are automatically passed through to `execute_code` and `terminal` sandboxes, including remote backends like Docker and Modal.

## Credential File Requirements

Skills that use OAuth or file-based credentials can declare files that need to be mounted into remote sandboxes:

```yaml
required_credential_files:
  - path: google_token.json
    description: Google OAuth2 token (created by setup script)
  - path: google_client_secret.json
    description: Google OAuth2 client credentials
```

Each `path` is relative to `~/.hermes/`. When loaded, existing files are automatically mounted into Docker containers and synced into Modal sandboxes. Missing files trigger `setup_needed`.

## Skill Guidelines

### No External Dependencies
Prefer stdlib Python, curl, and existing Hermes tools. If a dependency is needed, document installation steps in the skill.

### Progressive Disclosure
Put the most common workflow first. Edge cases and advanced usage go at the bottom. This keeps token usage low for common tasks.

### Include Helper Scripts
For XML/JSON parsing or complex logic, include helper scripts in `scripts/` -- don't expect the LLM to write parsers inline every time.

### Test It
```bash
hermes chat --toolsets skills -q "Use the X skill to do Y"
```

## Where Should the Skill Live?

- **`skills/`** -- Bundled with every install. Broadly useful to most users.
- **`optional-skills/`** -- Ships with the repo, discoverable via `hermes skills browse`, labeled "official". For paid service integrations or heavyweight dependencies.
- **Skills Hub** -- For specialized, community-contributed, or niche skills. Upload via `hermes skills publish` and share via `hermes skills install`.

## Publishing Skills

### To the Skills Hub

```bash
hermes skills publish skills/my-skill --to github --repo owner/repo
```

### To a Custom Repository

```bash
hermes skills tap add owner/repo
```

Users can then search and install from your repository.

## Security Scanning

All hub-installed skills go through a security scanner that checks for data exfiltration patterns, prompt injection attempts, destructive commands, and shell injection.

Trust levels:
- `builtin` -- ships with Hermes (always trusted)
- `official` -- from `optional-skills/` in the repo (builtin trust)
- `trusted` -- from recognized skill repositories
- `community` -- non-dangerous findings can be overridden with `--force`; `dangerous` verdicts remain blocked

Third-party skills can be consumed from GitHub identifiers, `skills.sh` identifiers, and well-known endpoints served from `/.well-known/skills/index.json`.
