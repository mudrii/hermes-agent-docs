# Skills

Skills are on-demand knowledge documents the agent can load when needed. They follow a **progressive disclosure** pattern to minimize token usage and are compatible with the [agentskills.io](https://agentskills.io/specification) open standard.

All skills live in **`~/.hermes/skills/`** — a single directory that serves as the source of truth. On fresh install, bundled skills are copied from the repository. Hub-installed and agent-created skills also live here. The agent can modify or delete any skill.

## Skills vs Tools

Skills and tools serve different purposes:

| | Skills | Tools |
|---|---|---|
| **What they are** | Markdown documents with instructions | Python functions exposed to the AI |
| **Format** | SKILL.md + optional supporting files | Python handler + JSON schema |
| **Execution** | Agent reads and follows instructions | Code executes directly |
| **Extension** | No code changes required | Requires editing Python files |
| **Best for** | CLI workflows, API guides, shell-based tasks | Binary data, streaming, complex Python logic |

Make it a **Skill** when the capability can be expressed as instructions plus shell commands plus existing tools (arXiv search, git workflows, Docker management, PDF processing, email via CLI tools).

Make it a **Tool** when it requires end-to-end integration with API keys, auth flows, multi-component configuration, custom processing logic that must execute precisely every time, binary data, streaming, or real-time events (browser automation, TTS, vision analysis).

## How Skills Are Loaded and Activated

Skills are discovered at startup by scanning `~/.hermes/skills/` recursively for all `SKILL.md` files. The scan excludes `.git`, `.github`, and `.hub` directories.

Skills are never loaded into context automatically in bulk. Instead, the agent uses three tools in a progressive disclosure pattern:

```
Level 0: skills_list()            -> [{name, description, category}, ...]   (~3k tokens)
Level 1: skill_view(name)         -> Full SKILL.md content + metadata        (varies)
Level 2: skill_view(name, path)   -> Specific reference/template/script file (varies)
```

At level 0, the agent receives only name and description — enough to decide whether a skill is relevant. Full content is loaded only on demand at level 1. Linked files (references, templates, scripts) are loaded individually at level 2. This keeps token usage low for sessions that do not need most skills.

A skill is filtered out and never shown to the agent if:
- Its `platforms` field excludes the current OS
- Its name appears in the `skills.disabled` config list for the active platform
- It cannot be parsed (YAML errors, encoding errors)

Skills without any platform or toolset restrictions are always shown.

### Conditional Activation

Skills can automatically show or hide themselves based on which tools or toolsets are available in the current session. This is used for fallback skills — free or local alternatives that only appear when a premium tool is unavailable.

```yaml
metadata:
  hermes:
    fallback_for_toolsets: [web]      # Show ONLY when these toolsets are unavailable
    requires_toolsets: [terminal]     # Show ONLY when these toolsets are available
    fallback_for_tools: [web_search]  # Show ONLY when these specific tools are unavailable
    requires_tools: [terminal]        # Show ONLY when these specific tools are available
```

| Field | Behavior |
|-------|----------|
| `fallback_for_toolsets` | Skill is hidden when the listed toolsets are available. Shown when they are missing. |
| `fallback_for_tools` | Same, but checks individual tools instead of toolsets. |
| `requires_toolsets` | Skill is hidden when the listed toolsets are unavailable. Shown when they are present. |
| `requires_tools` | Same, but checks individual tools. |

Example: The built-in `duckduckgo-search` skill uses `fallback_for_toolsets: [web]`. When `FIRECRAWL_API_KEY` is set, the web toolset is available and the agent uses `web_search` — the DuckDuckGo skill stays hidden. If the API key is missing, the skill automatically appears as a fallback.

## SKILL.md File Format

```markdown
---
name: my-skill
description: Brief description of what this skill does
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]     # Optional — restrict to specific OS platforms
                               #   Valid: macos, linux, windows
                               #   Omit to load on all platforms (default)
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: Tenor API key
    help: Get a key from https://developers.google.com/tenor
    required_for: full functionality
metadata:
  hermes:
    tags: [python, automation]
    category: devops
    related_skills: [other-skill-name]
    fallback_for_toolsets: [web]    # Optional — conditional activation
    requires_toolsets: [terminal]   # Optional — conditional activation
---

# Skill Title

Brief intro.

## When to Use
Trigger conditions — when should the agent load this skill?

## Quick Reference
Table of common commands or API calls.

## Procedure
Step-by-step instructions the agent follows.

## Pitfalls
Known failure modes and how to handle them.

## Verification
How the agent confirms it worked.
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Skill name, max 64 characters |
| `description` | Yes | Brief description, max 1024 characters |
| `version` | No | Semantic version string |
| `author` | No | Author name |
| `license` | No | License identifier (agentskills.io standard) |
| `platforms` | No | List of `macos`, `linux`, `windows`. Omit for all platforms. |
| `required_environment_variables` | No | List of required env vars with prompts |
| `metadata.hermes.tags` | No | List of keyword tags |
| `metadata.hermes.related_skills` | No | List of related skill names |
| `metadata.hermes.fallback_for_toolsets` | No | Show only when listed toolsets are absent |
| `metadata.hermes.requires_toolsets` | No | Show only when listed toolsets are present |
| `metadata.hermes.fallback_for_tools` | No | Show only when listed tools are absent |
| `metadata.hermes.requires_tools` | No | Show only when listed tools are present |
| `compatibility` | No | Compatibility note (agentskills.io standard) |
| `prerequisites.env_vars` | No | Legacy alias for `required_environment_variables` |

### Platform-Specific Skills

```yaml
platforms: [macos]            # macOS only (e.g., iMessage, Apple Reminders, FindMy)
platforms: [macos, linux]     # macOS and Linux
platforms: [windows]          # Windows only
```

When set, the skill is automatically hidden from the system prompt, `skills_list()`, and slash commands on incompatible platforms. The platform detection maps `macos` to `sys.platform.startswith("darwin")`, `linux` to `sys.platform.startswith("linux")`, and `windows` to `sys.platform.startswith("win32")`.

### Secure Setup on Load

Skills can declare required environment variables without disappearing from discovery:

```yaml
required_environment_variables:
  - name: TENOR_API_KEY
    prompt: Tenor API key
    help: Get a key from https://developers.google.com/tenor
    required_for: full functionality
```

When a missing value is encountered, Hermes prompts for it securely only when the skill is actually loaded in the local CLI. You can skip setup and keep using the skill. Messaging platforms never ask for secrets in chat — they tell you to use `hermes setup` or add the key to `~/.hermes/.env` manually.

The legacy `prerequisites.env_vars` field is supported as a backward-compatible alias.

## Skill Directory Structure

```text
~/.hermes/skills/                  # Single source of truth
├── mlops/                         # Category directory
│   ├── axolotl/
│   │   ├── SKILL.md               # Main instructions (required)
│   │   ├── references/            # Additional docs (API specs, examples)
│   │   ├── templates/             # Output formats, config boilerplate
│   │   ├── scripts/               # Helper scripts callable from the skill
│   │   └── assets/                # Supplementary files (agentskills.io standard)
│   └── vllm/
│       └── SKILL.md
├── devops/
│   └── deploy-k8s/                # Agent-created skill
│       ├── SKILL.md
│       └── references/
├── .hub/                          # Skills Hub state
│   ├── lock.json
│   ├── quarantine/
│   └── audit.log
└── .bundled_manifest              # Tracks seeded bundled skills
```

## Bundled Skills Catalog

Hermes ships with a large built-in skill library that is copied into `~/.hermes/skills/` on install. The repository organizes skills under `skills/` by category. The actual directory structure on disk is verified below.

### apple

Apple/macOS-specific skills. All four restrict themselves to `platforms: [macos]` and are hidden on Linux and Windows.

| Skill | Description | Path |
|-------|-------------|------|
| `apple-notes` | Manage Apple Notes via the memo CLI on macOS (create, view, search, edit). | `apple/apple-notes` |
| `apple-reminders` | Manage Apple Reminders via remindctl CLI (list, add, complete, delete). | `apple/apple-reminders` |
| `findmy` | Track Apple devices and AirTags via FindMy.app on macOS using AppleScript and screen capture. | `apple/findmy` |
| `imessage` | Send and receive iMessages/SMS via the imsg CLI on macOS. | `apple/imessage` |

### autonomous-ai-agents

Skills for spawning and orchestrating autonomous AI coding agents and multi-agent workflows.

| Skill | Description | Path |
|-------|-------------|------|
| `claude-code` | Delegate coding tasks to Claude Code (Anthropic's CLI agent). Requires the claude CLI. | `autonomous-ai-agents/claude-code` |
| `codex` | Delegate coding tasks to OpenAI Codex CLI agent. Requires the codex CLI and a git repository. | `autonomous-ai-agents/codex` |
| `hermes-agent-spawning` | Spawn additional Hermes Agent instances as autonomous subprocesses. Supports non-interactive one-shot mode (-q) and interactive PTY mode. | `autonomous-ai-agents/hermes-agent` |
| `opencode` | Delegate coding tasks to OpenCode CLI agent. Requires the opencode CLI. | `autonomous-ai-agents/opencode` |

### creative

| Skill | Description | Path |
|-------|-------------|------|
| `ascii-art` | Generate ASCII art using pyfiglet (571 fonts), cowsay, boxes, toilet, image-to-ascii, remote APIs, and LLM fallback. No API keys required. | `creative/ascii-art` |
| `ascii-video` | Production pipeline for ASCII art video — converts video/audio/images/generative input into colored ASCII character video output (MP4, GIF, image sequence). | `creative/ascii-video` |
| `excalidraw` | Create hand-drawn style diagrams using Excalidraw JSON format (.excalidraw files) for architecture diagrams, flowcharts, sequence diagrams, and concept maps. | `creative/excalidraw` |

### data-science

| Skill | Description | Path |
|-------|-------------|------|
| `jupyter-live-kernel` | Work with Jupyter notebooks using a live kernel for interactive data science. | `data-science/jupyter-live-kernel` |

### dogfood

| Skill | Description | Path |
|-------|-------------|------|
| `dogfood` | Systematic exploratory QA testing of web applications — find bugs, capture evidence, and generate structured reports. | `dogfood` |
| `hermes-agent-setup` | Help users configure Hermes Agent — CLI usage, setup wizard, model/provider selection, tools, skills, voice/STT/TTS, gateway, and troubleshooting. | `dogfood/hermes-agent-setup` |

### email

| Skill | Description | Path |
|-------|-------------|------|
| `himalaya` | CLI to manage emails via IMAP/SMTP. List, read, write, reply, forward, search, and organize emails from the terminal. Supports multiple accounts and MML composition. | `email/himalaya` |

### gaming

| Skill | Description | Path |
|-------|-------------|------|
| `minecraft-modpack-server` | Set up a modded Minecraft server from a CurseForge/Modrinth server pack zip. Covers NeoForge/Forge install, Java version, JVM tuning, firewall, backups, and launch scripts. | `gaming/minecraft-modpack-server` |
| `pokemon-player` | Play Pokemon games autonomously via headless emulation. Reads structured game state from RAM, makes strategic decisions, and sends button inputs. | `gaming/pokemon-player` |

### github

GitHub workflow skills for managing repositories, pull requests, code reviews, issues, and CI/CD pipelines using the gh CLI and git via terminal.

| Skill | Description | Path |
|-------|-------------|------|
| `codebase-inspection` | Inspect and analyze codebases using pygount for LOC counting, language breakdown, and code-vs-comment ratios. | `github/codebase-inspection` |
| `github-auth` | Set up GitHub authentication using git or the gh CLI. Covers HTTPS tokens, SSH keys, credential helpers, and gh auth. | `github/github-auth` |
| `github-code-review` | Review code changes by analyzing git diffs, leaving inline comments on PRs, and performing pre-push review. | `github/github-code-review` |
| `github-issues` | Create, manage, triage, and close GitHub issues. Search, add labels, assign people, and link to PRs. | `github/github-issues` |
| `github-pr-workflow` | Full pull request lifecycle — create branches, commit changes, open PRs, monitor CI, auto-fix failures, and merge. | `github/github-pr-workflow` |
| `github-repo-management` | Clone, create, fork, configure, and manage GitHub repositories. Manage remotes, secrets, releases, and workflows. | `github/github-repo-management` |

### inference-sh

| Skill | Description | Path |
|-------|-------------|------|
| `inference-sh-cli` | Run 150+ AI apps via inference.sh CLI (infsh) — image generation, video creation, LLMs, search, 3D, social automation. No GPU required. | `inference-sh/cli` |

### leisure

| Skill | Description | Path |
|-------|-------------|------|
| `find-nearby` | Find nearby places (restaurants, cafes, bars, pharmacies) using OpenStreetMap. Works with coordinates, addresses, cities, zip codes, or Telegram location pins. No API keys needed. | `leisure/find-nearby` |

### mcp

| Skill | Description | Path |
|-------|-------------|------|
| `mcporter` | Use the mcporter CLI to list, configure, auth, and call MCP servers/tools directly (HTTP or stdio), including ad-hoc servers, config edits, and CLI/type generation. | `mcp/mcporter` |
| `native-mcp` | Built-in MCP client documentation — explains configuring MCP servers in config.yaml for automatic tool discovery. Supports stdio and HTTP transports with automatic reconnection. | `mcp/native-mcp` |

### media

| Skill | Description | Path |
|-------|-------------|------|
| `gif-search` | Search and download GIFs from Tenor using curl. No dependencies beyond curl and jq. | `media/gif-search` |
| `heartmula` | Set up and run HeartMuLa, the open-source music generation model family. Generates full songs from lyrics and tags with multilingual support. | `media/heartmula` |
| `songsee` | Generate spectrograms and audio feature visualizations (mel, chroma, MFCC, tempogram) from audio files via CLI. | `media/songsee` |
| `youtube-content` | Fetch YouTube video transcripts and transform them into structured content (chapters, summaries, threads, blog posts). | `media/youtube-content` |

### mlops

| Skill | Description | Path |
|-------|-------------|------|
| `huggingface-hub` | Hugging Face Hub CLI (hf) — search, download, and upload models and datasets, manage repos, query datasets with SQL, deploy inference endpoints, manage Spaces and buckets. | `mlops/huggingface-hub` |

### mlops/cloud

GPU cloud providers and serverless compute platforms for ML workloads.

| Skill | Description | Path |
|-------|-------------|------|
| `lambda-labs-gpu-cloud` | Reserved and on-demand GPU cloud instances for ML training and inference via simple SSH access. | `mlops/cloud/lambda-labs` |
| `modal-serverless-gpu` | Serverless GPU cloud platform for running ML workloads on-demand with automatic scaling. | `mlops/cloud/modal` |

### mlops/evaluation

Model evaluation benchmarks, experiment tracking, data curation, tokenizers, and interpretability tools.

| Skill | Description | Path |
|-------|-------------|------|
| `evaluating-llms-harness` | Evaluate LLMs across 60+ academic benchmarks (MMLU, HumanEval, GSM8K, TruthfulQA, HellaSwag). Industry standard used by EleutherAI and HuggingFace. | `mlops/evaluation/lm-evaluation-harness` |
| `huggingface-tokenizers` | Fast Rust-based tokenizers. Trains custom vocabularies, handles BPE/WordPiece/Unigram, tokenizes 1GB in under 20 seconds. | `mlops/evaluation/huggingface-tokenizers` |
| `nemo-curator` | GPU-accelerated data curation for LLM training — fuzzy deduplication (16x faster), quality filtering (30+ heuristics), semantic dedup, PII redaction. | `mlops/evaluation/nemo-curator` |
| `sparse-autoencoder-training` | Train and analyze Sparse Autoencoders (SAEs) using SAELens to decompose neural network activations into interpretable features. | `mlops/evaluation/saelens` |
| `weights-and-biases` | Track ML experiments with automatic logging, visualize training in real-time, and optimize hyperparameters with sweeps. | `mlops/evaluation/weights-and-biases` |

### mlops/inference

Model serving, quantization (GGUF/GPTQ), structured output, inference optimization, and model surgery tools for deploying and running LLMs.

| Skill | Description | Path |
|-------|-------------|------|
| `gguf-quantization` | GGUF format and llama.cpp quantization for CPU/GPU inference. 2-8 bit quantization without GPU requirements. | `mlops/inference/gguf` |
| `guidance` | Control LLM output with regex and grammars, guarantee valid JSON/XML/code generation with Microsoft Guidance. | `mlops/inference/guidance` |
| `instructor` | Extract structured data from LLM responses with Pydantic validation, retry failed extractions, parse complex JSON with type safety. | `mlops/inference/instructor` |
| `llama-cpp` | Run LLM inference on CPU, Apple Silicon (M1/M2/M3), and consumer GPUs without NVIDIA hardware. Supports GGUF quantization (1.5-8 bit). | `mlops/inference/llama-cpp` |
| `obliteratus` | Remove refusal behaviors from open-weight LLMs using mechanistic interpretability techniques (diff-in-means, SVD, LEACE, SAE decomposition). 9 CLI methods, 28 analysis modules, 116 model presets. | `mlops/inference/obliteratus` |
| `outlines` | Guarantee valid JSON/XML/code structure during generation using Pydantic models for type-safe outputs. Supports local models (Transformers, vLLM). | `mlops/inference/outlines` |
| `serving-llms-vllm` | Serve LLMs with high throughput using vLLM's PagedAttention and continuous batching. OpenAI-compatible endpoints, GPTQ/AWQ/FP8 quantization. | `mlops/inference/vllm` |
| `tensorrt-llm` | Optimize LLM inference with NVIDIA TensorRT for maximum throughput on A100/H100 GPUs. In-flight batching, FP8/INT4 quantization. | `mlops/inference/tensorrt-llm` |

### mlops/models

Specific model architectures — computer vision, speech, audio generation, and multimodal models.

| Skill | Description | Path |
|-------|-------------|------|
| `audiocraft-audio-generation` | PyTorch library for audio generation including text-to-music (MusicGen) and text-to-sound (AudioGen). | `mlops/models/audiocraft` |
| `clip` | OpenAI's model connecting vision and language. Zero-shot image classification, image-text matching, cross-modal retrieval. Trained on 400M image-text pairs. | `mlops/models/clip` |
| `llava` | Large Language and Vision Assistant — visual instruction tuning and image-based conversations combining CLIP and LLaMA. | `mlops/models/llava` |
| `segment-anything-model` | Foundation model for image segmentation with zero-shot transfer. Segment any object using points, boxes, or masks as prompts. | `mlops/models/segment-anything` |
| `stable-diffusion-image-generation` | Text-to-image generation with Stable Diffusion via HuggingFace Diffusers. Supports img2img, inpainting, and custom pipelines. | `mlops/models/stable-diffusion` |
| `whisper` | OpenAI's general-purpose speech recognition. 99 languages, transcription, translation to English. Six model sizes from tiny (39M) to large (1550M). | `mlops/models/whisper` |

### mlops/research

| Skill | Description | Path |
|-------|-------------|------|
| `dspy` | Build complex AI systems with declarative programming, optimize prompts automatically, create modular RAG systems with Stanford NLP's DSPy. | `mlops/research/dspy` |

### mlops/training

Fine-tuning, RLHF/DPO/GRPO training, distributed training frameworks, and optimization tools for training LLMs.

| Skill | Description | Path |
|-------|-------------|------|
| `axolotl` | Expert guidance for fine-tuning LLMs with Axolotl — YAML configs, 100+ models, LoRA/QLoRA, DPO/KTO/ORPO/GRPO, multimodal support. | `mlops/training/axolotl` |
| `distributed-llm-pretraining-torchtitan` | PyTorch-native distributed LLM pretraining using torchtitan with 4D parallelism (FSDP2, TP, PP, CP). Supports Llama 3.1, DeepSeek V3 at scale. | `mlops/training/torchtitan` |
| `fine-tuning-with-trl` | Fine-tune LLMs with TRL — SFT for instruction tuning, DPO for preference alignment, PPO/GRPO for reward optimization. | `mlops/training/trl-fine-tuning` |
| `grpo-rl-training` | Expert guidance for GRPO/RL fine-tuning with TRL for reasoning and task-specific model training. | `mlops/training/grpo-rl-training` |
| `hermes-atropos-environments` | Build, test, and debug Hermes Agent RL environments for Atropos training — HermesAgentBaseEnv interface, reward functions, wandb logging. | `mlops/training/hermes-atropos-environments` |
| `huggingface-accelerate` | Distributed training API — 4 lines to add distributed support to any PyTorch script. Unified DeepSpeed/FSDP/Megatron/DDP API. | `mlops/training/accelerate` |
| `optimizing-attention-flash` | Optimize transformer attention with Flash Attention for 2-4x speedup and 10-20x memory reduction. | `mlops/training/flash-attention` |
| `peft-fine-tuning` | Parameter-efficient fine-tuning using LoRA, QLoRA, and 25+ methods. Train less than 1% of parameters for 7B-70B models. | `mlops/training/peft` |
| `pytorch-fsdp` | Expert guidance for Fully Sharded Data Parallel training with PyTorch FSDP — parameter sharding, mixed precision, CPU offloading. | `mlops/training/pytorch-fsdp` |
| `pytorch-lightning` | High-level PyTorch framework with Trainer class, automatic distributed training (DDP/FSDP/DeepSpeed), and callbacks system. | `mlops/training/pytorch-lightning` |
| `simpo-training` | Simple Preference Optimization — reference-free DPO alternative with better performance (+6.4 points on AlpacaEval 2.0). | `mlops/training/simpo` |
| `slime-rl-training` | LLM post-training with RL using slime, a Megatron+SGLang framework for GLM models. | `mlops/training/slime` |
| `unsloth` | Expert guidance for fast fine-tuning with Unsloth — 2-5x faster training, 50-80% less memory, LoRA/QLoRA optimization. | `mlops/training/unsloth` |

### mlops/vector-databases

Vector similarity search and embedding databases for RAG, semantic search, and AI application backends.

| Skill | Description | Path |
|-------|-------------|------|
| `chroma` | Open-source embedding database — vector and full-text search with metadata filtering. Scales from notebooks to production clusters. | `mlops/vector-databases/chroma` |
| `faiss` | Facebook's library for efficient similarity search of dense vectors. Billions of vectors, GPU acceleration, multiple index types (Flat, IVF, HNSW). | `mlops/vector-databases/faiss` |
| `pinecone` | Managed vector database with hybrid search (dense + sparse), metadata filtering, namespaces, and auto-scaling. Under 100ms p95 latency. | `mlops/vector-databases/pinecone` |
| `qdrant-vector-search` | High-performance vector similarity search engine built in Rust for RAG and semantic search. | `mlops/vector-databases/qdrant` |

### note-taking

| Skill | Description | Path |
|-------|-------------|------|
| `obsidian` | Read, search, and create notes in the Obsidian vault. | `note-taking/obsidian` |

### productivity

| Skill | Description | Path |
|-------|-------------|------|
| `google-workspace` | Gmail, Calendar, Drive, Contacts, Sheets, and Docs integration via Python with OAuth2 and automatic token refresh. | `productivity/google-workspace` |
| `nano-pdf` | Edit PDFs with natural-language instructions using the nano-pdf CLI — modify text, fix typos, update titles. | `productivity/nano-pdf` |
| `notion` | Notion API for creating and managing pages, databases, and blocks via curl. Search, create, update, and query workspaces. | `productivity/notion` |
| `linear` | Manage Linear issues, projects, and teams via the GraphQL API. Create, update, search, and organize issues. All operations via curl. | `productivity/linear` |
| `ocr-and-documents` | Extract text from PDFs and scanned documents using pymupdf, marker-pdf, and web_extract. | `productivity/ocr-and-documents` |
| `powerpoint` | Work with .pptx files — create slide decks, read/parse/extract text, edit content using python-pptx. | `productivity/powerpoint` |

### research

| Skill | Description | Path |
|-------|-------------|------|
| `arxiv` | Search and retrieve academic papers from arXiv using their free REST API. No API key needed. Search by keyword, author, category, or ID. | `research/arxiv` |
| `blogwatcher` | Monitor blogs and RSS/Atom feeds for updates using the blogwatcher CLI. | `research/blogwatcher` |
| `domain-intel` | Passive domain reconnaissance — subdomain discovery, SSL certificate inspection, WHOIS lookups, DNS records, domain availability checks. No API keys required. | `research/domain-intel` |
| `duckduckgo-search` | Free web search via DuckDuckGo — text, news, images, videos. No API key needed. Uses `fallback_for_toolsets: [web]` so it only appears when the web toolset is unavailable. | `research/duckduckgo-search` |
| `ml-paper-writing` | Write publication-ready ML/AI papers for NeurIPS, ICML, ICLR, ACL, AAAI, COLM. Includes LaTeX templates and reviewer guidelines. | `research/ml-paper-writing` |
| `parallel-cli` | Parallel CLI agent-native web search, extraction, deep research, enrichment, FindAll, and monitoring. JSON output and non-interactive flows. | `research/parallel-cli` |
| `polymarket` | Query Polymarket prediction market data — search markets, prices, orderbooks, and price history. Read-only via public REST APIs. | `research/polymarket` |

### smart-home

| Skill | Description | Path |
|-------|-------------|------|
| `openhue` | Control Philips Hue lights, rooms, and scenes via the OpenHue CLI. Turn lights on/off, adjust brightness, color, and color temperature. | `smart-home/openhue` |

### social-media

| Skill | Description | Path |
|-------|-------------|------|
| `xitter` | Interact with X/Twitter via the x-cli terminal client using official X API credentials. Post, read timelines, search tweets, like, retweet, bookmarks, mentions, and user lookups. | `social-media/xitter` |

### software-development

| Skill | Description | Path |
|-------|-------------|------|
| `code-review` | Guidelines for performing thorough code reviews with security and quality focus. | `software-development/code-review` |
| `plan` | Plan mode for Hermes — write a markdown implementation plan into `.hermes/plans/` without executing the work. | `software-development/plan` |
| `requesting-code-review` | Validates work meets requirements through a systematic review process. Use when completing tasks, implementing major features, or before merging. | `software-development/requesting-code-review` |
| `subagent-driven-development` | Execute implementation plans with independent tasks via `delegate_task` with two-stage review (spec compliance then code quality). | `software-development/subagent-driven-development` |
| `systematic-debugging` | 4-phase root cause investigation — no fixes without understanding the problem first. Use when encountering any bug or unexpected behavior. | `software-development/systematic-debugging` |
| `test-driven-development` | Enforces RED-GREEN-REFACTOR cycle with test-first approach. Use before writing any implementation code. | `software-development/test-driven-development` |
| `writing-plans` | Creates comprehensive implementation plans with bite-sized tasks, exact file paths, and complete code examples. | `software-development/writing-plans` |

## Optional Skills Catalog

Official optional skills live in `optional-skills/` in the repository. They are not bundled by default but are maintained by the Hermes team and install with the same trust level as bundled skills (no third-party warning, builtin trust).

Install with: `hermes skills install official/<category>/<skill>`
Browse all: `hermes skills browse --source official`

### autonomous-ai-agents

| Skill | Description | Path |
|-------|-------------|------|
| `blackbox` | Delegate coding tasks to Blackbox AI CLI agent. Multi-model with built-in judge that runs tasks through multiple LLMs and picks the best result. Requires the blackbox CLI and API key. | `autonomous-ai-agents/blackbox` |

### blockchain

| Skill | Description | Path |
|-------|-------------|------|
| `base` | Query Base (Ethereum L2) blockchain data with USD pricing — wallet balances, token info, transaction details, gas analysis, contract inspection, whale detection, and live network stats. Uses Base RPC + CoinGecko. No API key required. | `blockchain/base` |
| `solana` | Query Solana blockchain data with USD pricing — wallet balances, token portfolios, transaction details, NFTs, whale detection, and live network stats. Uses Solana RPC + CoinGecko. No API key required. | `blockchain/solana` |

### creative

| Skill | Description | Path |
|-------|-------------|------|
| `blender-mcp` | Control Blender directly from Hermes via socket connection to the blender-mcp addon. Create 3D objects, materials, animations, and run arbitrary Blender Python (bpy) code. Requires Blender 4.3+. | `creative/blender-mcp` |

### email

| Skill | Description | Path |
|-------|-------------|------|
| `agentmail` | Give the agent its own dedicated email inbox via AgentMail. Send, receive, and manage email autonomously using agent-owned addresses (e.g. hermes-agent@agentmail.to). | `email/agentmail` |

### health

| Skill | Description | Path |
|-------|-------------|------|
| `neuroskill-bci` | Incorporate real-time cognitive and emotional state from a BCI wearable (Muse 2/S or OpenBCI) into responses. Provides focus, relaxation, mood, cognitive load, drowsiness, heart rate, HRV, and 40+ derived EXG scores. | `health/neuroskill-bci` |

### migration

| Skill | Description | Path |
|-------|-------------|------|
| `openclaw-migration` | Migrate OpenClaw customization footprint (memories, SOUL.md, command allowlists, user skills, workspace assets from ~/.openclaw) into Hermes Agent. | `migration/openclaw-migration` |

### productivity

| Skill | Description | Path |
|-------|-------------|------|
| `telephony` | Give Hermes phone capabilities without core tool changes. Provision and persist a Twilio number, send and receive SMS/MMS, make direct calls, and place AI-driven outbound calls through Bland.ai or Vapi. | `productivity/telephony` |

### research

| Skill | Description | Path |
|-------|-------------|------|
| `qmd` | Search personal knowledge bases, notes, docs, and meeting transcripts locally using qmd — a hybrid retrieval engine with BM25, vector search, and LLM reranking. Supports CLI and MCP integration. | `research/qmd` |

### security

| Skill | Description | Path |
|-------|-------------|------|
| `1password` | Set up and use 1Password CLI (op). Install the CLI, enable desktop app integration, sign in, and read/inject secrets for commands. | `security/1password` |
| `oss-forensics` | Supply chain investigation, evidence recovery, and forensic analysis for GitHub repositories. Covers deleted commit recovery, force-push detection, IOC extraction, multi-source evidence collection, and structured forensic reporting. | `security/oss-forensics` |
| `sherlock` | OSINT username search across 400+ social networks. Hunt down social media accounts by username. Requires the sherlock CLI. | `security/sherlock` |

## Per-Platform Skill Enable/Disable

Skills can be enabled or disabled per platform in `~/.hermes/config.yaml`:

```yaml
skills:
  disabled:
    - duckduckgo-search     # Disabled globally on all platforms
  platform_disabled:
    telegram:
      - apple-notes         # Disabled only when HERMES_PLATFORM=telegram
    discord:
      - imessage
```

The `platform_disabled` key overrides the global `disabled` list for the named platform. The platform name is resolved from the `HERMES_PLATFORM` environment variable at runtime. If `platform_disabled` is set for the current platform, it completely replaces the global `disabled` list for that platform.

## Agent-Managed Skills

The agent can create, update, and delete its own skills via the `skill_manage` tool. This is the agent's procedural memory — when it figures out a non-trivial workflow, it saves the approach as a skill for future reuse.

### When the Agent Creates Skills

- After completing a complex task (5+ tool calls) successfully
- When it encountered errors or dead ends and found the working path
- When the user corrected its approach
- When it discovered a non-trivial workflow

### Actions

| Action | Use for | Key parameters |
|--------|---------|----------------|
| `create` | New skill from scratch | `name`, `content` (full SKILL.md), optional `category` |
| `patch` | Targeted fixes (preferred for updates) | `name`, `old_string`, `new_string` |
| `edit` | Major structural rewrites | `name`, `content` (full SKILL.md replacement) |
| `delete` | Remove a skill entirely | `name` |
| `write_file` | Add/update supporting files | `name`, `file_path`, `file_content` |
| `remove_file` | Remove a supporting file | `name`, `file_path` |

The `patch` action is preferred for updates — it is more token-efficient than `edit` because only the changed text appears in the tool call.

## How to Create a Custom Skill

### Step 1: Choose the right location

| Where | When to use |
|-------|-------------|
| `~/.hermes/skills/<category>/<name>/SKILL.md` | Personal skill, runs on your machine only |
| `skills/<category>/<name>/SKILL.md` in the repo | Bundled skill contributed to the repository |
| `optional-skills/<category>/<name>/SKILL.md` in the repo | Official optional skill contributed to the repository |

### Step 2: Create the SKILL.md file

Write a SKILL.md following the format specification above. Put the most common workflow first. Edge cases go at the bottom. Include helper scripts in `scripts/` for complex XML/JSON parsing rather than expecting the LLM to write parsers inline every time.

### Step 3: Add supporting files (optional)

```text
~/.hermes/skills/devops/my-workflow/
├── SKILL.md
├── references/
│   └── api-docs.md
├── templates/
│   └── deploy-config.yaml
└── scripts/
    └── validate-deployment.sh
```

### Step 4: Test

```bash
hermes chat --toolsets skills -q "Use the my-workflow skill to deploy"
```

### Step 5: Publish (optional)

```bash
# Publish to GitHub
hermes skills publish skills/my-workflow --to github --repo owner/repo

# Add a custom tap so others can install from your repo
hermes skills tap add owner/skills-repo
```

## Skills Hub

Browse, search, install, and manage skills from online registries, skills.sh, direct well-known skill endpoints, and official optional skills.

### Common Commands

```bash
hermes skills browse                              # Browse all hub skills (official first)
hermes skills browse --source official            # Browse only official optional skills
hermes skills search kubernetes                   # Search all sources
hermes skills search react --source skills-sh     # Search the skills.sh directory
hermes skills search https://mintlify.com/docs --source well-known
hermes skills inspect openai/skills/k8s           # Preview before installing
hermes skills install openai/skills/k8s           # Install with security scan
hermes skills install official/security/1password
hermes skills install skills-sh/vercel-labs/json-render/json-render-react --force
hermes skills install well-known:https://mintlify.com/docs/.well-known/skills/mintlify
hermes skills list --source hub                   # List hub-installed skills
hermes skills check                               # Check installed hub skills for upstream updates
hermes skills update                              # Reinstall hub skills with upstream changes
hermes skills audit                               # Re-scan all hub skills for security
hermes skills uninstall k8s                       # Remove a hub skill
hermes skills publish skills/my-skill --to github --repo owner/repo
hermes skills snapshot export setup.json          # Export skill config
hermes skills tap add myorg/skills-repo           # Add a custom GitHub source
```

All the same commands work as slash commands inside chat: `/skills browse`, `/skills search react --source skills-sh`, `/skills install openai/skills/skill-creator --force`, and so on.

### Supported Hub Sources

| Source | Example | Notes |
|--------|---------|-------|
| `official` | `official/security/1password` | Optional skills shipped with Hermes. Builtin trust level. |
| `skills-sh` | `skills-sh/vercel-labs/json-render/json-render-react` | Vercel's public skills directory at [skills.sh](https://skills.sh/). |
| `well-known` | `well-known:https://mintlify.com/docs/.well-known/skills/mintlify` | URL-based discovery from sites publishing `/.well-known/skills/index.json`. |
| `github` | `openai/skills/k8s` | Direct GitHub repo/path installs and custom taps. |
| `clawhub` | Source-specific identifiers | Third-party skills marketplace at [clawhub.ai](https://clawhub.ai/). |
| `claude-marketplace` | Source-specific identifiers | Marketplace repos publishing Claude-compatible manifests, including `anthropics/skills` and `aiskillstore/marketplace`. |
| `lobehub` | Source-specific identifiers | Agent entries from LobeHub's catalog at [lobehub.com](https://lobehub.com/), converted into installable Hermes skills. |

### Trust Levels

| Level | Source | Policy |
|-------|--------|--------|
| `builtin` | Ships with Hermes | Always trusted |
| `official` | `optional-skills/` in the repo | Builtin trust, no third-party warning |
| `trusted` | `openai/skills`, `anthropics/skills` | More permissive policy than community sources |
| `community` | Everything else — skills.sh, well-known endpoints, custom GitHub repos, most marketplaces | Non-dangerous findings can be overridden with `--force`; `dangerous` verdicts stay blocked |

### Security Scanning

All hub-installed skills go through a security scanner that checks for:
- Data exfiltration patterns
- Prompt injection attempts
- Destructive commands
- Shell injection
- Supply-chain signals

`hermes skills inspect` surfaces upstream metadata when available: repo URL, skills.sh detail page URL, install command, weekly installs, upstream security audit statuses, and well-known index/endpoint URLs.

`--force` can override policy blocks for caution/warn-style findings. `--force` does not override a `dangerous` scan verdict. Official optional skills (`official/...`) are treated as builtin trust and do not show the third-party warning panel.

### Update Lifecycle

```bash
hermes skills check            # Report which installed hub skills changed upstream
hermes skills update           # Reinstall only the skills with updates available
hermes skills update react     # Update one specific installed hub skill
```

The hub tracks provenance (source identifier plus upstream bundle content hash) to detect drift from the installed version.

## Using Skills in Practice

Every installed skill is automatically available as a slash command:

```bash
/gif-search funny cats
/axolotl help me fine-tune Llama 3 on my dataset
/github-pr-workflow create a PR for the auth refactor
/plan design a rollout for migrating our auth provider
/excalidraw
```

You can also interact with skills through natural conversation:

```bash
hermes chat --toolsets skills -q "What skills do you have?"
hermes chat --toolsets skills -q "Show me the axolotl skill"
```

The `plan` skill demonstrates custom slash command behavior. Running `/plan [request]` tells Hermes to inspect context if needed, write a markdown implementation plan instead of executing the task, and save the result under `.hermes/plans/` relative to the active workspace/backend working directory.

## Prerequisite Validation

When `skill_view` loads a skill, it checks each entry in `required_environment_variables` against the current environment:

1. The skill's `required_environment_variables` list is normalized from both the new format and the legacy `prerequisites.env_vars` field.
2. Each required variable name is checked against the `.env` snapshot loaded from `~/.hermes/.env` and the current process environment.
3. If variables are missing and the session is a local CLI session, Hermes calls the registered secret capture callback to prompt the user interactively.
4. On remote terminal backends (`docker`, `singularity`, `modal`, `ssh`, `daytona`), all required variables are always reported as needed because the agent cannot verify the remote environment.
5. On gateway/messaging surfaces, Hermes does not prompt for secrets in-band — it returns a hint to use `hermes setup` or `~/.hermes/.env` locally.
6. The skill view response includes `readiness_status: "available"` or `"setup_needed"`, `missing_required_environment_variables`, and an optional `setup_note` explaining what needs to be configured.
7. Path traversal (`..`) is blocked when loading linked files within a skill directory. Files are verified to be within the skill's own directory boundary before reading.
