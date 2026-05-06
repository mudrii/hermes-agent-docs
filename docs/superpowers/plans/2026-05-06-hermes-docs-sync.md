# Hermes Agent Docs Sync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sync hermes-agent-docs to reflect all documentation corrections and improvements present in the hermes-agent main branch (post v0.12.0 tag), covering only released v0.12.0 features.

**Architecture:** Compare upstream `website/docs/` files from hermes-agent main against hermes-agent-docs counterparts. Import verbatim content from the upstream source for pure corrections; adapt where the standalone docs repo has a different structure. Skip post-tag unreleased features (goals.md, kanban-tutorial.md). Add two new guide files that document v0.12.0 behavior.

**Tech Stack:** Markdown docs, hermes-agent source at `~/src/hermes/hermes-agent`, docs repo at `~/src/hermes/hermes-agent-docs`.

---

## Context

- Released version: **v0.12.0** / `v2026.4.30`
- hermes-agent main is 636 commits ahead of the v0.12.0 tag
- hermes-agent-docs was synced to v0.12.0 in commit `3c7099b`
- 58 `website/docs/` files changed in hermes-agent since v0.12.0; most are corrections/improvements for already-released features

**Files to update** (sourced from `git diff v2026.4.30..HEAD --name-only -- website/docs/`):

### Group 1 — Developer Guide (5 files)
| Source (hermes-agent) | Destination (hermes-agent-docs) |
|---|---|
| `website/docs/developer-guide/adding-providers.md` | `developer-guide/adding-providers.md` |
| `website/docs/developer-guide/adding-tools.md` | `developer-guide/adding-tools.md` |
| `website/docs/developer-guide/contributing.md` | `developer-guide/contributing.md` |
| `website/docs/developer-guide/prompt-assembly.md` | `developer-guide/prompt-assembly.md` |
| `website/docs/developer-guide/provider-runtime.md` | `developer-guide/provider-runtime.md` |

### Group 2 — Getting Started (4 files)
| Source | Destination |
|---|---|
| `website/docs/getting-started/learning-path.md` | `getting-started/learning-path.md` |
| `website/docs/getting-started/nix-setup.md` | `getting-started/nix-setup.md` |
| `website/docs/getting-started/quickstart.md` | `getting-started/quickstart.md` |
| `website/docs/getting-started/updating.md` | `getting-started/updating.md` |

### Group 3 — Guides (7 files, 2 new)
| Source | Destination | Status |
|---|---|---|
| `website/docs/guides/automate-with-cron.md` | `guides/automate-with-cron.md` | update |
| `website/docs/guides/aws-bedrock.md` | `guides/aws-bedrock.md` | update |
| `website/docs/guides/build-a-hermes-plugin.md` | `guides/build-a-hermes-plugin.md` | update |
| `website/docs/guides/cron-script-only.md` | `guides/cron-script-only.md` | **NEW** |
| `website/docs/guides/google-gemini.md` | `guides/google-gemini.md` | update |
| `website/docs/guides/local-ollama-setup.md` | `guides/local-ollama-setup.md` | update |
| `website/docs/guides/use-mcp-with-hermes.md` | `guides/use-mcp-with-hermes.md` | update |

### Group 4 — Reference (6 files)
| Source | Destination |
|---|---|
| `website/docs/reference/cli-commands.md` | `reference/cli-commands.md` |
| `website/docs/reference/environment-variables.md` | `reference/environment-variables.md` |
| `website/docs/reference/faq.md` | `reference/faq.md` |
| `website/docs/reference/optional-skills-catalog.md` | `reference/optional-skills-catalog.md` |
| `website/docs/reference/skills-catalog.md` | `reference/skills-catalog.md` |
| `website/docs/reference/slash-commands.md` | `reference/slash-commands.md` |

### Group 5 — User Guide Features (10 files)
| Source | Destination |
|---|---|
| `website/docs/user-guide/features/acp.md` | `features/acp.md` |
| `website/docs/user-guide/features/browser.md` | `features/browser.md` OR `browser.md` |
| `website/docs/user-guide/features/cron.md` | `cron.md` OR `features/cron.md` |
| `website/docs/user-guide/features/curator.md` | `features/curator.md` OR `curator.md` |
| `website/docs/user-guide/features/fallback-providers.md` | `features/fallback-providers.md` OR `fallback-providers.md` |
| `website/docs/user-guide/features/hooks.md` | `hooks.md` OR `features/hooks.md` |
| `website/docs/user-guide/features/plugins.md` | `features/plugins.md` OR `plugins.md` |
| `website/docs/user-guide/features/tools.md` | `features/tools.md` OR `tools.md` |
| `website/docs/user-guide/features/tts.md` | `tts.md` OR `features/tts.md` |
| `website/docs/user-guide/features/voice-mode.md` | `voice-mode.md` OR `features/voice-mode.md` |

### Group 6 — Messaging (5 files)
| Source | Destination |
|---|---|
| `website/docs/user-guide/messaging/feishu.md` | `messaging/feishu.md` |
| `website/docs/user-guide/messaging/index.md` | `messaging/README.md` or `messaging/index.md` |
| `website/docs/user-guide/messaging/open-webui.md` | `messaging/open-webui.md` |
| `website/docs/user-guide/messaging/teams.md` | `messaging/teams.md` |
| `website/docs/user-guide/messaging/telegram.md` | `messaging/telegram.md` |

### Group 7 — User Guide Core (5 files)
| Source | Destination |
|---|---|
| `website/docs/user-guide/configuration.md` | `user-guide/configuration.md` |
| `website/docs/user-guide/configuring-models.md` | `user-guide/configuring-models.md` |
| `website/docs/user-guide/docker.md` | `docker.md` |
| `website/docs/user-guide/sessions.md` | `sessions.md` |
| `website/docs/user-guide/windows-wsl-quickstart.md` | `user-guide/windows-wsl-quickstart.md` |

### Group 8 — Skills (6 files)
| Source | Destination |
|---|---|
| `website/docs/user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-codex.md` | find/create mirror path |
| `website/docs/user-guide/skills/bundled/autonomous-ai-agents/autonomous-ai-agents-hermes-agent.md` | find/create mirror path |
| `website/docs/user-guide/skills/bundled/creative/creative-comfyui.md` | find/create mirror path |
| `website/docs/user-guide/skills/bundled/devops/devops-kanban-orchestrator.md` | find/create mirror path |
| `website/docs/user-guide/skills/bundled/devops/devops-kanban-worker.md` | find/create mirror path |
| `website/docs/user-guide/skills/bundled/note-taking/note-taking-obsidian.md` | find/create mirror path |

### Group 9 — Index/Integration (3 files)
| Source | Destination |
|---|---|
| `website/docs/index.md` | `index.md` |
| `website/docs/integrations/index.md` | `integrations/index.md` |
| `website/docs/integrations/providers.md` | `integrations/providers.md` OR `providers.md` |

### Skip (post-tag unreleased features)
- `website/docs/user-guide/features/goals.md` — not in v0.12.0 release note
- `website/docs/user-guide/features/kanban-tutorial.md` — tutorial for post-tag kanban enhancements
- `website/docs/user-stories.mdx` — mdx format, not needed in standalone docs

---

## File Structure

All changes are to existing files in `~/src/hermes/hermes-agent-docs/`. Two new files to create:
- `guides/cron-script-only.md` (new guide for cron without LLM)
- `guides/local-ollama-setup.md` (if not already present — check first)

---

## Tasks

### Task 1: Developer Guide Updates

**Files:**
- Modify: `~/src/hermes/hermes-agent-docs/developer-guide/adding-providers.md`
- Modify: `~/src/hermes/hermes-agent-docs/developer-guide/adding-tools.md`
- Modify: `~/src/hermes/hermes-agent-docs/developer-guide/contributing.md`
- Modify: `~/src/hermes/hermes-agent-docs/developer-guide/prompt-assembly.md`
- Modify: `~/src/hermes/hermes-agent-docs/developer-guide/provider-runtime.md`

- [ ] **Step 1: Read each source file from hermes-agent**

```bash
# Read each file from hermes-agent main
cat ~/src/hermes/hermes-agent/website/docs/developer-guide/adding-providers.md
cat ~/src/hermes/hermes-agent/website/docs/developer-guide/adding-tools.md
cat ~/src/hermes/hermes-agent/website/docs/developer-guide/contributing.md
cat ~/src/hermes/hermes-agent/website/docs/developer-guide/prompt-assembly.md
cat ~/src/hermes/hermes-agent/website/docs/developer-guide/provider-runtime.md
```

- [ ] **Step 2: Compare each file against the docs repo counterpart**

For each file, identify:
1. Content that exists in source but not docs (additions to make)
2. Content in docs that is now stale/incorrect (corrections)
3. Content that is identical or structurally equivalent (skip)

- [ ] **Step 3: Apply the key addition to adding-providers.md**

The source now has a "Fast path: Simple API-key providers" section documenting the `plugins/model-providers/` plugin system. This was released in v0.12.0 and needs to be added to the docs after the existing "Path A" / "Path B" description.

Content to insert (after the current "Choose the implementation path first" section):

```markdown
## Fast path: Simple API-key providers

If your provider is just an OpenAI-compatible endpoint that authenticates with a single API key, you do not need to touch `auth.py`, `runtime_provider.py`, `main.py`, or any of the other files in the full checklist below.

All you need is:

1. A plugin directory under `plugins/model-providers/<your-provider>/` containing:
   - `__init__.py` — calls `register_provider(profile)` at module-level
   - `plugin.yaml` — manifest (name, kind: model-provider, version, description)
2. That's it. Provider plugins auto-load the first time anything calls `get_provider_profile()` or `list_providers()` — bundled plugins (this repo) and user plugins at `$HERMES_HOME/plugins/model-providers/` both get picked up.

When you add a plugin and it calls `register_provider()`, the following wire up automatically:

1. `PROVIDER_REGISTRY` entry in `auth.py` (credential resolution, env-var lookup)
2. `api_mode` set to `chat_completions`
3. `base_url` sourced from the config or the declared env var
4. `env_vars` checked in priority order for the API key
5. `fallback_models` list registered for the provider
6. `--provider` CLI flag accepts the provider id
7. `hermes model` menu includes the provider
8. `hermes setup` wizard delegates to `main.py` automatically
9. `provider:model` alias syntax works
10. Runtime resolver returns the correct `base_url` and `api_key`
11. `HERMES_INFERENCE_PROVIDER` env-var override accepts the provider id
12. Fallback model activation can switch into the provider cleanly

User plugins at `$HERMES_HOME/plugins/model-providers/<name>/` override bundled plugins of the same name (last-writer-wins in `register_provider()`) — so third parties can monkey-patch or replace any built-in profile without editing the repo.

See `plugins/model-providers/nvidia/` or `plugins/model-providers/gmi/` as a template, and `plugins/model-providers/README.md` for the full contract.

## Full path: OAuth and complex providers

Use the full checklist below when your provider needs any of the following:

- OAuth or token refresh (Nous Portal, Codex, Google Gemini, Qwen Portal, Copilot)
- A non-OpenAI API shape that requires a new adapter (Anthropic Messages, Codex Responses)
- Custom endpoint detection or multi-region probing (z.ai, Kimi)
- A curated static model catalog or live `/models` fetch
- Provider-specific `hermes model` menu entries with bespoke auth flows
```

- [ ] **Step 4: Apply corrections to other developer-guide files**

For each of the remaining 4 files, read the diff and apply factual corrections only. Do not apply docs for post-tag features.

```bash
# Check diffs
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/developer-guide/adding-tools.md
git diff v2026.4.30..HEAD -- website/docs/developer-guide/contributing.md
git diff v2026.4.30..HEAD -- website/docs/developer-guide/prompt-assembly.md
git diff v2026.4.30..HEAD -- website/docs/developer-guide/provider-runtime.md
```

- [ ] **Step 5: Commit developer-guide updates**

```bash
cd ~/src/hermes/hermes-agent-docs
git add developer-guide/
git commit -m "docs(developer-guide): sync provider fast-path and developer guide corrections from upstream"
```

---

### Task 2: Getting Started Updates

**Files:**
- Modify: `~/src/hermes/hermes-agent-docs/getting-started/learning-path.md`
- Modify: `~/src/hermes/hermes-agent-docs/getting-started/nix-setup.md`
- Modify: `~/src/hermes/hermes-agent-docs/getting-started/quickstart.md`
- Modify: `~/src/hermes/hermes-agent-docs/getting-started/updating.md`

- [ ] **Step 1: Diff each getting-started file**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/getting-started/
```

- [ ] **Step 2: Read current docs counterparts**

```bash
cat ~/src/hermes/hermes-agent-docs/getting-started/learning-path.md
cat ~/src/hermes/hermes-agent-docs/getting-started/nix-setup.md
cat ~/src/hermes/hermes-agent-docs/getting-started/quickstart.md
cat ~/src/hermes/hermes-agent-docs/getting-started/updating.md
```

- [ ] **Step 3: Apply corrections**

Key changes in upstream:
- quickstart.md: link to Onchain AI Garage Hermes tutorials playlist added
- nix-setup.md: anchor correction for container-aware CLI section
- updating.md: corrections to update procedure docs

Apply factual corrections only — no post-tag feature documentation.

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add getting-started/
git commit -m "docs(getting-started): sync corrections from upstream"
```

---

### Task 3: Guides Updates (including 2 new files)

**Files:**
- Modify: `guides/automate-with-cron.md`
- Modify: `guides/aws-bedrock.md`
- Modify: `guides/build-a-hermes-plugin.md`
- Create: `guides/cron-script-only.md` (NEW)
- Modify: `guides/google-gemini.md`
- Modify: `guides/local-ollama-setup.md`
- Modify: `guides/use-mcp-with-hermes.md`

- [ ] **Step 1: Read all source guide files**

```bash
cat ~/src/hermes/hermes-agent/website/docs/guides/automate-with-cron.md
cat ~/src/hermes/hermes-agent/website/docs/guides/aws-bedrock.md
cat ~/src/hermes/hermes-agent/website/docs/guides/build-a-hermes-plugin.md
cat ~/src/hermes/hermes-agent/website/docs/guides/cron-script-only.md
cat ~/src/hermes/hermes-agent/website/docs/guides/google-gemini.md
cat ~/src/hermes/hermes-agent/website/docs/guides/local-ollama-setup.md
cat ~/src/hermes/hermes-agent/website/docs/guides/use-mcp-with-hermes.md
```

- [ ] **Step 2: Read current docs counterparts (where they exist)**

```bash
ls ~/src/hermes/hermes-agent-docs/guides/
cat ~/src/hermes/hermes-agent-docs/guides/automate-with-cron.md 2>/dev/null || echo "MISSING"
cat ~/src/hermes/hermes-agent-docs/guides/aws-bedrock.md 2>/dev/null || echo "MISSING"
cat ~/src/hermes/hermes-agent-docs/guides/build-a-hermes-plugin.md 2>/dev/null || echo "MISSING"
cat ~/src/hermes/hermes-agent-docs/guides/google-gemini.md 2>/dev/null || echo "MISSING"
cat ~/src/hermes/hermes-agent-docs/guides/local-ollama-setup.md 2>/dev/null || echo "MISSING"
cat ~/src/hermes/hermes-agent-docs/guides/use-mcp-with-hermes.md 2>/dev/null || echo "MISSING"
```

- [ ] **Step 3: Create guides/cron-script-only.md (NEW)**

Copy content from `~/src/hermes/hermes-agent/website/docs/guides/cron-script-only.md` verbatim (this documents the v0.12.0 cron `wakeAgent` gate feature where cron scripts can bypass the LLM entirely).

- [ ] **Step 4: Apply diffs to existing guides**

Key changes:
- `automate-with-cron.md`: `context_from` chaining section added
- `aws-bedrock.md`: IAM permissions fix, quickstart entry, fallback provider section
- `build-a-hermes-plugin.md`: `ctx.dispatch_tool()` documentation section added
- `google-gemini.md`: new guide content
- `local-ollama-setup.md`: vLLM/Ollama connection section
- `use-mcp-with-hermes.md`: corrections

- [ ] **Step 5: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add guides/
git commit -m "docs(guides): add cron-script-only guide, sync corrections from upstream"
```

---

### Task 4: Reference Updates

**Files:**
- Modify: `reference/cli-commands.md`
- Modify: `reference/environment-variables.md`
- Modify: `reference/faq.md`
- Modify: `reference/optional-skills-catalog.md`
- Modify: `reference/skills-catalog.md`
- Modify: `reference/slash-commands.md`

- [ ] **Step 1: Diff each reference file**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/reference/
```

- [ ] **Step 2: Read current docs counterparts**

```bash
cat ~/src/hermes/hermes-agent-docs/reference/cli-commands.md
cat ~/src/hermes/hermes-agent-docs/reference/environment-variables.md
cat ~/src/hermes/hermes-agent-docs/reference/faq.md
cat ~/src/hermes/hermes-agent-docs/reference/optional-skills-catalog.md
cat ~/src/hermes/hermes-agent-docs/reference/skills-catalog.md
cat ~/src/hermes/hermes-agent-docs/reference/slash-commands.md
```

- [ ] **Step 3: Apply corrections**

Key changes:
- `cli-commands.md`: `hermes skills reset` subcommand added, `hermes import` expanded with description/warning/examples, `--deliver-only` flag for `hermes webhook subscribe`
- `environment-variables.md`: new env vars documented
- `faq.md`: messaging extra for gateway deps clarification
- `skills-catalog.md` / `optional-skills-catalog.md`: updated catalogs with v0.12.0 skills
- `slash-commands.md`: corrections to slash command list

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add reference/
git commit -m "docs(reference): sync CLI commands, env vars, FAQ, skills catalogs from upstream"
```

---

### Task 5: User Guide Features Updates

**Files:**
- Modify: `features/acp.md`
- Modify: `features/browser.md` (or `browser.md`)
- Modify: `features/cron.md` (or `cron.md`)
- Modify: `features/curator.md` (or `curator.md`)
- Modify: `features/fallback-providers.md` (or `fallback-providers.md`)
- Modify: `features/hooks.md` (or `hooks.md`)
- Modify: `features/plugins.md` (or `plugins.md`)
- Modify: `features/tools.md` (or `tools.md`)
- Modify: `features/tts.md` (or `tts.md`)
- Modify: `features/voice-mode.md` (or `voice-mode.md`)

- [ ] **Step 1: Diff features files**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/user-guide/features/
```

- [ ] **Step 2: Locate docs counterparts**

The docs repo has a mixed structure. Check which files are under `features/` vs top-level:

```bash
ls ~/src/hermes/hermes-agent-docs/features/
ls ~/src/hermes/hermes-agent-docs/*.md | grep -E "browser|cron|curator|fallback|hooks|plugins|tools|tts|voice"
```

- [ ] **Step 3: Apply corrections for each feature**

Key changes per feature:
- `acp.md`: VS Code setup instructions corrected
- `browser.md`: WSL-to-Windows Chrome MCP bridge documented
- `cron.md`: `context_from` chaining section added; cron per-job `enabled_toolsets` clarifications
- `curator.md`: `hermes curator status` usage rankings documented
- `fallback-providers.md`: config paths corrected
- `hooks.md`: corrections
- `plugins.md`: `ctx.dispatch_tool()` documented in capabilities table
- `tools.md`: corrections
- `tts.md`: per-provider `max_text_length` caps documented
- `voice-mode.md`: Doubao speech integration documented (if Doubao was released in v0.12.0 — verify)

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add features/ browser.md cron.md curator.md fallback-providers.md hooks.md plugins.md tools.md tts.md voice-mode.md
git commit -m "docs(features): sync ACP, browser, cron, curator, TTS, voice, plugins corrections"
```

---

### Task 6: Messaging Updates

**Files:**
- Modify: `messaging/feishu.md`
- Modify: `messaging/README.md` or `messaging/index.md`
- Modify: `messaging/open-webui.md`
- Modify: `messaging/teams.md`
- Modify: `messaging/telegram.md`

- [ ] **Step 1: Diff messaging files**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/user-guide/messaging/
```

- [ ] **Step 2: Read docs counterparts**

```bash
cat ~/src/hermes/hermes-agent-docs/messaging/feishu.md
cat ~/src/hermes/hermes-agent-docs/messaging/telegram.md
cat ~/src/hermes/hermes-agent-docs/messaging/teams.md
cat ~/src/hermes/hermes-agent-docs/messaging/open-webui.md
```

- [ ] **Step 3: Apply corrections**

Key changes:
- `telegram.md`: group chat troubleshooting fixes
- `teams.md`: Teams platform docs corrections
- `feishu.md`: PLATFORM_HINTS corrections
- `open-webui.md`: Open WebUI bootstrap script documented
- `messaging/index.md` or `README.md`: platform count and naming aligned with code (18 native + 1 plugin)

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add messaging/
git commit -m "docs(messaging): sync Telegram, Teams, Feishu, Open WebUI corrections from upstream"
```

---

### Task 7: User Guide Core and Docker

**Files:**
- Modify: `user-guide/configuration.md`
- Modify: `user-guide/configuring-models.md`
- Modify: `docker.md`
- Modify: `sessions.md`
- Modify: `user-guide/windows-wsl-quickstart.md`

- [ ] **Step 1: Diff user-guide core files**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/user-guide/configuration.md
git diff v2026.4.30..HEAD -- website/docs/user-guide/configuring-models.md
git diff v2026.4.30..HEAD -- website/docs/user-guide/docker.md
git diff v2026.4.30..HEAD -- website/docs/user-guide/sessions.md
git diff v2026.4.30..HEAD -- website/docs/user-guide/windows-wsl-quickstart.md
```

- [ ] **Step 2: Read docs counterparts**

Check if these live at `user-guide/` or top-level in the docs repo:
```bash
ls ~/src/hermes/hermes-agent-docs/user-guide/
cat ~/src/hermes/hermes-agent-docs/docker.md
cat ~/src/hermes/hermes-agent-docs/sessions.md
```

- [ ] **Step 3: Apply corrections**

Key changes:
- `docker.md`: `API_SERVER_*` env vars for OpenAI-compatible endpoint exposure; section on connecting to local inference servers (vLLM, Ollama)
- `configuration.md`: fallback provider config paths corrected, `display.language` for i18n documented
- `configuring-models.md`: custom model aliases for `/model` command documented
- `sessions.md`: corrections

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add user-guide/ docker.md sessions.md
git commit -m "docs(user-guide): sync docker, configuration, configuring-models, sessions corrections"
```

---

### Task 8: Skills Docs Updates

**Files:**
- Modify: skill pages under `user-guide/skills/bundled/`

- [ ] **Step 1: Find skills directory structure in docs repo**

```bash
find ~/src/hermes/hermes-agent-docs -path "*/skills/*" -name "*.md" | sort
```

- [ ] **Step 2: Diff skills in source**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/user-guide/skills/
```

- [ ] **Step 3: Apply corrections**

Key changes:
- `note-taking-obsidian.md`: Obsidian file workflows modernized
- `creative-comfyui.md`: ComfyUI v5 docs (ask local vs cloud FIRST)
- `devops-kanban-orchestrator.md`: kanban orchestrator docs
- `devops-kanban-worker.md`: kanban worker docs
- `autonomous-ai-agents-codex.md`: Codex auth prerequisite clarification
- `autonomous-ai-agents-hermes-agent.md`: corrections

Create/update mirror paths in docs repo matching the upstream structure.

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add user-guide/skills/
git commit -m "docs(skills): sync bundled skill pages from upstream"
```

---

### Task 9: Index and Integrations

**Files:**
- Modify: `index.md`
- Modify: `integrations/index.md` (or `integrations/`)
- Modify: `providers.md` or `integrations/providers.md`

- [ ] **Step 1: Diff these files**

```bash
cd ~/src/hermes/hermes-agent
git diff v2026.4.30..HEAD -- website/docs/index.md
git diff v2026.4.30..HEAD -- website/docs/integrations/
```

- [ ] **Step 2: Read docs counterparts**

```bash
cat ~/src/hermes/hermes-agent-docs/index.md
ls ~/src/hermes/hermes-agent-docs/integrations/
cat ~/src/hermes/hermes-agent-docs/integrations/providers.md 2>/dev/null || echo "MISSING"
```

- [ ] **Step 3: Apply corrections**

Key changes:
- `index.md`: platform/LOC/test counts refreshed; clarify gateway vs plugin platforms
- `integrations/providers.md`: Together/Groq/Perplexity cookbook via `custom_providers`; terminal-backend count and naming aligned

- [ ] **Step 4: Commit**

```bash
cd ~/src/hermes/hermes-agent-docs
git add index.md integrations/
git commit -m "docs(index,integrations): refresh platform counts and providers cookbook"
```

---

### Task 10: Final audit, push, and update audit artifacts

- [ ] **Step 1: Run a final diff check**

```bash
# Verify no obviously stale content remains
grep -r "v0.11.0\|v2026.4.23" ~/src/hermes/hermes-agent-docs/ --include="*.md" \
  | grep -v "changelog\|RELEASE\|review_plan\|release_evaluation\|repo_review\|documentation_integrity\|source_validation\|history\|Full Changelog\|Since" \
  | head -20
```

- [ ] **Step 2: Update documentation integrity findings**

Create `documentation_integrity_findings_v2026.5.6.md` with:
- List of files updated in this pass
- Source: post-v0.12.0 upstream doc corrections
- Note: bounded to v0.12.0 released features

- [ ] **Step 3: Final commit with audit artifact**

```bash
cd ~/src/hermes/hermes-agent-docs
git add documentation_integrity_findings_v2026.5.6.md
git commit -m "docs: post-v0.12.0 upstream doc corrections sync (2026-05-06)"
```

- [ ] **Step 4: Push to origin/main**

```bash
cd ~/src/hermes/hermes-agent-docs
git push origin main
```

Expected: clean push to remote.

---

## Self-Review

**Spec coverage check:**
- [x] Developer guide — adding-providers fast path (plugin-based providers)
- [x] Getting started — quickstart, nix-setup, updating corrections
- [x] Guides — cron-script-only (new), automate-with-cron (context_from), aws-bedrock (IAM fix), plugins (dispatch_tool)
- [x] Reference — CLI commands (skills reset, hermes import, --deliver-only), env vars, skills catalogs
- [x] Features — ACP, browser (WSL), cron, curator, hooks, plugins, TTS (max_text_length), voice
- [x] Messaging — Telegram, Teams, Feishu, Open WebUI corrections
- [x] User-guide — docker (API_SERVER_* env vars), configuration (display.language), configuring-models (model aliases)
- [x] Skills — ComfyUI, Obsidian, kanban worker/orchestrator, Codex, hermes-agent skill pages
- [x] Index/integrations — platform counts, providers cookbook

**Post-tag features excluded:**
- goals.md — unreleased
- kanban-tutorial.md — unreleased tutorial
- user-stories.mdx — mdx format, standalone docs don't use it

**Placeholder scan:** None found.

**Type consistency:** All file paths reference actual files in the docs repo or files that will be created.
