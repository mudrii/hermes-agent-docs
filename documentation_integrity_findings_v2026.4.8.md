# Documentation Integrity Findings: Hermes Agent v0.8.0 (v2026.4.8)

**Source repo:** `~/src/claw/hermes-agent/`
**Docs repo:** `~/src/claw/hermes-agent-docs/`
**Released version audited:** `v0.8.0` / `v2026.4.8`
**Audit date:** April 9, 2026
**Source tag:** `v2026.4.8`

---

## 1. Stale Version Framing

### Clean

No pages were found framing themselves as `v0.6.0` or `v0.7.0` as the current version. The v0.7.0 docs pass corrected the major stale-version surfaces; this pass found no regressions.

Pages correctly framing v0.6.0 and v0.7.0 as historical introductions (e.g., `profiles.md` "Introduced in v0.6.0", `developer-guide/memory-provider-plugin.md` "introduced in v0.7.0") are accurate historical attributions, not stale framing.

---

## 2. Broken Cross-References

### Clean

A full link audit of `README.md` found no broken `.md` links. All referenced files exist on disk. The v0.7.0 docs pass had rebuilt the README index from scratch; no regressions were introduced in this pass.

New files added in this pass (`logging.md`, `live-model-switching.md`, `notify-on-complete.md`, `documentation_integrity_findings_v2026.4.8.md`) were verified to exist before being added to the index.

---

## 3. Undocumented Released Features

The following v0.8.0 features from `RELEASE_v0.8.0.md` were not yet covered in a dedicated doc page before this pass:

| Feature | Status |
|---------|--------|
| Centralized logging (`~/.hermes/logs/`, `hermes logs`) | **Added** — `logging.md` created in `b06aa27` |
| Live model switching (`/model` command) | **Added** — `live-model-switching.md` created in `76ab251` |
| `notify_on_complete` background process notifications | **Added** — `notify-on-complete.md` created in `76ab251` |
| Google AI Studio native provider (`GOOGLE_AI_STUDIO_API_KEY`) | Covered in `providers.md` and `configuration.md` (updated in `b06aa27`) |
| Supermemory memory provider | Covered in `memory.md` (updated in `d6a724c`) |
| MCP OAuth 2.1 PKCE (tokens at `~/.hermes/mcp-tokens/<server>.json`) | Covered in `mcp.md` (updated in `b19445c`) |
| Matrix platform Tier 1 | Covered in `messaging/matrix.md` (updated in `99fbf54`) |
| Approval buttons on Telegram and Discord | Covered in `gateway.md` and platform docs (updated in `99fbf54`) |
| Inactivity-based cron timeout | Covered in `cron.md` (updated in `b19445c`) |
| Plugin API hooks: `pre_api_request`, `post_api_request`, `on_session_finalize`, `on_session_reset` | Covered in `plugins.md` and `hooks.md` (updated in `b19445c`, `ffa1475`, `09f0acb`) |
| Honcho toolset removed (now a memory provider plugin) | Marked removed in `tools-reference.md`, `toolsets.md`, `toolsets-reference.md` (corrected in `5925f6f`, `4081cda`) |
| Jittered backoff for API retries | Referenced in `configuration.md`; added to changelog in this pass |
| Smart thinking block signature management | Internal reliability improvement; noted in changelog in this pass |

### Remaining gap

`changelog.md` omitted three v0.8.0 items that were called out in the task specification: jittered backoff, smart thinking block signatures, and Honcho removal. These were added to `changelog.md` in this pass (Task 9).

---

## 4. Contradictions Between Files

### Clean

No new contradictions were found between docs files beyond what was already corrected in the v0.8.0 pass. Specific contradiction fixes during this pass include:

- `architecture.md` and `developer-guide/architecture.md`: MiniMax auxiliary model name corrected (`012324f`)
- `providers.md`: aggregator list and DASHSCOPE URL corrected (`8f04151`); Nous Portal auxiliary model corrected (`f2a8203`, `73121b7`)
- `plugins.md`: `required` behavior and `handler_fn` signature corrected (`2797b73`); v0.8.0 hook diagram added (`ffa1475`, `09f0acb`)
- `mcp.md`: OAuth token storage path corrected (`2797b73`)
- `security.md`: escalation table corrected (`2797b73`)
- `agent-loop.md`: three-rule description corrected (`62cbdf5`)
- `context-compression.md`: cooldown description corrected (`62cbdf5`)
- `tools.md`, `toolsets.md`, `reference/toolsets-reference.md`, `reference/tools-reference.md`: Honcho toolset correctly removed (`5925f6f`, `4081cda`)
- Gateway and messaging docs: platform count consistency corrected (`7cddc76`)

---

## 5. Summary of Changes in the v0.8.0 Docs Pass

The v0.8.0 docs pass began after commit `177d0f5` (design spec) and includes 25 commits, all prefixed `docs:`. The full commit list:

| Commit | Description |
|--------|-------------|
| `177d0f5` | Add v0.8.0 documentation review design spec and implementation plan |
| `9d862a0` | Add v0.8.0 review plan, release evaluation, and repo review |
| `d6a724c` | Update memory, sessions, and persistence docs for v0.8.0 |
| `b19445c` | Update MCP (OAuth 2.1 PKCE), plugins, security, and cron docs for v0.8.0 |
| `b06aa27` | Update core architecture and configuration docs for v0.8.0; add logging.md |
| `40f8866` | Fix v0.8.0 audit files — source attribution and prior-audit list |
| `dacf9a5` | Update media, vision, TTS, voice, API server, and skins docs for v0.8.0 |
| `76ab251` | Update CLI, tools, and skills docs for v0.8.0; add notify-on-complete.md |
| `99fbf54` | Update gateway and all messaging platform docs for v0.8.0 |
| `8182ea6` | Add shared thread sessions to session-storage.md for v0.8.0 |
| `62cbdf5` | Fix agent-loop.md, context-compression.md, environments.md for v0.8.0 |
| `2797b73` | Fix plugins.md, mcp.md, security.md for v0.8.0 |
| `8f04151` | Fix provider docs factual errors — aggregator list, MiniMax aux model, DASHSCOPE URL |
| `9b4a81a` | Fix audit file wording |
| `5925f6f` | Remove honcho toolset (v0.8.0); fix safe toolset composition in tools.md |
| `7cddc76` | Fix platform count consistency and malformed anchor in gateway docs |
| `012324f` | Fix MiniMax auxiliary model name in architecture.md |
| `408eb2a` | Fix Task 4 quality issues — double HR, hermes_home kwarg, v0.8.0 status notes |
| `6247ee5` | Fix Task 8 quality issues — vision backends, browser_close deprecation, tts config |
| `4081cda` | Mark honcho toolset as removed in reference/tools-reference.md |
| `f2a8203` | Fix Nous Portal auxiliary model name and aggregator framing in provider docs |
| `93642ec` | Fix Task 2 quality issues — logging gateway explanation, backoff signature, rollback example |
| `ffa1475` | Fix Task 7 quality issues — remove unverifiable PR numbers, add v0.8.0 hooks to plugins diagram |
| `73121b7` | Fix residual 'current aggregator' phrasing and Nous Portal model name in provider docs |
| `09f0acb` | Fix Task 7 quality pass 2 — hook execution order, pre_llm_call description, PR #5613 table |

**Task 9 (this commit):**
- `README.md`: added `logging.md`, `live-model-switching.md`, `notify-on-complete.md` to the Features index; added `documentation_integrity_findings_v2026.4.8.md` to Audit & Validation; added `hermes logs` to the Getting Started quick reference; updated `/model` row in the CLI vs Messaging table
- `changelog.md`: added plugin API-level hooks (`pre_api_request`, `post_api_request`, `on_session_finalize`, `on_session_reset`), jittered backoff, smart thinking block signatures, and Honcho toolset removal to the v0.8.0 section
- `documentation_integrity_findings_v2026.4.8.md`: created (this file)
- `contributing.md`: no changes — the file is version-agnostic

---

## 6. contributing.md Assessment

`contributing.md` does not reference a specific Hermes version for contributing guidelines. The only version-specific note is the `> **v0.5.0 note:**` callout about removed submodules, which is an accurate historical note and should stay. No changes required.

---

## Validation Links

- Release tag: `https://github.com/NousResearch/hermes-agent/releases/tag/v2026.4.8`
- Compare view: `https://github.com/NousResearch/hermes-agent/compare/v2026.4.3...v2026.4.8`
- Release notes: `RELEASE_v0.8.0.md` in source repo
