# Documentation Integrity Findings: Hermes Agent v0.12.0 (v2026.4.30)

## Findings Addressed

1. The docs repo still identified v0.11.0 / v2026.4.23 as current stable. Updated release framing and added the v0.12 changelog section.
2. Curator documentation was absent. Added the first-party Curator page and references for CLI/slash command surfaces.
3. Provider docs omitted or under-described v0.12 providers: LM Studio, GMI Cloud, Azure AI Foundry, MiniMax OAuth, and Tencent Tokenhub. Updated provider and environment/reference surfaces.
4. Messaging docs omitted Yuanbao and Microsoft Teams. Added both pages and refreshed the messaging index.
5. Skills catalogs were stale and still treated promoted skills such as TouchDesigner-MCP as optional. Replaced catalogs with upstream generated v0.12 versions and added generated skill pages.
6. Built-in plugin docs described all bundled plugins as opt-in. Updated from upstream v0.12 docs to distinguish backend/platform/standalone/dashboard plugin behavior.
7. TTS docs missed Piper and custom command providers. Updated from upstream v0.12 docs.
8. Vision, ACP, prompt-cache, redaction, and provider-runtime pages had stale implementation details. Patched to match current source.
9. Post-tag unreleased Kanban, Goals, optional productivity skill, Curator backup/rollback/dry-run, and Telegram `/topic` command claims were removed from stable-release docs after checking the `v2026.4.30` tag.
10. Terminal cwd, Docker backend lifecycle, platform plugin adapter guidance, Teams platform docs, and `HERMES_REDACT_SECRETS` defaults were corrected against release-tag source code and first-party docs.

## Online Check

Public release search was used to confirm `v2026.4.30` / `v0.12.0` as the latest released Hermes Agent version before treating it as the docs target.

## Notes

The standalone docs repo carries upstream generated skill pages under `user-guide/skills/**` where those pages correspond to files present in the `v2026.4.30` source tag. Generated pages for post-tag unreleased skills are intentionally excluded.
