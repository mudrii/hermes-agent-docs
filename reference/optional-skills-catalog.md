# Official Optional Skills Catalog

Official optional skills live in the repository under `optional-skills/`. Install them with `hermes skills install official/<category>/<skill>` or browse them with `hermes skills browse --source official`.

At v0.11.0 (tag `v2026.4.23`) there are **59 optional skills** across 15 categories, in addition to the 72 bundled skills that ship in `~/.hermes/skills/`.

## autonomous-ai-agents

| Skill | Description | Path |
|-------|-------------|------|
| `blackbox` | Delegate coding tasks to Blackbox AI CLI agent. Multi-model agent with built-in judge that runs tasks through multiple LLMs and picks the best result. Requires the blackbox CLI and a Blackbox AI API key. | `autonomous-ai-agents/blackbox` |
| `honcho` | Configure and use Honcho memory with Hermes — cross-session user modeling, multi-profile peer isolation, observation config, dialectic reasoning, session summaries, and context budget enforcement. | `autonomous-ai-agents/honcho` |

## blockchain

| Skill | Description | Path |
|-------|-------------|------|
| `base` | Query Base (Ethereum L2) blockchain data with USD pricing — wallet balances, token info, transaction details, gas analysis, contract inspection, whale detection, and live network stats. Uses Base RPC + CoinGecko. No API key required. | `blockchain/base` |
| `solana` | Query Solana blockchain data with USD pricing — wallet balances, token portfolios with values, transaction details, NFTs, whale detection, and live network stats. Uses Solana RPC + CoinGecko. No API key required. | `blockchain/solana` |

## communication

| Skill | Description | Path |
|-------|-------------|------|
| `one-three-one-rule` | Structured decision-making framework for technical proposals: 1 problem statement, 3 distinct options with pros/cons, 1 concrete recommendation with definition of done. Use when the user asks for a "1-3-1" or needs help choosing between competing approaches. Added in v0.11.0. | `communication/one-three-one-rule` |

## creative

| Skill | Description | Path |
|-------|-------------|------|
| `blender-mcp` | Control Blender directly from Hermes via socket connection to the blender-mcp addon. Create 3D objects, materials, animations, and run arbitrary Blender Python (bpy) code. | `creative/blender-mcp` |
| `concept-diagrams` | Generate flat, minimal light/dark-aware SVG diagrams as standalone HTML files. 9 semantic color ramps, automatic dark mode, 15 example diagrams. Best for educational/non-software visuals (physics, chemistry, anatomy, floor plans, lifecycle journeys). Added in v0.11.0. | `creative/concept-diagrams` |
| `meme-generation` | Generate real meme images by picking a template and overlaying text with Pillow. Produces actual .png meme files. | `creative/meme-generation` |
| `touchdesigner-mcp` | Control a running TouchDesigner instance via the twozero MCP — create operators, set parameters, wire connections, execute Python, build real-time visuals. 36 native tools. Added in v0.11.0. | `creative/touchdesigner-mcp` |

## devops

| Skill | Description | Path |
|-------|-------------|------|
| `inference-sh-cli` | Run 150+ AI apps via inference.sh CLI (`infsh`) — image generation, video creation, LLMs, search, 3D, social automation. Triggers: inference.sh, infsh, FLUX, Veo, Seedream, Tavily. Added in v0.11.0. | `devops/cli` |
| `docker-management` | Manage Docker containers, images, volumes, networks, and Compose stacks — lifecycle ops, debugging, cleanup, and Dockerfile optimization. | `devops/docker-management` |

## dogfood

| Skill | Description | Path |
|-------|-------------|------|
| `adversarial-ux-test` | Roleplay the most difficult, tech-resistant user for your product. Browse the app as that persona, find every UX pain point, then filter complaints through a pragmatism layer to separate real problems from noise. Creates actionable tickets from genuine issues only. Added in v0.11.0. | `dogfood/adversarial-ux-test` |

## email

| Skill | Description | Path |
|-------|-------------|------|
| `agentmail` | Give the agent its own dedicated email inbox via AgentMail. Send, receive, and manage email autonomously using agent-owned email addresses (e.g. hermes-agent@agentmail.to). | `email/agentmail` |

## health

| Skill | Description | Path |
|-------|-------------|------|
| `fitness-nutrition` | Gym workout planner and nutrition tracker. Search 690+ exercises by muscle/equipment/category via wger. Look up macros and calories for 380,000+ foods via USDA FoodData Central. Compute BMI, TDEE, one-rep max, macro splits, and body fat. Added in v0.11.0. | `health/fitness-nutrition` |
| `neuroskill-bci` | Connect to a running NeuroSkill instance and incorporate the user's real-time cognitive and emotional state (focus, relaxation, mood, cognitive load, drowsiness, heart rate, HRV, sleep staging, and 40+ derived EXG scores). Requires a BCI wearable (Muse 2/S or OpenBCI) and the NeuroSkill desktop app. | `health/neuroskill-bci` |

## mcp

| Skill | Description | Path |
|-------|-------------|------|
| `fastmcp` | Build, test, inspect, install, and deploy MCP servers with FastMCP in Python. Use when creating a new MCP server, wrapping an API or database as MCP tools, exposing resources or prompts, or preparing a FastMCP server for HTTP deployment. | `mcp/fastmcp` |
| `mcporter` | Use the `mcporter` CLI to list, configure, auth, and call MCP servers/tools directly (HTTP or stdio), including ad-hoc servers, config edits, and CLI/type generation. Added in v0.11.0. | `mcp/mcporter` |

## migration

| Skill | Description | Path |
|-------|-------------|------|
| `openclaw-migration` | Migrate a user's OpenClaw customization footprint into Hermes Agent. Imports Hermes-compatible memories, SOUL.md, command allowlists, user skills, and selected workspace assets from `~/.openclaw`, then reports what could not be migrated and why. | `migration/openclaw-migration` |

## mlops

The `mlops` category bundles 25 ML/AI engineering skills covering distributed training, fine-tuning, vector databases, model serving, and multimodal models. All added in v0.11.0.

| Skill | Description | Path |
|-------|-------------|------|
| `huggingface-accelerate` | Simplest distributed training API. 4 lines to add distributed support to any PyTorch script. Unified API for DeepSpeed/FSDP/Megatron/DDP, automatic device placement, mixed precision (FP16/BF16/FP8). | `mlops/accelerate` |
| `chroma` | Open-source embedding database. Store embeddings + metadata, perform vector and full-text search, filter by metadata. Best for local development and open-source projects. | `mlops/chroma` |
| `clip` | OpenAI's vision-language model. Zero-shot image classification, image-text matching, cross-modal retrieval. Trained on 400M image-text pairs. | `mlops/clip` |
| `faiss` | Facebook's library for efficient similarity search and clustering of dense vectors. Supports billions of vectors, GPU acceleration, and various index types (Flat, IVF, HNSW). | `mlops/faiss` |
| `optimizing-attention-flash` | Optimize transformer attention with Flash Attention for 2-4x speedup and 10-20x memory reduction. Supports PyTorch native SDPA, flash-attn library, H100 FP8, and sliding window attention. | `mlops/flash-attention` |
| `guidance` | Control LLM output with regex and grammars, guarantee valid JSON/XML/code generation, enforce structured formats, and build multi-step workflows with Microsoft Research's constrained generation framework. | `mlops/guidance` |
| `hermes-atropos-environments` | Build, test, and debug Hermes Agent RL environments for Atropos training. Covers `HermesAgentBaseEnv`, reward functions, agent loop integration, evaluation with tools, wandb logging, and the three CLI modes (serve/process/evaluate). | `mlops/hermes-atropos-environments` |
| `huggingface-tokenizers` | Fast Rust-based tokenizers (1GB in <20s). Supports BPE, WordPiece, and Unigram algorithms. Train custom vocabularies, track alignments, handle padding/truncation. | `mlops/huggingface-tokenizers` |
| `instructor` | Extract structured data from LLM responses with Pydantic validation, retry failed extractions automatically, parse complex JSON with type safety, and stream partial results. | `mlops/instructor` |
| `lambda-labs-gpu-cloud` | Reserved and on-demand GPU cloud instances for ML training and inference. SSH access, persistent filesystems, multi-node clusters. | `mlops/lambda-labs` |
| `llava` | Large Language and Vision Assistant. Visual instruction tuning and image-based conversations combining CLIP vision encoder with Vicuna/LLaMA language models. | `mlops/llava` |
| `modal-serverless-gpu` | Serverless GPU cloud platform for running ML workloads. On-demand GPU access, deploy ML models as APIs, run batch jobs with automatic scaling. | `mlops/modal` |
| `nemo-curator` | GPU-accelerated data curation for LLM training. Text/image/video/audio. Fuzzy deduplication (16× faster), 30+ quality heuristics, semantic deduplication, PII redaction, NSFW detection. | `mlops/nemo-curator` |
| `peft-fine-tuning` | Parameter-efficient fine-tuning for LLMs using LoRA, QLoRA, and 25+ methods. Fine-tune 7B-70B models with limited GPU memory, train <1% of parameters. | `mlops/peft` |
| `pinecone` | Managed vector database for production AI applications. Auto-scaling, hybrid search (dense + sparse), metadata filtering, namespaces. Low latency (<100ms p95). | `mlops/pinecone` |
| `pytorch-fsdp` | Fully Sharded Data Parallel training with PyTorch FSDP — parameter sharding, mixed precision, CPU offloading, FSDP2. | `mlops/pytorch-fsdp` |
| `pytorch-lightning` | High-level PyTorch framework with Trainer class, automatic distributed training (DDP/FSDP/DeepSpeed), callbacks system, and minimal boilerplate. | `mlops/pytorch-lightning` |
| `qdrant-vector-search` | High-performance vector similarity search engine. Production RAG with fast nearest neighbor search, hybrid search with filtering, scalable Rust-powered storage. | `mlops/qdrant` |
| `sparse-autoencoder-training` | Train and analyze Sparse Autoencoders (SAEs) using SAELens to decompose neural network activations into interpretable features. For mechanistic interpretability research. | `mlops/saelens` |
| `simpo-training` | Simple Preference Optimization for LLM alignment. Reference-free DPO alternative (+6.4 points on AlpacaEval 2.0). No reference model needed. | `mlops/simpo` |
| `slime-rl-training` | LLM post-training with RL using slime, a Megatron+SGLang framework. For training GLM models, custom data generation workflows, or Megatron-LM RL scaling. | `mlops/slime` |
| `stable-diffusion-image-generation` | State-of-the-art text-to-image generation with Stable Diffusion via HuggingFace Diffusers. Text-to-image, image-to-image translation, inpainting, custom diffusion pipelines. | `mlops/stable-diffusion` |
| `tensorrt-llm` | Optimize LLM inference with NVIDIA TensorRT for maximum throughput. 10-100x faster than PyTorch on A100/H100. Quantization (FP8/INT4), in-flight batching, multi-GPU. | `mlops/tensorrt-llm` |
| `distributed-llm-pretraining-torchtitan` | PyTorch-native distributed LLM pretraining using torchtitan with 4D parallelism (FSDP2, TP, PP, CP). Llama 3.1, DeepSeek V3, custom models from 8 to 512+ GPUs. | `mlops/torchtitan` |
| `whisper` | OpenAI's speech recognition model. 99 languages, transcription, translation to English, language ID. Six model sizes from tiny (39M) to large (1550M). | `mlops/whisper` |

## productivity

| Skill | Description | Path |
|-------|-------------|------|
| `canvas` | Canvas LMS integration — fetch enrolled courses and assignments using API token authentication. Added in v0.11.0. | `productivity/canvas` |
| `memento-flashcards` | Spaced-repetition flashcard system. Create cards from facts or text, chat with flashcards using free-text answers graded by the agent, generate quizzes from YouTube transcripts. Added in v0.11.0. | `productivity/memento-flashcards` |
| `siyuan` | SiYuan Note API for searching, reading, creating, and managing blocks and documents in a self-hosted knowledge base via curl. Added in v0.11.0. | `productivity/siyuan` |
| `telephony` | Give Hermes phone capabilities — provision and persist a Twilio number, send and receive SMS/MMS, make direct calls, and place AI-driven outbound calls through Bland.ai or Vapi. | `productivity/telephony` |

## research

| Skill | Description | Path |
|-------|-------------|------|
| `bioinformatics` | Gateway to 400+ bioinformatics skills from bioSkills and ClawBio. Covers genomics, transcriptomics, single-cell, variant calling, pharmacogenomics, metagenomics, structural biology, and more. | `research/bioinformatics` |
| `domain-intel` | Passive domain reconnaissance using Python stdlib. Subdomain discovery, SSL certificate inspection, WHOIS lookups, DNS records, domain availability checks, bulk multi-domain analysis. No API keys required. Added in v0.11.0. | `research/domain-intel` |
| `drug-discovery` | Pharmaceutical research assistant for drug discovery workflows. Search bioactive compounds on ChEMBL, calculate drug-likeness (Lipinski Ro5, QED, TPSA, synthetic accessibility), and look up drug-drug interactions. Added in v0.11.0. | `research/drug-discovery` |
| `duckduckgo-search` | Free web search via DuckDuckGo — text, news, images, videos. No API key needed. Prefers the `ddgs` CLI when installed; falls back to the Python DDGS library. | `research/duckduckgo-search` |
| `gitnexus-explorer` | Index a codebase with GitNexus and serve an interactive knowledge graph via web UI + Cloudflare tunnel. Added in v0.11.0. | `research/gitnexus-explorer` |
| `parallel-cli` | Vendor skill for Parallel CLI — agent-native web search, extraction, deep research, enrichment, FindAll, and monitoring. Prefers JSON output and non-interactive flows. Added in v0.11.0. | `research/parallel-cli` |
| `qmd` | Search personal knowledge bases, notes, docs, and meeting transcripts locally using qmd — a hybrid retrieval engine with BM25, vector search, and LLM reranking. Supports CLI and MCP integration. | `research/qmd` |
| `scrapling` | Web scraping with Scrapling — HTTP fetching, stealth browser automation, Cloudflare bypass, and spider crawling via CLI and Python. Added in v0.11.0. | `research/scrapling` |

## security

| Skill | Description | Path |
|-------|-------------|------|
| `1password` | Set up and use 1Password CLI (`op`). Use when installing the CLI, enabling desktop app integration, signing in, and reading/injecting secrets for commands. | `security/1password` |
| `oss-forensics` | Supply chain investigation, evidence recovery, and forensic analysis for GitHub repositories. Covers deleted commit recovery, force-push detection, IOC extraction, multi-source evidence collection, and structured forensic reporting. | `security/oss-forensics` |
| `sherlock` | OSINT username search across 400+ social networks. Hunt down social media accounts by username. | `security/sherlock` |

## web-development

| Skill | Description | Path |
|-------|-------------|------|
| `page-agent` | Embed alibaba/page-agent into your own web application — a pure-JavaScript in-page GUI agent that ships as a single `<script>` tag or npm package and lets end-users drive the UI with natural language. For web developers adding an AI copilot to their SaaS / admin panel. NOT for server-side browser automation (use the built-in browser tool instead). Added in v0.11.0. | `web-development/page-agent` |

## Installing optional skills

```bash
# Browse all optional skills
hermes skills browse --source official

# Install a specific skill
hermes skills install official/security/1password
hermes skills install official/blockchain/solana
hermes skills install official/creative/blender-mcp

# Search for skills
hermes skills search blockchain --source official
```

Optional skills are not active by default because they cover heavier or niche use cases. Once installed, they appear in `hermes skills list` and can be loaded via slash commands or the `skill_view` tool.
