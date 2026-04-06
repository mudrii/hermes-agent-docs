# Repository Review: Hermes Agent source and docs audit on 2026-04-07

## 1) Objective

Review the Hermes Agent source tree and the standalone docs repo in detail, but keep normative claims restricted to the latest published release: `v0.7.0` / `v2026.4.3`.

## 2) Release baseline

- Latest published release: `v2026.4.3`
- Previous release: `v2026.3.30`
- Current `origin/main`: 165 commits ahead of the latest release
- Conclusion: `origin/main` is useful for future-doc planning, but not for release-facing documentation

## 3) Source-tree inventory by subsystem

### Root entrypoints and packaging

- `run_agent.py` is the shared orchestration entrypoint used across CLI, gateway, ACP, cron, and other runtimes
- `cli.py` and `hermes_cli/` provide the user-facing CLI surface and operational commands
- `pyproject.toml`, `requirements.txt`, `Dockerfile`, `flake.nix`, and `packaging/` define installation and packaging surfaces
- Release notes live in `RELEASE_v0.2.0.md` through `RELEASE_v0.7.0.md`

### `agent/`

- Prompt assembly, routing, redaction, model metadata, context compression, and memory coordination live here
- The released `v0.7.0` memory-provider architecture is anchored in this area
- This subsystem is covered in `architecture.md`, `developer-guide/*`, `memory.md`, `credential-pools.md`, and `security.md`

### `hermes_cli/`

- CLI commands, auth flows, config migration, setup wizard, provider selection, profile handling, and gateway management live here
- Post-release `origin/main` adds centralized logging and `hermes logs`; that remains intentionally excluded from release docs
- Coverage lives in `cli-reference.md`, `configuration.md`, `providers.md`, `profiles.md`, and `reference/cli-commands.md`

### `gateway/`

- All messaging adapters and the OpenAI-compatible API server live here
- The released `v0.7.0` window includes API continuity/tool-progress streaming, Discord approval improvements, WhatsApp mention gating, and broader gateway hardening
- Coverage lives in `gateway.md`, `api-server.md`, and `messaging/*`

### `cron/`

- Scheduled-job storage and execution model
- Release-critical behavior is already documented in `cron.md` and `developer-guide/cron-internals.md`

### `acp_adapter/` and `acp_registry/`

- ACP stdio server surface for editor integrations
- Released `v0.7.0` adds client-provided MCP server ingestion
- Coverage lives in `acp.md` and supporting architecture docs

### `plugins/memory/`

- External memory-provider plugins for Honcho, OpenViking, Mem0, Hindsight, Holographic, RetainDB, and ByteRover
- This directory replaced older docs assumptions that memory plugins were still future work
- Coverage lives in `memory.md`, `plugins.md`, and `developer-guide/memory-provider-plugin.md`

### `tools/`

- Core browser, file, terminal, messaging, memory, vision, code-execution, and RL tool surfaces
- The docs repo contains broad tool coverage, but fixed count claims had drifted and were softened in this pass

### `skills/` and `optional-skills/`

- Large bundled skill catalog plus official optional skills
- The docs repo already has strong coverage in `skills.md` and `reference/*skills*`
- Current `origin/main` adds more skills after `v2026.4.3`; those were not documented as released

### `environments/`

- RL and benchmark environments for training and evaluation
- Coverage lives in `environments.md` and `rl-training.md`

### `tests/`

- Broad suite covering agent loop, gateway, CLI, ACP, tools, cron, plugins, and skills
- The released test tree is evidence of subsystem maturity, but this audit did not attempt exhaustive semantic verification of every test assertion

### `website/docs/`

- Public docs-site source
- Valuable as a validation source, but not release-pinned
- Used only after a claim was already confirmed in released code or release notes

## 4) Docs-repo review

### What was already strong

- Wide surface-area coverage across install, configuration, providers, gateway, API server, memory, security, tools, skills, and messaging platforms
- Prior release-bounded audit artifacts already existed for the `v2026.4.3` line

### What was broken or stale

- `README.md` linked many files that do not exist in this repo
- `plugins.md` still described released memory-provider plugins as planned
- `architecture.md`, `developer-guide/architecture.md`, `messaging/README.md`, and several platform pages still framed themselves around `v0.6.0`
- `reference/tools-reference.md` still exposed stale fixed totals

### Corrections applied

- Rebuilt README navigation around real files
- Corrected released memory-provider plugin status
- Refreshed architecture and messaging release framing
- Added a fresh, dated audit set for April 7, 2026

## 5) Online fact-check summary

Primary confirmation sources:

- GitHub latest-release API
- GitHub release page for `v2026.4.3`
- GitHub compare view for `v2026.3.30...v2026.4.3`

Supplementary current-docs confirmation sources:

- Memory Providers page
- Browser Automation page
- API Server page
- Open WebUI page
- ACP page

## 6) Key excluded unreleased work

Excluded from release-facing docs because it landed after `v2026.4.3`:

- native Discord slash-command registration for `/queue`, `/background`, `/btw`
- RetainDB follow-up fixes and API expansion
- centralized logging and `hermes logs`
- MCP OAuth 2.1 PKCE client support
- config-structure validation at startup

## 7) Recommended next step

When the next stable release is published, rerun the same process with a new dated audit set and promote only those post-`v2026.4.3` features that are part of the new published release.
