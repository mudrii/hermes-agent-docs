# Documentation Integrity Findings -- v2026.4.03

**Audit date:** April 3, 2026
**Auditor:** Automated cross-reference against source code + online verification
**Source repo:** `~/src/claw/hermes-agent/` (read-only, commit HEAD as of 2026-04-02)
**Docs repo:** `~/src/claw/hermes-agent-docs/`
**Hermes version audited:** v0.6.0 (v2026.3.30)

---

## Review Scope

- Full source code review of hermes-agent at v2026.3.30 (v0.6.0)
- Cross-reference with hermes-agent-docs
- Online fact-checking against GitHub and docs site
- Continuation of v2026.3.30 integrity review with additional corrections

## Summary

- Documents reviewed: 8
- Corrections made: 4
- New sections added: 2
- New files created: 1

## Critical Corrections

1. **README.md**: Skill count updated from "117 skills (74 bundled + 43 optional)" to "118 skills (96 bundled + 22 optional)" -- reflects actual SKILL.md file counts in source tree
2. **README.md**: Skills doc index entry updated from "74 bundled + 43 optional" to "96 bundled + 22 optional" for consistency
3. **architecture.md**: Skill count in component diagram updated from "74 bundled + 43 optional" to "96 bundled + 22 optional"
4. **architecture.md** (prior review): protect_last_n corrected from 4 to 20
4. **tools.md** (prior review): Web backend priority order corrected
5. **providers.md** (prior review): 4 provider base URLs corrected (MiniMax, MiniMax-CN, Alibaba/DashScope, HuggingFace)

## New Documentation Added

### New Files

1. **developer-guide/memory-provider-plugin.md** -- Placeholder documenting that memory provider plugins (byterover, holographic, honcho-plugin, mem0, openviking, retaindb) are planned for a future release. Notes current v0.6.0 dual-layer architecture (built-in + Honcho) and planned plugin interface.

### New Sections

1. **honcho.md**: Added explicit `set_session_context()` and `_resolve_session_context()` function references to the gateway integration section, clarifying the per-session context threading mechanism.
2. **plugins.md**: Added "Memory Provider Plugins (Planned)" section noting the feature is on the development branch but unreleased in v0.6.0, with link to the new developer guide page.

## Verified Accurate

### memory.md -- All Requested Content Already Present

- **Frozen Snapshot Pattern**: Already documented (lines 50-54). Memory injected once at session start via `load_from_disk()` stored in `_system_prompt_snapshot`. Disk writes don't change system prompt mid-session. Explicitly notes prefix cache preservation.
- **Concurrency**: Already documented (lines 157-165). File lock via `fcntl.flock()` on separate `.lock` file + atomic writes via `tempfile.mkstemp()` + `os.replace()`. `_reload_target()` re-reads under lock before every mutation.
- **Security Scanning**: Already documented (lines 168-179). Lists all blocked patterns: prompt injection, credential exfiltration, secret file reads, SSH backdoor attempts, Hermes env file access, invisible Unicode characters (zero-width spaces, bidirectional overrides, word joiners, byte order marks).

### honcho.md -- All Requested Content Already Present

- **Async Pipeline**: Already documented (lines 174-184). Context prefetched on Turn N for use on Turn N+1, zero HTTP latency on response path, cold start on Turn 1.
- **Gateway Isolation**: Already documented (lines 209-219), now enhanced with explicit function names (`set_session_context()`, `_resolve_session_context()`).
- **Recall Modes**: Already documented (lines 141-149). Three modes: `context` (auto-injected context only, Honcho tools hidden), `tools` (Honcho tools only, no auto-injected context), `hybrid` (both, default).

### plugins.md -- All Requested Content Already Present

- **Plugin Lifecycle Hooks (v0.5.0)**: Already documented (lines 298-374). All six hooks: `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`, `pre_tool_call`, `post_tool_call`. Execution order diagram included. Complete example with all hooks.
- **Plugin Message Injection**: Already documented (lines 475-516) as "Message Injection (v0.6.0)". Covers `ctx.inject_message()` signature, idle vs mid-turn behavior, CLI-only limitation.
- **pre_llm_call context injection**: Already documented (line 296). Hook can return dict with `"context"` key for ephemeral turn-level system prompt injection.

### contributing.md -- All Requested Content Already Present

- **v0.4.0 slash commands**: Lines 613-624 list all 6 commands (/browser, /queue, /permission, /statusbar, /cost, /approve+/deny)
- **v0.5.0 supply chain section**: Lines 517-543 with Tirith, 11 security layers table
- **ACP adapter**: Line 198-199 in project structure
- **Honcho integration**: Lines 202-203 in project structure
- **agent/redact.py**: Line 157 in project structure

### README.md -- Feature Coverage Confirmed

- All 16 messaging platforms listed in doc index (line 316)
- 14 native gateway adapters listed in feature text (line 19, 44) -- correct since open-webui/API server is a separate entry point
- Profiles (v0.6.0) present at line 90
- MCP Server Mode (v0.6.0) present at line 94
- Version references point to v0.6.0 throughout

## Known Remaining Gaps

1. **Skill counts across other docs**: The "118 (96+22)" count has been applied to README.md. Other files (skills.md, architecture.md) may still reference older counts from the prior review's "74+43" correction and should be checked.
2. **Native Windows support**: README says "Windows native is not supported; use WSL2" while `scripts/install.ps1` exists. This ambiguity remains open from the prior review.
3. **MCP server transport**: Upstream v0.6.0 release notes still over-claim Streamable HTTP support for `hermes mcp serve`. Docs correctly say stdio-only. Open from prior review.
4. **Memory provider plugins**: Development branch has 6 implementations but no release timeline documented. Placeholder created.

## Online Verification

- **Docs site**: Live at hermes-agent.nousresearch.com/docs/ (Mintlify-powered)
- **GitHub**: ~22.2k stars, all release tags verified through v0.6.0
- **CONTRIBUTING.md**: Repo version is stale vs the docs version. The docs contributing.md is more current and includes v0.4.0 slash commands, v0.5.0 supply chain section, and complete project structure with ACP adapter, Honcho integration, and agent/redact.py.
- **Discord**: discord.gg/NousResearch confirmed live (Nous Research community server)
- **Skills Hub**: agentskills.io confirmed as the open standard site
