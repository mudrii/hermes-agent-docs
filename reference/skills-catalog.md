# Bundled Skills Catalog

Hermes ships with a large built-in skill library copied into `~/.hermes/skills/` on install. This page catalogs the bundled skills that live in the repository under `skills/`.

## apple

Apple/macOS-specific skills -- iMessage, Reminders, Notes, FindMy, and macOS automation. These skills only load on macOS systems.

| Skill | Description | Path |
|-------|-------------|------|
| `apple-notes` | Manage Apple Notes via the memo CLI on macOS (create, view, search, edit). | `apple/apple-notes` |
| `apple-reminders` | Manage Apple Reminders via remindctl CLI (list, add, complete, delete). | `apple/apple-reminders` |
| `findmy` | Track Apple devices and AirTags via FindMy.app on macOS using AppleScript and screen capture. | `apple/findmy` |
| `imessage` | Send and receive iMessages/SMS via the imsg CLI on macOS. | `apple/imessage` |

## autonomous-ai-agents

Skills for spawning and orchestrating autonomous AI coding agents and multi-agent workflows -- running independent agent processes, delegating tasks, and coordinating parallel workstreams.

| Skill | Description | Path |
|-------|-------------|------|
| `claude-code` | Delegate coding tasks to Claude Code (Anthropic's CLI agent). Use for building features, refactoring, PR reviews, and iterative coding. Requires the claude CLI installed. | `autonomous-ai-agents/claude-code` |
| `codex` | Delegate coding tasks to OpenAI Codex CLI agent. Use for building features, refactoring, PR reviews, and batch issue fixing. Requires the codex CLI and a git repository. | `autonomous-ai-agents/codex` |
| `hermes-agent-spawning` | Spawn additional Hermes Agent instances as autonomous subprocesses for independent long-running tasks. Supports non-interactive one-shot mode (-q) and interactive PTY mode for multi-turn collaboration. Different from delegate_task -- this runs a full separate hermes process. | `autonomous-ai-agents/hermes-agent` |
| `opencode` | Delegate coding tasks to OpenCode CLI agent for feature implementation, refactoring, PR review, and long-running autonomous sessions. Requires the opencode CLI installed and authenticated. | `autonomous-ai-agents/opencode` |

## data-science

Skills for data science workflows -- interactive exploration, Jupyter notebooks, data analysis, and visualization.

| Skill | Description | Path |
|-------|-------------|------|
| `jupyter-live-kernel` | Use a live Jupyter kernel for stateful, iterative Python execution via hamelnb. Load this skill when the task involves exploration, iteration, or inspecting intermediate results. | `data-science/jupyter-live-kernel` |

## creative

Creative content generation -- ASCII art, hand-drawn style diagrams, visual design, and music tools.

| Skill | Description | Path |
|-------|-------------|------|
| `ascii-art` | Generate ASCII art using pyfiglet (571 fonts), cowsay, boxes, toilet, image-to-ascii, remote APIs, and LLM fallback. No API keys required. | `creative/ascii-art` |
| `ascii-video` | Production pipeline for ASCII art video -- any format. Converts video/audio/images/generative input into colored ASCII character video output (MP4, GIF, image sequence). | `creative/ascii-video` |
| `excalidraw` | Create hand-drawn style diagrams using Excalidraw JSON format. Generate .excalidraw files for architecture diagrams, flowcharts, sequence diagrams, concept maps, and more. | `creative/excalidraw` |
| `songwriting-and-ai-music` | Songwriting craft and AI music generation -- chord progressions, lyric structure, song sections, and prompting for Suno and Udio. | `creative/songwriting-and-ai-music` |

## devops

DevOps and infrastructure automation skills.

| Skill | Description | Path |
|-------|-------------|------|
| `webhook-subscriptions` | Create and manage webhook subscriptions for event-driven agent activation. External services (GitHub, Stripe, CI/CD, IoT) POST events to trigger agent runs. Requires webhook platform to be enabled. | `devops/webhook-subscriptions` |

## dogfood

| Skill | Description | Path |
|-------|-------------|------|
| `dogfood` | Systematic exploratory QA testing of web applications -- find bugs, capture evidence, and generate structured reports. | `dogfood/dogfood` |
| `hermes-agent-setup` | Help users configure Hermes Agent -- CLI usage, setup wizard, model/provider selection, tools, skills, voice/STT/TTS, gateway, and troubleshooting. | `dogfood/hermes-agent-setup` |

## email

Skills for sending, receiving, searching, and managing email from the terminal.

| Skill | Description | Path |
|-------|-------------|------|
| `himalaya` | CLI to manage emails via IMAP/SMTP. Use himalaya to list, read, write, reply, forward, search, and organize emails from the terminal. Supports multiple accounts. | `email/himalaya` |

## gaming

Skills for setting up, configuring, and managing game servers, modpacks, and gaming-related infrastructure.

| Skill | Description | Path |
|-------|-------------|------|
| `minecraft-modpack-server` | Set up a modded Minecraft server from a CurseForge/Modrinth server pack zip. Covers NeoForge/Forge install, Java version, JVM tuning, firewall, LAN config, backups, and launch scripts. | `gaming/minecraft-modpack-server` |
| `pokemon-player` | Play Pokemon games autonomously via headless emulation. Starts a game server, reads structured game state from RAM, makes strategic decisions, and sends button inputs -- all from the terminal. | `gaming/pokemon-player` |

## github

GitHub workflow skills for managing repositories, pull requests, code reviews, issues, and CI/CD pipelines using the gh CLI and git via terminal.

| Skill | Description | Path |
|-------|-------------|------|
| `codebase-inspection` | Inspect and analyze codebases using pygount for LOC counting, language breakdown, and code-vs-comment ratios. | `github/codebase-inspection` |
| `github-auth` | Set up GitHub authentication for the agent using git or the gh CLI. Covers HTTPS tokens, SSH keys, credential helpers, and gh auth. | `github/github-auth` |
| `github-code-review` | Review code changes by analyzing git diffs, leaving inline comments on PRs, and performing thorough pre-push review. | `github/github-code-review` |
| `github-issues` | Create, manage, triage, and close GitHub issues. Search existing issues, add labels, assign people, and link to PRs. | `github/github-issues` |
| `github-pr-workflow` | Full pull request lifecycle -- create branches, commit changes, open PRs, monitor CI status, auto-fix failures, and merge. | `github/github-pr-workflow` |
| `github-repo-management` | Clone, create, fork, configure, and manage GitHub repositories. Manage remotes, secrets, releases, and workflows. | `github/github-repo-management` |

## inference-sh

Skills for AI app execution via inference.sh cloud platform.

| Skill | Description | Path |
|-------|-------------|------|
| `inference-sh-cli` | Run 150+ AI apps via inference.sh CLI (infsh) -- image generation, video creation, LLMs, search, 3D, social automation. | `inference-sh/cli` |

## leisure

| Skill | Description | Path |
|-------|-------------|------|
| `find-nearby` | Find nearby places (restaurants, cafes, bars, pharmacies, etc.) using OpenStreetMap. Works with coordinates, addresses, cities, zip codes, or Telegram location pins. No API keys needed. | `leisure/find-nearby` |

## mcp

Skills for working with MCP (Model Context Protocol) servers, tools, and integrations.

| Skill | Description | Path |
|-------|-------------|------|
| `mcporter` | Use the mcporter CLI to list, configure, auth, and call MCP servers/tools directly (HTTP or stdio), including ad-hoc servers, config edits, and CLI/type generation. | `mcp/mcporter` |
| `native-mcp` | Built-in MCP client that connects to external MCP servers, discovers their tools, and registers them as native Hermes Agent tools. Supports stdio and HTTP transports with automatic reconnection, security filtering, and zero-config tool injection. | `mcp/native-mcp` |

## media

Skills for working with media content -- YouTube transcripts, GIF search, music generation, and audio visualization.

| Skill | Description | Path |
|-------|-------------|------|
| `gif-search` | Search and download GIFs from Tenor using curl. No dependencies beyond curl and jq. | `media/gif-search` |
| `heartmula` | Set up and run HeartMuLa, the open-source music generation model family (Suno-like). Generates full songs from lyrics + tags with multilingual support. | `media/heartmula` |
| `songsee` | Generate spectrograms and audio feature visualizations from audio files via CLI. Useful for audio analysis, music production debugging, and visual documentation. | `media/songsee` |
| `youtube-content` | Fetch YouTube video transcripts and transform them into structured content (chapters, summaries, threads, blog posts). | `media/youtube-content` |

## mlops

General-purpose ML operations tools -- model hub management, dataset operations, and workflow orchestration.

| Skill | Description | Path |
|-------|-------------|------|
| `huggingface-hub` | Hugging Face Hub CLI (hf) -- search, download, and upload models and datasets, manage repos, deploy inference endpoints. | `mlops/huggingface-hub` |

## mlops/cloud

GPU cloud providers and serverless compute platforms for ML workloads.

| Skill | Description | Path |
|-------|-------------|------|
| `lambda-labs-gpu-cloud` | Reserved and on-demand GPU cloud instances for ML training and inference. | `mlops/cloud/lambda-labs` |
| `modal-serverless-gpu` | Serverless GPU cloud platform for running ML workloads. On-demand GPU access without infrastructure management. | `mlops/cloud/modal` |

## mlops/evaluation

Model evaluation benchmarks, experiment tracking, data curation, tokenizers, and interpretability tools.

| Skill | Description | Path |
|-------|-------------|------|
| `evaluating-llms-harness` | Evaluates LLMs across 60+ academic benchmarks (MMLU, HumanEval, GSM8K, TruthfulQA, HellaSwag). Industry standard used by EleutherAI and HuggingFace. | `mlops/evaluation/lm-evaluation-harness` |
| `huggingface-tokenizers` | Fast Rust-based tokenizers for research and production. Tokenizes 1GB in <20 seconds. Supports BPE, WordPiece, and Unigram algorithms. | `mlops/evaluation/huggingface-tokenizers` |
| `nemo-curator` | GPU-accelerated data curation for LLM training. Text/image/video/audio. Fuzzy deduplication (16x faster), quality filtering (30+ heuristics), semantic deduplication, PII redaction. | `mlops/evaluation/nemo-curator` |
| `sparse-autoencoder-training` | Train and analyze Sparse Autoencoders (SAEs) using SAELens to decompose neural network activations into interpretable features. | `mlops/evaluation/saelens` |
| `weights-and-biases` | Track ML experiments with automatic logging, visualize training in real-time, optimize hyperparameters with sweeps, and manage model registry with W&B. | `mlops/evaluation/weights-and-biases` |

## mlops/inference

Model serving, quantization (GGUF/GPTQ), structured output, inference optimization, and model surgery tools.

| Skill | Description | Path |
|-------|-------------|------|
| `gguf-quantization` | GGUF format and llama.cpp quantization for efficient CPU/GPU inference. 2-8 bit quantization without GPU requirements. | `mlops/inference/gguf` |
| `guidance` | Control LLM output with regex and grammars, guarantee valid JSON/XML/code generation. Microsoft Research's constrained generation framework. | `mlops/inference/guidance` |
| `instructor` | Extract structured data from LLM responses with Pydantic validation, retry failed extractions automatically, stream partial results. | `mlops/inference/instructor` |
| `llama-cpp` | Run LLM inference on CPU, Apple Silicon, and consumer GPUs without NVIDIA hardware. Supports GGUF quantization (1.5-8 bit). | `mlops/inference/llama-cpp` |
| `obliteratus` | Remove refusal behaviors from open-weight LLMs using mechanistic interpretability techniques. 9 CLI methods, 28 analysis modules, 116 model presets. | `mlops/inference/obliteratus` |
| `outlines` | Guarantee valid JSON/XML/code structure during generation using Pydantic models. dottxt.ai's structured generation library. | `mlops/inference/outlines` |
| `serving-llms-vllm` | Serve LLMs with high throughput using vLLM's PagedAttention and continuous batching. Supports OpenAI-compatible endpoints. | `mlops/inference/vllm` |
| `tensorrt-llm` | Optimize LLM inference with NVIDIA TensorRT for maximum throughput and lowest latency on A100/H100 GPUs. | `mlops/inference/tensorrt-llm` |

## mlops/models

Specific model architectures and tools -- computer vision, speech, audio generation, and multimodal models.

| Skill | Description | Path |
|-------|-------------|------|
| `audiocraft-audio-generation` | PyTorch library for audio generation including text-to-music (MusicGen) and text-to-sound (AudioGen). | `mlops/models/audiocraft` |
| `clip` | OpenAI's model connecting vision and language. Zero-shot image classification, image-text matching, cross-modal retrieval. Trained on 400M image-text pairs. | `mlops/models/clip` |
| `llava` | Large Language and Vision Assistant. Visual instruction tuning and image-based conversations. Combines CLIP vision encoder with Vicuna/LLaMA language models. | `mlops/models/llava` |
| `segment-anything-model` | Foundation model for image segmentation with zero-shot transfer. Segment any object using points, boxes, or masks as prompts. | `mlops/models/segment-anything` |
| `stable-diffusion-image-generation` | Text-to-image generation with Stable Diffusion models via HuggingFace Diffusers. | `mlops/models/stable-diffusion` |
| `whisper` | OpenAI's general-purpose speech recognition model. 99 languages, transcription, translation. Six model sizes from tiny (39M) to large (1550M). | `mlops/models/whisper` |

## mlops/research

ML research frameworks for building and optimizing AI systems.

| Skill | Description | Path |
|-------|-------------|------|
| `dspy` | Build complex AI systems with declarative programming, optimize prompts automatically, create modular RAG systems and agents with DSPy. Stanford NLP's framework. | `mlops/research/dspy` |

## mlops/training

Fine-tuning, RLHF/DPO/GRPO training, distributed training frameworks, and optimization tools.

| Skill | Description | Path |
|-------|-------------|------|
| `axolotl` | Fine-tune LLMs with Axolotl -- YAML configs, 100+ models, LoRA/QLoRA, DPO/KTO/ORPO/GRPO, multimodal support. | `mlops/training/axolotl` |
| `distributed-llm-pretraining-torchtitan` | PyTorch-native distributed LLM pretraining using torchtitan with 4D parallelism (FSDP2, TP, PP, CP). | `mlops/training/torchtitan` |
| `fine-tuning-with-trl` | Fine-tune LLMs using reinforcement learning with TRL -- SFT, DPO, PPO/GRPO, and reward model training. | `mlops/training/trl-fine-tuning` |
| `grpo-rl-training` | GRPO/RL fine-tuning with TRL for reasoning and task-specific model training. | `mlops/training/grpo-rl-training` |
| `hermes-atropos-environments` | Build, test, and debug Hermes Agent RL environments for Atropos training. | `mlops/training/hermes-atropos-environments` |
| `huggingface-accelerate` | Simplest distributed training API. 4 lines to add distributed support to any PyTorch script. | `mlops/training/accelerate` |
| `optimizing-attention-flash` | Optimize transformer attention with Flash Attention for 2-4x speedup and 10-20x memory reduction. | `mlops/training/flash-attention` |
| `peft-fine-tuning` | Parameter-efficient fine-tuning using LoRA, QLoRA, and 25+ methods. Train <1% of parameters with minimal accuracy loss. | `mlops/training/peft` |
| `pytorch-fsdp` | Fully Sharded Data Parallel training with PyTorch FSDP -- parameter sharding, mixed precision, CPU offloading. | `mlops/training/pytorch-fsdp` |
| `pytorch-lightning` | High-level PyTorch framework with Trainer class, automatic distributed training, callbacks system, minimal boilerplate. | `mlops/training/pytorch-lightning` |
| `simpo-training` | Simple Preference Optimization for LLM alignment. Reference-free alternative to DPO with better performance (+6.4 points on AlpacaEval 2.0). | `mlops/training/simpo` |
| `slime-rl-training` | LLM post-training with RL using slime, a Megatron+SGLang framework. | `mlops/training/slime` |
| `unsloth` | Fast fine-tuning with Unsloth -- 2-5x faster training, 50-80% less memory, LoRA/QLoRA optimization. | `mlops/training/unsloth` |

## mlops/vector-databases

Vector similarity search and embedding databases for RAG, semantic search, and AI application backends.

| Skill | Description | Path |
|-------|-------------|------|
| `chroma` | Open-source embedding database for AI applications. Simple 4-function API. Scales from notebooks to production clusters. | `mlops/vector-databases/chroma` |
| `faiss` | Facebook's library for efficient similarity search and clustering of dense vectors. Supports billions of vectors, GPU acceleration. | `mlops/vector-databases/faiss` |
| `pinecone` | Managed vector database for production AI applications. Hybrid search, metadata filtering, auto-scaling. | `mlops/vector-databases/pinecone` |
| `qdrant-vector-search` | High-performance vector similarity search engine for RAG and semantic search. Rust-powered performance. | `mlops/vector-databases/qdrant` |

## note-taking

| Skill | Description | Path |
|-------|-------------|------|
| `obsidian` | Read, search, and create notes in the Obsidian vault. | `note-taking/obsidian` |

## productivity

| Skill | Description | Path |
|-------|-------------|------|
| `google-workspace` | Gmail, Calendar, Drive, Contacts, Sheets, and Docs integration via Python. Uses OAuth2 with automatic token refresh. | `productivity/google-workspace` |
| `linear` | Manage Linear issues, projects, and teams via the GraphQL API. | `productivity/linear` |
| `nano-pdf` | Edit PDFs with natural-language instructions using the nano-pdf CLI. | `productivity/nano-pdf` |
| `notion` | Notion API for creating and managing pages, databases, and blocks via curl. | `productivity/notion` |
| `ocr-and-documents` | Extract text from PDFs and scanned documents. Use web_extract for remote URLs, pymupdf for local text-based PDFs, marker-pdf for OCR/scanned docs. | `productivity/ocr-and-documents` |
| `powerpoint` | Use this skill any time a .pptx file is involved -- creating slide decks, reading, parsing, or extracting text from .pptx files. | `productivity/powerpoint` |

## research

Skills for academic research, paper discovery, literature review, domain reconnaissance, market data, content monitoring, and scientific knowledge retrieval.

| Skill | Description | Path |
|-------|-------------|------|
| `arxiv` | Search and retrieve academic papers from arXiv using their free REST API. No API key needed. | `research/arxiv` |
| `blogwatcher` | Monitor blogs and RSS/Atom feeds for updates using the blogwatcher CLI. | `research/blogwatcher` |
| `domain-intel` | Passive domain reconnaissance using Python stdlib. Subdomain discovery, SSL certificate inspection, WHOIS lookups, DNS records, domain availability checks. No API keys required. | `research/domain-intel` |
| `duckduckgo-search` | Free web search via DuckDuckGo -- text, news, images, videos. No API key needed. | `research/duckduckgo-search` |
| `ml-paper-writing` | Write publication-ready ML/AI papers for NeurIPS, ICML, ICLR, ACL, AAAI, COLM. Includes LaTeX templates, reviewer guidelines, and citation verification. | `research/ml-paper-writing` |
| `polymarket` | Query Polymarket prediction market data -- search markets, get prices, orderbooks, and price history. Read-only via public REST APIs, no API key needed. | `research/polymarket` |

## red-teaming

Skills for LLM red-teaming, jailbreaking, and safety filter bypass research.

| Skill | Description | Path |
|-------|-------------|------|
| `godmode` | Jailbreak API-served LLMs using G0DM0D3 techniques -- Parseltongue input obfuscation (33 techniques), GODMODE CLASSIC system prompt templates, ULTRAPLINIAN multi-model racing, encoding escalation. Works on any model accessible via API. | `red-teaming/godmode` |

## smart-home

| Skill | Description | Path |
|-------|-------------|------|
| `openhue` | Control Philips Hue lights, rooms, and scenes via the OpenHue CLI. Turn lights on/off, adjust brightness, color, color temperature, and activate scenes. | `smart-home/openhue` |

## social-media

| Skill | Description | Path |
|-------|-------------|------|
| `xitter` | Interact with X/Twitter via the x-cli terminal client using official X API credentials. | `social-media/xitter` |

## software-development

| Skill | Description | Path |
|-------|-------------|------|
| `code-review` | Guidelines for performing thorough code reviews with security and quality focus. | `software-development/code-review` |
| `plan` | Plan mode for Hermes -- inspect context, write a markdown plan into `.hermes/plans/` in the active workspace/backend working directory, and do not execute the work. | `software-development/plan` |
| `requesting-code-review` | Use when completing tasks, implementing major features, or before merging. Validates work meets requirements through systematic review process. | `software-development/requesting-code-review` |
| `subagent-driven-development` | Use when executing implementation plans with independent tasks. Dispatches fresh delegate_task per task with two-stage review. | `software-development/subagent-driven-development` |
| `systematic-debugging` | Use when encountering any bug, test failure, or unexpected behavior. 4-phase root cause investigation -- no fixes without understanding the problem first. | `software-development/systematic-debugging` |
| `test-driven-development` | Use when implementing any feature or bugfix, before writing implementation code. Enforces RED-GREEN-REFACTOR cycle with test-first approach. | `software-development/test-driven-development` |
| `writing-plans` | Use when you have a spec or requirements for a multi-step task. Creates comprehensive implementation plans with bite-sized tasks, exact file paths, and complete code examples. | `software-development/writing-plans` |
