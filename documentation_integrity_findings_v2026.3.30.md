# Documentation Integrity Findings (Hermes Agent v0.6.0 / v2026.3.30)

## Scope

This file captures the final integrity findings for `hermes-agent-docs` against the released `v2026.3.30` codebase.

## Finding format

- `SEVERITY`: High / Medium / Low
- `AREA`: Core, Gateway, Messaging, MCP, Security, Installation
- `EVIDENCE`: exact doc target plus released source evidence
- `RISK`: operator or reader impact
- `STATUS`: fixed / open

## Findings

1. High — MCP server transport was over-claimed
- `AREA`: MCP
- `EVIDENCE`: `README.md` and `changelog.md` claimed `hermes mcp serve` supported “stdio and Streamable HTTP transports”. Released tag evidence shows stdio-only server wording in `mcp_serve.py`, and CLI wiring exposes only `hermes mcp serve` plus `--verbose`.
- `RISK`: readers may try unsupported transport flags or assume HTTP exposure exists in the released server.
- `STATUS`: **Fixed** in `README.md`, `changelog.md`, and `mcp.md`.

2. High — Approval-timeout config key drift
- `AREA`: Security / Configuration
- `EVIDENCE`: `security.md` and `changelog.md` documented `security.approval_timeout_seconds`, while released code reads `approvals.timeout` and `configuration.md` already documented the correct shape.
- `RISK`: operators using the wrong key will not change timeout behavior.
- `STATUS`: **Fixed** in `security.md` and `changelog.md`.

3. High — Messaging approval flow documented the wrong commands
- `AREA`: Security / Gateway
- `EVIDENCE`: `security.md` described bare-text `yes` / `no` replies, while released gateway approval flow uses `/approve` and `/deny`.
- `RISK`: dangerous-command approvals can fail in chat because users send replies the gateway no longer accepts.
- `STATUS`: **Fixed** in `security.md`.

4. Medium — Profile command surface mixed temporary targeting with sticky default switching
- `AREA`: Core / CLI
- `EVIDENCE`: `README.md` and `changelog.md` implied `switch` semantics from `hermes -p <name>` or `profile switch`, while released CLI uses `hermes profile use <name>` for sticky default and `-p` for per-command targeting.
- `RISK`: users may think `-p` changes the default profile persistently.
- `STATUS`: **Fixed** in `README.md` and `changelog.md`.

5. Medium — Compression defaults drifted from released code
- `AREA`: Core
- `EVIDENCE`: `README.md` and `architecture.md` documented `protect_last_n = 4` and the old `summary_target_tokens` framing, while released code uses `protect_last_n = 20` and ratio-based tuning.
- `RISK`: readers get the wrong mental model for compression pressure and tuning.
- `STATUS`: **Fixed** in `README.md` and `architecture.md`.

6. Medium — BOOT.md hook behavior was mischaracterized
- `AREA`: Gateway / Hooks
- `EVIDENCE`: `gateway.md` described `BOOT.md` as a skill in SKILL.md format, but released code treats it as a built-in startup instruction file executed from a wrapped prompt in the background. `hooks.md` had no BOOT.md coverage at all.
- `RISK`: users may build the file in the wrong format or look in the wrong extension system.
- `STATUS`: **Fixed** in `gateway.md` and `hooks.md`.

7. Medium — Slack multi-workspace release feature was missing from adapter docs
- `AREA`: Messaging
- `EVIDENCE`: released `gateway/platforms/slack.py` loads multiple workspace tokens, including `~/.hermes/slack_tokens.json`, but `messaging/slack.md` only documented single-workspace env setup.
- `RISK`: users miss a released capability and configure Slack more narrowly than necessary.
- `STATUS`: **Fixed** in `messaging/slack.md`.

8. Medium — Feishu group-policy wording overstated open-group behavior
- `AREA`: Messaging
- `EVIDENCE`: `messaging/feishu.md` implied `FEISHU_GROUP_POLICY=open` responds to all group mentions without noting the explicit bot-mention gate that still applies in released code.
- `RISK`: operators may expect unsolicited group responses that never arrive.
- `STATUS`: **Fixed** in `messaging/feishu.md`.

9. Low — Native Windows support remains ambiguous at the release boundary
- `AREA`: Installation
- `EVIDENCE`: the released repository ships `scripts/install.ps1`, but the released README still says native Windows is unsupported and recommends WSL2.
- `RISK`: platform-support expectations remain muddy even after docs corrections.
- `STATUS`: **Open**. `installation.md` was softened to prefer WSL2, but the upstream project should reconcile README and installer support claims.

10. Low — Upstream v0.6.0 release notes themselves over-claim MCP server transport
- `AREA`: Release notes / external source
- `EVIDENCE`: the GitHub release body says `hermes mcp serve` supports both stdio and Streamable HTTP, while the released code surface is stdio-only.
- `RISK`: future doc writers may re-import the release-note wording without checking the tag.
- `STATUS`: **Open**. Recorded here and in the validation matrix; not fixable from this docs repo alone.

## Acceptance statement

This pass removed the highest-impact doc/code mismatches for `v2026.3.30` and added durable audit artifacts for future release reviews. Remaining open items are primarily upstream ambiguity, not local doc drift.
