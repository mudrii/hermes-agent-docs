# Documentation Integrity Findings: Post-v0.12.0 Upstream Corrections (2026-05-06)

## Scope

- Source: post-`v2026.4.30` upstream doc corrections on `hermes-agent` main (`aa88dcc57`)
- Docs baseline: `hermes-agent-docs` at `3c7099b` (previously synced to v0.12.0)
- Release boundary: v0.12.0 / `v2026.4.30` — post-tag unreleased features excluded

## Files Updated

### Developer Guide (5 files)
- `developer-guide/adding-providers.md` — added "Fast path: Simple API-key providers" section documenting the `plugins/model-providers/` plugin system with 12 auto-wired items; added "Full path: OAuth and complex providers" header
- `developer-guide/adding-tools.md` — added built-in-core-tool warning callout; plugin-route vs built-in distinction
- `developer-guide/contributing.md` — split tool-building guidance into plugin vs built-in paths
- `developer-guide/prompt-assembly.md` — added "Supported prompt customization surfaces" section
- `developer-guide/provider-runtime.md` — added `providers/` ABC/registry and plugin auto-pickup entries

### Getting Started (4 files)
- `getting-started/quickstart.md` — added Onchain AI Garage video embed; added AWS Bedrock provider row
- `getting-started/nix-setup.md` — fixed container-aware CLI anchor/heading structure
- `getting-started/learning-path.md` — prepended plugin-first reading path for custom tools
- `getting-started/updating.md` — added Teams to messaging platforms update section

### Guides (7 files, 3 new)
- `guides/cron-script-only.md` — **new file**: no-agent cron mode (v0.12.0 `wakeAgent` gate)
- `guides/google-gemini.md` — **new file**: full native Gemini provider guide
- `guides/local-ollama-setup.md` — **new file**: Ollama + vLLM local setup guide
- `guides/automate-with-cron.md` — added cron-script-only tip callout
- `guides/aws-bedrock.md` — added one-click CloudFormation deployment section
- `guides/build-a-hermes-plugin.md` — added `ctx.dispatch_tool()` API section
- `guides/use-mcp-with-hermes.md` — added WSL2+Windows Chrome MCP bridge section

### Reference (5 files modified, 1 already current)
- `reference/environment-variables.md` — added HERMES_OPENROUTER_CACHE*, GMI inference URLs, STEPFUN_*, FEISHU_ALLOW_BOTS, HERMES_GATEWAY_BUSY_ACK_ENABLED; fixed TERMINAL_CWD description; updated fallback_providers YAML
- `reference/faq.md` — added WSL2/Windows Chrome FAQ; fixed messaging install to `[messaging]`
- `reference/optional-skills-catalog.md` — added here.now and shopify skills
- `reference/skills-catalog.md` — added sync/restore paragraph; updated comfyui/obsidian descriptions
- `reference/slash-commands.md` — updated Teams reference; excluded post-tag kanban/goal entries
- `reference/cli-commands.md` — no changes needed (already current)

### User Guide Features (10 files)
- `user-guide/features/acp.md` — corrected VS Code setup to ACP Client extension flow
- `user-guide/features/browser.md` — added Camofox Docker workflow; added WSL2+Windows Chrome MCP bridge
- `user-guide/features/cron.md` — added no-agent mode section; added `context_from` chaining section
- `user-guide/features/curator.md` — added `--dry-run`/`backup`/`rollback` subcommands; corrected pinning description
- `user-guide/features/fallback-providers.md` — added optional base URL notes for GMI and StepFun
- `user-guide/features/hooks.md` — no changes needed (already current)
- `user-guide/features/plugins.md` — added custom-vs-core guidance; updated minimal example; added `dispatch_tool` row
- `user-guide/features/tools.md` — added persistent Docker container explanation
- `user-guide/features/tts.md` — added per-provider `max_text_length` table; added Doubao/Volcengine ASR section
- `user-guide/features/voice-mode.md` — corrected Discord SSRC/SPEAKING opcode/intent description

### Messaging (4 files modified, 1 already current)
- `user-guide/messaging/telegram.md` — added group chat troubleshooting; added `/topic` DM mode section
- `user-guide/messaging/teams.md` — corrected @mention behavior; fixed install flow; added TEAMS_ALLOW_ALL_USERS
- `user-guide/messaging/open-webui.md` — added bootstrap script; API verification steps; Ollama notes
- `user-guide/messaging/index.md` — added `busy_ack_enabled` config key with explanatory paragraph
- `user-guide/messaging/feishu.md` — no changes needed (already current)

### User Guide Core (4 files modified, 1 new)
- `user-guide/docker.md` — added API_SERVER_* env vars; rewrote dashboard section to HERMES_DASHBOARD=1 model; added vLLM/Ollama inference section
- `user-guide/configuration.md` — added `display.language` i18n config option; corrected TERMINAL_CWD references
- `user-guide/configuring-models.md` — added `model_aliases` custom aliases section
- `user-guide/windows-wsl-quickstart.md` — **new file**: WSL2 minimum path quickstart
- `user-guide/sessions.md` — no changes needed (already current)

### Skills (3 files)
- `user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-codex.md` — corrected prerequisites to OAuth/API-key dual-path
- `user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-hermes-agent.md` — added Teams to supported platforms
- `user-guide/skills/bundled/note-taking/note-taking-obsidian.md` — rewrote to file-tool guidance (read_file/write_file/patch)
- `user-guide/skills/bundled/creative/creative-comfyui.md` — no changes needed (already at v5)
- `user-guide/skills/bundled/devops/devops-kanban-orchestrator.md` — excluded (zero bytes at v0.12.0 tag, post-tag addition)
- `user-guide/skills/bundled/devops/devops-kanban-worker.md` — excluded (same reason)

### Integrations (2 files)
- `integrations/index.md` — updated platform count to 19+; added Yuanbao
- `integrations/providers.md` — added GMI Cloud section; added StepFun section; added Together/Groq/Perplexity cookbook; added Gemini guide link

## Excluded (Post-Tag Unreleased)

- `user-guide/features/goals.md` — not in v0.12.0 release note
- `user-guide/features/kanban-tutorial.md` — post-tag tutorial
- `user-guide/skills/bundled/devops/devops-kanban-*` — zero bytes at v0.12.0 tag
- `user-stories.mdx` — MDX format, not used in standalone docs repo

## Notes

All changes are sourced to upstream `website/docs/` files present in `hermes-agent` main as of `aa88dcc57`. Content is bounded to v0.12.0 released features only. Post-tag unreleased content was identified by checking file existence at the `v2026.4.30` tag and cross-referencing `RELEASE_v0.12.0.md`.
