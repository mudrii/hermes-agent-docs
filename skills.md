# Skills

Skills are persistent Markdown documents that inject procedural knowledge into the agent's system prompt. They follow the [agentskills.io](https://agentskills.io) open standard and are shared across Claude Code, Cursor, GitHub Copilot, OpenHands, and other AI agents.

---

## How Skills Work

1. When a skill is enabled, its `SKILL.md` content is appended to the system prompt
2. The agent reads the skill's instructions and applies them when relevant
3. Skills can include linked reference files, templates, scripts, and assets
4. The agent can autonomously create new skills using `skill_manage`

---

## Managing Skills

```bash
# View installed skills
hermes skills list

# Browse available skills
hermes skills browse
hermes skills browse --source official      # Official optional skills
hermes skills browse --source skills-sh     # skills.sh community directory

# Search
hermes skills search kubernetes
hermes skills search pytorch --source skills-sh

# Install
hermes skills install github                # Bundled skill
hermes skills install official/security/1password   # Optional skill
hermes skills install openai/skills/k8s     # GitHub direct

# Remove
hermes skills remove github

# Enable/Disable without removing
hermes skills enable github-pr-workflow
hermes skills disable github-pr-workflow
```

---

## 70+ Bundled Skills

Skills are organized into categories. All ship with Hermes and are installed in `~/.hermes/skills/`.

### Apple / macOS (4 skills)

| Skill | Description |
|-------|-------------|
| `apple-notes` | Manage Apple Notes via memo CLI |
| `apple-reminders` | Manage Apple Reminders via remindctl |
| `findmy` | Track Apple devices and AirTags |
| `imessage` | Send and receive iMessages and SMS |

### Autonomous AI Agents (4 skills)

| Skill | Description |
|-------|-------------|
| `claude-code` | Delegate coding tasks to Claude Code CLI |
| `codex` | Delegate to OpenAI Codex CLI |
| `hermes-agent-spawning` | Spawn Hermes subprocesses |
| `opencode` | Delegate to OpenCode CLI |

### Creative (3 skills)

| Skill | Description |
|-------|-------------|
| `ascii-art` | 571 fonts (pyfiglet), cowsay, boxes, image-to-ascii |
| `ascii-video` | Full video-to-ASCII pipeline with animations |
| `excalidraw` | Hand-drawn diagrams in Excalidraw JSON format |

### Data Science & MLOps

#### MLOps / Cloud (2 skills)

| Skill | Description |
|-------|-------------|
| `lambda-labs-gpu-cloud` | GPU instance management |
| `modal-serverless-gpu` | Serverless GPU tasks |

#### Evaluation & Benchmarking (5 skills)

| Skill | Description |
|-------|-------------|
| `evaluating-llms-harness` | 60+ benchmarks: MMLU, HumanEval, etc. |
| `huggingface-tokenizers` | Fast tokenizers: BPE, WordPiece, Unigram |
| `nemo-curator` | GPU-accelerated data curation |
| `sparse-autoencoder-training` | SAE training (SAELens) |
| `weights-and-biases` | MLOps experiment tracking |

#### Model Inference & Serving (8 skills)

| Skill | Description |
|-------|-------------|
| `gguf-quantization` | llama.cpp quantization (2–8 bit) |
| `guidance` | Constrained output with regex/grammars |
| `instructor` | Structured data extraction with Pydantic |
| `llama-cpp` | CPU/Apple Silicon/GPU inference |
| `obliteratus` | Remove refusal behaviors from LLMs |
| `outlines` | Valid JSON/XML/code generation |
| `serving-llms-vllm` | vLLM high-throughput serving |
| `tensorrt-llm` | NVIDIA TensorRT LLM optimization |

#### Vision & Multimodal (6 skills)

| Skill | Description |
|-------|-------------|
| `audiocraft-audio-generation` | Text-to-music via MusicGen |
| `clip` | Zero-shot vision-language classification |
| `llava` | Large Language and Vision Assistant |
| `segment-anything-model` | Zero-shot image segmentation |
| `stable-diffusion-image-generation` | Text-to-image via Diffusers |
| `whisper` | Speech recognition (99 languages) |

#### Training & Fine-tuning (14 skills)

| Skill | Description |
|-------|-------------|
| `axolotl` | Fine-tuning: 100+ models, LoRA/QLoRA |
| `distributed-llm-pretraining-torchtitan` | PyTorch distributed pretraining |
| `fine-tuning-with-trl` | SFT/DPO/PPO/GRPO with TRL |
| `grpo-rl-training` | GRPO/RL fine-tuning |
| `hermes-atropos-environments` | RL environment development for Atropos |
| `huggingface-accelerate` | Simple distributed training |
| `optimizing-attention-flash` | Flash Attention optimization |
| `peft-fine-tuning` | LoRA/QLoRA parameter-efficient tuning |
| `pytorch-fsdp` | Fully Sharded Data Parallel |
| `pytorch-lightning` | High-level training framework |
| `simpo-training` | Simple Preference Optimization |
| `slime-rl-training` | Megatron + SGLang RL training |
| `unsloth` | Fast fine-tuning (2–5× speedup) |
| `dspy` | Stanford NLP declarative programming |

#### Vector Databases (4 skills)

| Skill | Description |
|-------|-------------|
| `chroma` | Open-source embedding database |
| `faiss` | Facebook's similarity search |
| `pinecone` | Managed vector database |
| `qdrant-vector-search` | Rust-powered vector search |

### Email (2 skills)

| Skill | Description |
|-------|-------------|
| `himalaya` | CLI email management (IMAP/SMTP) |

### Gaming (2 skills)

| Skill | Description |
|-------|-------------|
| `minecraft-modpack-server` | Modded Minecraft server setup |
| `pokemon-player` | Autonomous Pokemon gameplay via emulation |

### GitHub (6 skills)

| Skill | Description |
|-------|-------------|
| `codebase-inspection` | LOC counting, language breakdown |
| `github-auth` | GitHub authentication setup |
| `github-code-review` | Code review analysis |
| `github-issues` | Issue management |
| `github-pr-workflow` | Full PR lifecycle |
| `github-repo-management` | Repository management |

### Leisure (1 skill)

| Skill | Description |
|-------|-------------|
| `find-nearby` | Find nearby places using OpenStreetMap |

### MCP (2 skills)

| Skill | Description |
|-------|-------------|
| `mcporter` | mcporter CLI bridge for MCP servers |
| `native-mcp` | Built-in MCP client with automatic tool discovery |

### Media (4 skills)

| Skill | Description |
|-------|-------------|
| `gif-search` | Search/download GIFs from Tenor |
| `heartmula` | Music generation (Suno-like) |
| `songsee` | Spectrogram and audio visualization |
| `youtube-content` | YouTube transcripts and content extraction |

### Note-Taking (1 skill)

| Skill | Description |
|-------|-------------|
| `obsidian` | Read, search, and create Obsidian vault notes |

### Productivity (5 skills)

| Skill | Description |
|-------|-------------|
| `google-workspace` | Gmail, Calendar, Drive, Contacts, Sheets, Docs |
| `nano-pdf` | Edit PDFs with natural language |
| `notion` | Notion API for pages and databases |
| `ocr-and-documents` | PDF/document text extraction |
| `powerpoint` | PowerPoint presentation manipulation |

### Research (6 skills)

| Skill | Description |
|-------|-------------|
| `arxiv` | Academic paper search (no API key needed) |
| `blogwatcher` | Blog and RSS feed monitoring |
| `domain-intel` | Passive domain reconnaissance |
| `duckduckgo-search` | Free web search (no API key required) |
| `ml-paper-writing` | Publication-ready ML/AI paper guidance |
| `polymarket` | Prediction market data (read-only) |

### Smart Home (1 skill)

| Skill | Description |
|-------|-------------|
| `openhue` | Philips Hue light control |

### Software Development (7 skills)

| Skill | Description |
|-------|-------------|
| `code-review` | Code review guidelines and checklist |
| `plan` | Plan mode — write plans without executing |
| `requesting-code-review` | Systematic code review process |
| `subagent-driven-development` | Multi-task delegation with review |
| `systematic-debugging` | 4-phase root cause investigation |
| `test-driven-development` | RED-GREEN-REFACTOR cycle |
| `writing-plans` | Implementation planning |

### QA (1 skill)

| Skill | Description |
|-------|-------------|
| `dogfood` | Exploratory QA testing of web applications |

---

## Optional Skills (9 official)

These are not bundled by default. Install from the official source:

| Category | Skill | Description |
|----------|-------|-------------|
| Autonomous AI Agents | `blackbox` | Delegate to Blackbox AI CLI agent |
| Blockchain | `solana` | Solana blockchain queries: balances, pricing, NFTs |
| Email | `agentmail` | Agent-owned email via AgentMail |
| Health | `neuroskill-bci` | Real-time cognitive state from BCI wearable |
| Migration | `openclaw-migration` | Migrate from OpenClaw to Hermes |
| Research | `qmd` | Personal knowledge base search (BM25 + vector) |
| Security | `1password` | 1Password CLI integration |
| Security | `oss-forensics` | Open-source security forensics |

Install:
```bash
hermes skills install official/security/1password
hermes skills install official/migration/openclaw-migration
```

---

## Creating Skills

### Skill File Format (SKILL.md)

```yaml
---
name: my-skill
description: Brief one-line description (shown in search results)
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]      # Optional: restrict to specific OS
metadata:
  hermes:
    tags: [Category, Subcategory]
    related_skills: [other-skill-name]
    fallback_for_toolsets: [web]   # Show ONLY when web toolset unavailable
    requires_toolsets: [terminal]  # Show ONLY when terminal available
required_environment_variables:
  - name: MY_API_KEY
    prompt: "Enter your API key"
    help: "Get from https://example.com"
    required_for: "core functionality"
---

# Skill Title

Brief intro paragraph.

## When to Use

Explain trigger conditions — when the agent should activate this skill.

## Quick Reference

| Command | Description |
|---------|-------------|
| `tool --flag` | What it does |

## Procedure

Step-by-step instructions.

## Pitfalls

Known failure modes and how to handle them.

## Verification

How to confirm the skill worked correctly.
```

### Skill Directory Structure

```
~/.hermes/skills/
└── category/
    └── my-skill/
        ├── SKILL.md              # Required: main instructions
        ├── references/           # Additional documentation
        ├── templates/            # Output format templates
        ├── scripts/              # Helper scripts
        └── assets/               # Supplementary files
```

### Creating a Skill via Tool

The agent can create skills autonomously using `skill_manage`:

```python
# Create
skill_manage(action="create", name="my-skill", category="research",
             content="""---
name: my-skill
...
---
# My Skill
...""")

# Edit
skill_manage(action="patch", name="my-skill",
             old_string="old text", new_string="new text")

# Delete
skill_manage(action="delete", name="my-skill")

# Add auxiliary file
skill_manage(action="write_file", name="my-skill",
             file_path="templates/output.md",
             file_content="# Template\n...")
```

### Creating a Skill via CLI

```bash
hermes skills create my-new-skill
```

Launches an interactive prompt to name, describe, and populate the skill.

---

## Platform-Specific Skills

Skills can be restricted to specific platforms:

```yaml
platforms: [macos]            # macOS only
platforms: [macos, linux]     # macOS and Linux
platforms: [windows]          # Windows only (via WSL2)
```

Skills not matching the current platform are hidden from the system prompt, `skills_list`, and all slash commands.

---

## Conditional Activation

Skills can be conditionally shown based on available toolsets:

```yaml
metadata:
  hermes:
    # Show skill only when 'web' toolset is NOT available
    fallback_for_toolsets: [web]

    # Show skill only when 'terminal' toolset IS available
    requires_toolsets: [terminal]

    # Tool-level checks (individual tools instead of toolsets)
    fallback_for_tools: [web_search]
    requires_tools: [terminal]
```

This prevents irrelevant instructions from cluttering the context window.

---

## Skills Hub

Browse and install skills from multiple sources:

| Source | Description |
|--------|-------------|
| `official` | Hermes official optional skills (builtin trust) |
| `skills-sh` | skills.sh Vercel public directory |
| `well-known` | URL-based discovery |
| `github` | Direct GitHub repository |
| `community` | Community submissions |

**Security scanning:**

All Skills Hub installs are scanned for security issues:
- Trust levels: `builtin → official → trusted → community`
- `--force` overrides non-dangerous findings
- `dangerous` verdicts are always blocked

---

## Skills in the agentskills.io Ecosystem

Hermes skills follow the [agentskills.io](https://agentskills.io) standard, which is also used by:

- Claude Code (Anthropic)
- Cursor
- OpenAI Codex CLI
- Gemini CLI
- GitHub Copilot
- VS Code
- OpenHands
- Goose
- Junie (JetBrains)
- Roo Code
- Letta, Amp, and others

Skills authored for Hermes work in all compatible agents with no modification.
