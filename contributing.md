# Contributing

Hermes Agent is open source under the MIT license. Contributions are welcome via GitHub pull requests.

**Repository:** https://github.com/NousResearch/hermes-agent

---

## Contribution Priorities

Contributions are valued in this order:

1. **Bug fixes** — crashes, incorrect behavior, data loss. Always top priority.
2. **Cross-platform compatibility** — macOS, different Linux distros, WSL2. Hermes should work everywhere.
3. **Security hardening** — shell injection, prompt injection, path traversal, privilege escalation.
4. **Performance and robustness** — retry logic, error handling, graceful degradation.
5. **New skills** — broadly useful ones only. See [Skills vs. Tools](#skills-vs-tools) below.
6. **New tools** — rarely needed. Most new capabilities should be skills.
7. **Documentation** — fixes, clarifications, new examples.

---

## Skills vs. Tools

This is the most common decision for new contributors. The answer is almost always **skill**.

### Make it a Skill when:

- The capability can be expressed as instructions plus shell commands plus existing tools
- It wraps an external CLI or API that the agent can call via `terminal` or `web_extract`
- It does not need custom Python integration or API key management baked into the agent
- Examples: arXiv search, git workflows, Docker management, PDF processing, email via CLI tools

### Make it a Tool when:

- It requires end-to-end integration with API keys, auth flows, or multi-component configuration managed by the agent harness
- It needs custom processing logic that must execute precisely every time (not best-effort from LLM interpretation)
- It handles binary data, streaming, or real-time events that cannot go through the terminal
- Examples: browser automation (Browserbase session management), TTS (audio encoding and platform delivery), vision analysis (base64 image handling)

### Should the Skill be bundled?

Bundled skills in `skills/` ship with every Hermes install. They should be broadly useful to most users:
- Document handling, web research, common dev workflows, system administration
- Used regularly by a wide range of people

If your skill is official and useful but not universally needed (e.g., a paid service integration, a heavyweight dependency), put it in `optional-skills/` — it ships with the repo but is not activated by default. Users can discover it via `hermes skills browse` and install it with `hermes skills install`.

If your skill is specialized, community-contributed, or niche, it is better suited for a Skills Hub registry — upload it and share it in the Nous Research Discord.

---

## Development Environment Setup

### Prerequisites

| Requirement | Notes |
|-------------|-------|
| **Git** | With `--recurse-submodules` support |
| **Python 3.11+** | uv will install it if missing |
| **uv** | Fast Python package manager — install from https://docs.astral.sh/uv/ |
| **Node.js 18+** | Optional — needed for browser tools and WhatsApp bridge |

### Clone and Install

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# Create venv with Python 3.11
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"

# Install with all extras plus dev tools
uv pip install -e ".[all,dev]"
uv pip install -e "./mini-swe-agent"
uv pip install -e "./tinker-atropos"

# Optional: browser tools and WhatsApp bridge
npm install
```

### Configure for Development

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env

# Add at minimum an LLM provider key:
echo 'OPENROUTER_API_KEY=sk-or-v1-your-key' >> ~/.hermes/.env
```

### Make `hermes` Available Globally

```bash
mkdir -p ~/.local/bin
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
```

### Verify

```bash
hermes doctor
hermes chat -q "Hello"
```

---

## Running Tests

```bash
pytest tests/ -v
```

The test suite uses `pytest-xdist` for parallel execution (`-n auto` is set by default in `pyproject.toml`). Integration tests that require external services (API keys, Modal, etc.) are marked with `@pytest.mark.integration` and are excluded from the default run.

To run integration tests explicitly:

```bash
pytest tests/ -v -m integration
```

The test configuration in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "integration: marks tests requiring external services (API keys, Modal, etc.)",
]
addopts = "-m 'not integration' -n auto"
```

The ACP test suite is in `tests/acp/`.

---

## Project Structure

```
hermes-agent/
├── run_agent.py              # AIAgent class — core conversation loop, tool dispatch, session persistence
├── cli.py                    # HermesCLI class — interactive TUI, prompt_toolkit integration
├── model_tools.py            # Tool orchestration (thin layer over tools/registry.py)
├── toolsets.py               # Tool groupings and presets (hermes-cli, hermes-telegram, etc.)
├── hermes_state.py           # SQLite session database with FTS5 full-text search, session titles
├── batch_runner.py           # Parallel batch processing for trajectory generation
│
├── agent/                    # Agent internals
│   ├── prompt_builder.py         # System prompt assembly (identity, skills, context files, memory)
│   ├── context_compressor.py     # Auto-summarization when approaching context limits
│   ├── auxiliary_client.py       # Resolves auxiliary OpenAI clients (summarization, vision)
│   ├── display.py                # KawaiiSpinner, tool progress formatting
│   ├── model_metadata.py         # Model context lengths, token estimation
│   └── trajectory.py             # Trajectory saving helpers
│
├── hermes_cli/               # CLI command implementations
│   ├── main.py                   # Entry point, argument parsing, command dispatch
│   ├── config.py                 # Config management, migration, env var definitions
│   ├── setup.py                  # Interactive setup wizard
│   ├── auth.py                   # Provider resolution, OAuth, Nous Portal
│   ├── copilot_auth.py           # GitHub Copilot OAuth device code flow
│   ├── runtime_provider.py       # Shared runtime provider resolution
│   ├── models.py                 # OpenRouter model selection lists
│   ├── banner.py                 # Welcome banner, ASCII art
│   ├── commands.py               # Slash command registry (CommandDef), autocomplete
│   ├── callbacks.py              # Interactive callbacks (clarify, sudo, approval)
│   ├── doctor.py                 # Diagnostics
│   ├── skills_hub.py             # Skills Hub CLI + /skills slash command
│   └── skin_engine.py            # Skin/theme engine — data-driven CLI visual customization
│
├── tools/                    # Tool implementations (self-registering)
│   ├── registry.py               # Central tool registry (schemas, handlers, dispatch)
│   ├── approval.py               # Dangerous command detection + per-session approval
│   ├── terminal_tool.py          # Terminal orchestration (sudo, env lifecycle, backends)
│   ├── file_operations.py        # read_file, write_file, search, patch, etc.
│   ├── web_tools.py              # web_search, web_extract
│   ├── vision_tools.py           # Image analysis via multimodal models
│   ├── delegate_tool.py          # Subagent spawning and parallel task execution
│   ├── code_execution_tool.py    # Sandboxed Python with RPC tool access
│   ├── session_search_tool.py    # Search past conversations with FTS5 + summarization
│   ├── cronjob_tools.py          # Scheduled task management
│   ├── skill_tools.py            # Skill search, load, manage
│   └── environments/             # Terminal execution backends
│       ├── base.py                   # BaseEnvironment ABC
│       ├── local.py, docker.py, ssh.py, singularity.py, modal.py, daytona.py
│
├── gateway/                  # Messaging gateway
│   ├── run.py                    # GatewayRunner — platform lifecycle, message routing, cron
│   ├── config.py                 # Platform configuration resolution
│   ├── session.py                # Session store, context prompts, reset policies
│   └── platforms/                # Platform adapters
│       ├── telegram.py, discord_adapter.py, slack.py, whatsapp.py
│
├── acp_adapter/              # ACP editor integration
│   ├── entry.py, server.py, session.py, events.py, permissions.py, tools.py, auth.py
│
├── honcho_integration/       # Honcho memory integration
│   ├── client.py, session.py, cli.py
│
├── scripts/                  # Installer and bridge scripts
│   ├── install.sh                # Linux/macOS installer
│   └── whatsapp-bridge/          # Node.js WhatsApp bridge (Baileys)
│
├── skills/                   # Bundled skills (copied to ~/.hermes/skills/ on install)
├── optional-skills/          # Official optional skills (not activated by default)
├── environments/             # RL training environments (Atropos integration)
├── tests/                    # Test suite
├── website/                  # Documentation site
│
├── setup-hermes.sh           # Developer setup script
├── cli-config.yaml.example   # Example configuration
└── AGENTS.md                 # Development guide for AI coding assistants
```

### User Configuration (`~/.hermes/`)

| Path | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | Settings (model, terminal, toolsets, compression, etc.) |
| `~/.hermes/.env` | API keys and secrets |
| `~/.hermes/auth.json` | OAuth credentials (Nous Portal) |
| `~/.hermes/skills/` | All active skills (bundled, hub-installed, agent-created) |
| `~/.hermes/memories/` | Persistent memory (MEMORY.md, USER.md) |
| `~/.hermes/state.db` | SQLite session database |
| `~/.hermes/sessions/` | JSON session logs |
| `~/.hermes/cron/` | Scheduled job data |
| `~/.hermes/whatsapp/session/` | WhatsApp bridge credentials |

---

## Architecture Overview

### Core Loop

```
User message -> AIAgent._run_agent_loop()
  ├── Build system prompt (prompt_builder.py)
  ├── Build API kwargs (model, messages, tools, reasoning config)
  ├── Call LLM (OpenAI-compatible API)
  ├── If tool_calls in response:
  │     ├── Execute each tool via registry dispatch
  │     ├── Add tool results to conversation
  │     └── Loop back to LLM call
  ├── If text response:
  │     ├── Persist session to DB
  │     └── Return final_response
  └── Context compression if approaching token limit
```

### Key Design Patterns

- **Self-registering tools**: Each tool file calls `registry.register()` at import time. `model_tools.py` triggers discovery by importing all tool modules.
- **Toolset grouping**: Tools are grouped into toolsets (`web`, `terminal`, `file`, `browser`, etc.) that can be enabled or disabled per platform.
- **Session persistence**: All conversations are stored in SQLite with full-text search and unique session titles. JSON logs go to `~/.hermes/sessions/`.
- **Ephemeral injection**: System prompts and prefill messages are injected at API call time, never persisted to the database or logs.
- **Provider abstraction**: The agent works with any OpenAI-compatible API. Provider resolution happens at init time (OAuth, API key, or custom endpoint).

---

## Code Style

- **PEP 8** with practical exceptions (no strict line length enforcement)
- **Comments**: Only when explaining non-obvious intent, trade-offs, or API quirks. Do not narrate what the code does.
- **Error handling**: Catch specific exceptions. Use `logger.warning()` and `logger.error()` with `exc_info=True` for unexpected errors so stack traces appear in logs.
- **Cross-platform**: Never assume Unix. See [Cross-Platform Compatibility](#cross-platform-compatibility) below.

---

## Adding a New Tool

Before writing a tool, confirm it cannot be a skill. Tools self-register with the central registry. Each tool file co-locates its schema, handler, and registration:

```python
"""my_tool — Brief description of what this tool does."""

import json
from tools.registry import registry


def my_tool(param1: str, param2: int = 10, **kwargs) -> str:
    """Handler. Returns a string result (often JSON)."""
    result = do_work(param1, param2)
    return json.dumps(result)


MY_TOOL_SCHEMA = {
    "type": "function",
    "function": {
        "name": "my_tool",
        "description": "What this tool does and when the agent should use it.",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {"type": "string", "description": "What param1 is"},
                "param2": {"type": "integer", "description": "What param2 is", "default": 10},
            },
            "required": ["param1"],
        },
    },
}


def _check_requirements() -> bool:
    return True


registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema=MY_TOOL_SCHEMA,
    handler=lambda args, **kw: my_tool(**args, **kw),
    check_fn=_check_requirements,
)
```

Then add the import to `model_tools.py` in the `_modules` list. If it is a new toolset, add it to `toolsets.py` and to the relevant platform presets.

---

## Adding a Skill

Bundled skills live in `skills/` organized by category. The structure:

```
skills/
├── research/
│   └── arxiv/
│       ├── SKILL.md              # Required: main instructions
│       └── scripts/              # Optional: helper scripts
│           └── search_arxiv.py
├── productivity/
│   └── ocr-and-documents/
│       ├── SKILL.md
│       ├── scripts/
│       └── references/
└── ...
```

### SKILL.md Format

```markdown
---
name: my-skill
description: Brief description (shown in skill search results)
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]          # Optional — restrict to specific OS platforms
required_environment_variables:    # Optional — secure setup-on-load metadata
  - name: MY_API_KEY
    prompt: API key
    help: Where to get it
    required_for: full functionality
metadata:
  hermes:
    tags: [Category, Subcategory, Keywords]
    related_skills: [other-skill-name]
    fallback_for_toolsets: [web]       # Show only when toolset is unavailable
    requires_toolsets: [terminal]      # Show only when toolset is available
---

# Skill Title

## When to Use

## Quick Reference

## Procedure

## Pitfalls

## Verification
```

### Skill Guidelines

- No external dependencies unless absolutely necessary. Prefer stdlib Python, curl, and existing Hermes tools (`web_extract`, `terminal`, `read_file`).
- Put the most common workflow first. Edge cases go at the bottom.
- Include helper scripts for XML/JSON parsing or complex logic.
- Test with: `hermes --toolsets skills -q "Use the X skill to do Y"`

### Platform-Specific Skills

Skills can declare which OS platforms they support via the `platforms` frontmatter field:

```yaml
platforms: [macos]            # macOS only
platforms: [macos, linux]     # macOS and Linux
platforms: [windows]          # Windows only
```

If omitted, the skill loads on all platforms.

### Conditional Skill Activation

```yaml
metadata:
  hermes:
    fallback_for_toolsets: [web]      # Show ONLY when web toolset is unavailable
    requires_toolsets: [terminal]     # Show ONLY when terminal toolset is available
    fallback_for_tools: [web_search]  # Show ONLY when web_search tool is unavailable
    requires_tools: [terminal]        # Show ONLY when terminal tool is available
```

- `fallback_for_*`: hidden when the listed tools are available, shown when unavailable
- `requires_*`: shown only when the listed tools are available

---

## Cross-Platform Compatibility

Hermes officially supports Linux, macOS, and WSL2. Key rules:

### 1. termios and fcntl are Unix-only

Always catch both `ImportError` and `NotImplementedError`:

```python
try:
    from simple_term_menu import TerminalMenu
    menu = TerminalMenu(options)
    idx = menu.show()
except (ImportError, NotImplementedError):
    # Fallback: numbered menu
    for i, opt in enumerate(options):
        print(f"  {i+1}. {opt}")
    idx = int(input("Choice: ")) - 1
```

### 2. File encoding

Some environments save `.env` files in non-UTF-8 encodings:

```python
try:
    load_dotenv(env_path)
except UnicodeDecodeError:
    load_dotenv(env_path, encoding="latin-1")
```

### 3. Process management

`os.setsid()`, `os.killpg()`, and signal handling differ across platforms:

```python
import platform
if platform.system() != "Windows":
    kwargs["preexec_fn"] = os.setsid
```

### 4. Path separators

Use `pathlib.Path` instead of string concatenation with `/`.

---

## Security Considerations

Hermes has terminal access. Security matters.

When contributing security-sensitive code:

- Always use `shlex.quote()` when interpolating user input into shell commands
- Resolve symlinks with `os.path.realpath()` before path-based access control checks
- Do not log secrets — API keys, tokens, and passwords must never appear in log output
- Catch broad exceptions around tool execution so a single failure does not crash the agent loop
- Test on all platforms if your change touches file paths, process management, or shell commands

If your PR affects security, note it explicitly in the PR description.

Existing security layers to understand before contributing:

| Layer | Implementation |
|-------|---------------|
| Sudo password piping | Uses `shlex.quote()` to prevent shell injection |
| Dangerous command detection | Regex patterns in `tools/approval.py` with user approval flow |
| Cron prompt injection | Scanner in `tools/cronjob_tools.py` blocks instruction-override patterns |
| Write deny list | Protected paths resolved via `os.path.realpath()` to prevent symlink bypass |
| Skills guard | Security scanner for hub-installed skills (`tools/skills_guard.py`) |
| Code execution sandbox | `execute_code` child process runs with API keys stripped from environment |
| Container hardening | Docker: all capabilities dropped, no privilege escalation, PID limits |

---

## Pull Request Process

### Branch Naming

```
fix/description        # Bug fixes
feat/description       # New features
docs/description       # Documentation
test/description       # Tests
refactor/description   # Code restructuring
```

### Before Submitting

1. **Run tests**: `pytest tests/ -v`
2. **Test manually**: Run `hermes` and exercise the code path you changed
3. **Check cross-platform impact**: Consider macOS and different Linux distros if you touch file I/O, process management, or terminal handling
4. **Keep PRs focused**: One logical change per PR. Do not mix a bug fix with a refactor with a new feature.

### PR Description

Include:
- **What** changed and **why**
- **How to test** it (reproduction steps for bugs, usage examples for features)
- **What platforms** you tested on
- Reference to any related issues

### Commit Messages

Hermes uses [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>
```

| Type | Use for |
|------|---------|
| `fix` | Bug fixes |
| `feat` | New features |
| `docs` | Documentation |
| `test` | Tests |
| `refactor` | Code restructuring (no behavior change) |
| `chore` | Build, CI, dependency updates |

Valid scopes: `cli`, `gateway`, `tools`, `skills`, `agent`, `install`, `whatsapp`, `security`, and others.

Examples:

```
fix(cli): prevent crash in save_config_value when model is a string
feat(gateway): add WhatsApp multi-user session isolation
fix(security): prevent shell injection in sudo password piping
test(tools): add unit tests for file_operations
```

---

## Reporting Issues

- Use [GitHub Issues](https://github.com/NousResearch/hermes-agent/issues)
- Include: OS, Python version, Hermes version (`hermes version`), full error traceback
- Include steps to reproduce
- Check existing issues before creating duplicates
- For security vulnerabilities, please report privately (do not open a public issue)

---

## Areas Looking for Contributions

Based on the contribution priority order:

- **Bug fixes** — check the GitHub Issues page for open `bug` labeled issues
- **Cross-platform testing** — macOS-specific fixes, different Linux distro compatibility
- **New bundled skills** — broadly useful skills in `skills/` (document processing, research tools, dev workflow automation)
- **Official optional skills** — service-specific skills in `optional-skills/` (paid APIs, heavyweight tools)
- **Test coverage** — additional unit tests for tools, gateway session management, and auth flows
- **Documentation** — accuracy fixes, missing examples, new guides

---

## Community Resources

- **Discord**: [discord.gg/NousResearch](https://discord.gg/NousResearch) — for questions, showcasing projects, and sharing skills
- **GitHub Discussions**: For design proposals and architecture discussions
- **GitHub Issues**: [github.com/NousResearch/hermes-agent/issues](https://github.com/NousResearch/hermes-agent/issues) — for bug reports and feature requests
- **Skills Hub**: Upload specialized skills to a registry and share them with the community via `hermes skills install`

---

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).
