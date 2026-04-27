# Official Optional Skills Catalog

Official optional skills live in the repository under `optional-skills/`. Install them with `hermes skills install official/<category>/<skill>` or browse them with `hermes skills browse --source official`.

## autonomous-ai-agents

| Skill | Description | Path |
|-------|-------------|------|
| `blackbox` | Delegate coding tasks to Blackbox AI CLI agent. Multi-model agent with built-in judge that runs tasks through multiple LLMs and picks the best result. Requires the blackbox CLI and a Blackbox AI API key. | `autonomous-ai-agents/blackbox` |

## blockchain

| Skill | Description | Path |
|-------|-------------|------|
| `base` | Query Base (Ethereum L2) blockchain data with USD pricing -- wallet balances, token info, transaction details, gas analysis, contract inspection, whale detection, and live network stats. Uses Base RPC + CoinGecko. No API key required. | `blockchain/base` |
| `solana` | Query Solana blockchain data with USD pricing -- wallet balances, token portfolios with values, transaction details, NFTs, whale detection, and live network stats. Uses Solana RPC + CoinGecko. No API key required. | `blockchain/solana` |

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
| `docker-management` | Manage Docker containers, images, volumes, networks, and Compose stacks -- lifecycle ops, debugging, cleanup, and Dockerfile optimization. | `devops/docker-management` |

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
| `neuroskill-bci` | Connect to a running NeuroSkill instance and incorporate the user's real-time cognitive and emotional state (focus, relaxation, mood, cognitive load, drowsiness, heart rate, HRV, sleep staging, and 40+ derived EXG scores) into responses. Requires a BCI wearable (Muse 2/S or OpenBCI) and the NeuroSkill desktop app. | `health/neuroskill-bci` |

## mcp

| Skill | Description | Path |
|-------|-------------|------|
| `fastmcp` | Build, test, inspect, install, and deploy MCP servers with FastMCP in Python. Use when creating a new MCP server, wrapping an API or database as MCP tools, exposing resources or prompts, or preparing a FastMCP server for HTTP deployment. | `mcp/fastmcp` |

## migration

| Skill | Description | Path |
|-------|-------------|------|
| `openclaw-migration` | Migrate a user's OpenClaw customization footprint into Hermes Agent. Imports Hermes-compatible memories, SOUL.md, command allowlists, user skills, and selected workspace assets from ~/.openclaw, then reports what could not be migrated and why. | `migration/openclaw-migration` |

## productivity

| Skill | Description | Path |
|-------|-------------|------|
| `telephony` | Give Hermes phone capabilities -- provision and persist a Twilio number, send and receive SMS/MMS, make direct calls, and place AI-driven outbound calls through Bland.ai or Vapi. | `productivity/telephony` |

## research

| Skill | Description | Path |
|-------|-------------|------|
| `bioinformatics` | Gateway to 400+ bioinformatics skills from bioSkills and ClawBio. Covers genomics, transcriptomics, single-cell, variant calling, pharmacogenomics, metagenomics, structural biology, and more. | `research/bioinformatics` |
| `drug-discovery` | Pharmaceutical research assistant for drug discovery workflows. Search bioactive compounds on ChEMBL, calculate drug-likeness (Lipinski Ro5, QED, TPSA, synthetic accessibility), and look up drug-drug interactions. Added in v0.11.0. | `research/drug-discovery` |
| `qmd` | Search personal knowledge bases, notes, docs, and meeting transcripts locally using qmd -- a hybrid retrieval engine with BM25, vector search, and LLM reranking. Supports CLI and MCP integration. | `research/qmd` |

## security

| Skill | Description | Path |
|-------|-------------|------|
| `1password` | Set up and use 1Password CLI (op). Use when installing the CLI, enabling desktop app integration, signing in, and reading/injecting secrets for commands. | `security/1password` |
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

Optional skills are not active by default because they cover heavier or niche use cases. Once installed, they appear in `hermes skills list` and can be loaded via slash commands or the skill_view tool.
