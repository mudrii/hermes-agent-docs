# Contributing

Hermes Agent is open source under the MIT license. Contributions are welcome via GitHub pull requests.

**Repository:** https://github.com/NousResearch/hermes-agent

v0.2.0 was built by 63 contributors in under 3 weeks — the project has high contribution velocity and an active maintainer team.

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Python | ≥ 3.11 | Required |
| uv | latest | Preferred package manager |
| Node.js | ≥ 18 | Optional (WhatsApp/Baileys) |
| An LLM API key | any | OpenRouter free tier works for testing |

---

## Development Setup

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
git submodule update --init mini-swe-agent

# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create virtualenv
uv venv .venv --python 3.11
source .venv/bin/activate

# Install with all extras + dev tools
uv pip install -e ".[all,dev]"
uv pip install -e "./mini-swe-agent"

# Verify
python -m pytest tests/ -q
hermes --version
```

---

## Running Tests

```bash
# Full test suite (3,289+ tests)
python -m pytest tests/ -q

# Specific module
python -m pytest tests/tools/ -q
python -m pytest tests/gateway/ -q

# Parallel (faster)
python -m pytest tests/ -q -n auto   # Requires pytest-xdist

# Single test
python -m pytest tests/test_agent_loop.py::test_tool_calling -v
```

**All tests must pass before submitting a PR.**

---

## Code Style

- **No `unwrap`-equivalents** — handle all errors properly (no bare `except: pass`)
- **Type hints** on all public functions
- **Docstrings** on all public classes and functions
- **`black`** formatting: `black hermes_cli/ agent/ tools/ gateway/`
- **`isort`** imports: `isort --profile black .`
- **`ruff`** linting: `ruff check .`

Run all checks:
```bash
black .
isort --profile black .
ruff check .
```

---

## Adding a New Tool

Tools are implemented in `tools/` and registered via the central registry.

### Step 1: Create the tool file

```python
# tools/my_tool.py

import json
import os
from tools.registry import registry

def check_my_tool() -> bool:
    """Return False if tool cannot be used (missing deps/keys)."""
    return bool(os.environ.get("MY_API_KEY"))

async def my_tool_handler(args: dict, **kwargs) -> str:
    """Handle tool call. Must return JSON string. Never raise."""
    try:
        result = do_something(args["input"])
        return json.dumps({"result": result})
    except Exception as e:
        return json.dumps({"error": str(e)})

MY_TOOL_SCHEMA = {
    "name": "my_tool",
    "description": "What this tool does (be specific — the LLM reads this)",
    "parameters": {
        "type": "object",
        "properties": {
            "input": {
                "type": "string",
                "description": "The input to process"
            }
        },
        "required": ["input"]
    }
}

registry.register(
    name="my_tool",
    toolset="my_toolset",
    schema=MY_TOOL_SCHEMA,
    handler=my_tool_handler,
    check_fn=check_my_tool,
    requires_env=["MY_API_KEY"]
)
```

### Step 2: Import in model_tools.py

```python
# model_tools.py
import tools.my_tool  # noqa — triggers registration
```

### Step 3: Add to toolsets.py

```python
# toolsets.py
TOOLSETS = {
    ...
    "my_toolset": {
        "description": "My toolset description",
        "tools": ["my_tool"],
        "includes": []
    },
    ...
}
```

### Step 4: Add tests

```python
# tests/tools/test_my_tool.py

import pytest
from tools.my_tool import my_tool_handler

@pytest.mark.asyncio
async def test_my_tool_basic():
    result = await my_tool_handler({"input": "test"})
    data = json.loads(result)
    assert "result" in data

@pytest.mark.asyncio
async def test_my_tool_error_handling():
    result = await my_tool_handler({"input": ""})
    data = json.loads(result)
    assert "error" in data
```

---

## Adding a Platform Adapter

Platform adapters live in `gateway/platforms/`.

### Implement the Adapter

```python
# gateway/platforms/my_platform.py

from gateway.platforms.base import BasePlatformAdapter
from gateway.session import SessionSource

class MyPlatformAdapter(BasePlatformAdapter):
    name = "my_platform"

    async def connect(self) -> None:
        """Initialize connection to platform API."""
        pass

    async def disconnect(self) -> None:
        """Clean up connections."""
        pass

    async def send_text(self, session_source: SessionSource, text: str) -> None:
        """Send text message to platform."""
        pass

    async def send_image(self, session_source: SessionSource, image_path: str) -> None:
        """Send image to platform (optional)."""
        pass

    async def start_listening(self) -> None:
        """Start receiving messages and dispatch via self.handle_message()."""
        while True:
            msg = await receive_from_platform()
            source = SessionSource(
                platform="my_platform",
                chat_id=msg.chat_id,
                user_id=msg.user_id,
            )
            await self.handle_message(source, msg.text)
```

### Register in Gateway

```python
# gateway/__init__.py
from gateway.platforms.my_platform import MyPlatformAdapter
PLATFORM_ADAPTERS["my_platform"] = MyPlatformAdapter
```

---

## Adding a Bundled Skill

1. Create directory: `skills/<category>/<skill-name>/`
2. Create `SKILL.md` with the [SKILL.md format](skills.md#creating-skills)
3. Skills are loaded automatically — no Rust/Python code changes needed
4. Test by running `hermes skills list` and verifying the skill appears

Example:

```bash
mkdir -p skills/productivity/my-skill
cat > skills/productivity/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: Expert at doing X
version: 1.0.0
author: yourname
license: MIT
metadata:
  hermes:
    tags: [Productivity, Tools]
---

# My Skill

## When to Use
When the user asks to do X.

## Procedure
1. Step one
2. Step two

## Verification
Check that the output is Y.
EOF
```

---

## Pull Request Process

1. Fork the repository
2. Create a branch: `git checkout -b feat/my-feature`
3. Implement with tests
4. Ensure all tests pass: `python -m pytest tests/ -q`
5. Run linting: `ruff check . && black --check . && isort --check-only .`
6. Commit using conventional commit format
7. Open PR against `main`

**Commit format:**
```
feat: add my_tool for doing X
fix: handle empty response from platform Y
docs: add ACP editor setup guide
test: add integration tests for gateway session reset
chore: update dependencies
```

---

## Key Test Files

| File | What It Tests |
|------|---------------|
| `tests/test_agent_loop.py` | Core agent loop iteration |
| `tests/test_agent_loop_tool_calling.py` | Tool dispatch and parallel execution |
| `tests/test_413_compression.py` | Context overflow handling |
| `tests/test_cli_approval_ui.py` | Command approval prompts |
| `tests/test_cli_interrupt_subagent.py` | Interrupt handling |
| `tests/test_toolsets.py` | Toolset resolution |
| `tests/tools/` | Individual tool unit tests |
| `tests/gateway/` | Platform gateway integration |
| `tests/cron/` | Scheduled task execution |

---

## Architecture Notes for Contributors

- **Registry pattern:** Tools self-register at import time — always use `registry.register()`, never direct dispatch
- **Error handling:** Tool handlers must never raise — return `{"error": "..."}` JSON instead
- **Atomic writes:** Use `utils.atomic_json_write()` and `utils.atomic_yaml_write()` for all config/data writes
- **Parallel tools:** The agent can call up to 8 tools concurrently — ensure handlers are thread-safe
- **Callbacks:** Use the callback pattern (not direct side effects) for UI updates during tool execution
- **Iteration budget:** Respect the shared `IterationBudget` when spawning subagents — don't bypass it
