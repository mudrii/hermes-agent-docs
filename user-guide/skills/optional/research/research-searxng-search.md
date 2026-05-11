---
title: "Searxng Search — Free meta-search via SearXNG — aggregates results from 70+ search engines"
sidebar_label: "Searxng Search"
description: "Free meta-search via SearXNG — aggregates results from 70+ search engines"
---

# Searxng Search

Free meta-search via SearXNG — aggregates results from 70+ search engines. Self-hosted or use a public instance. No API key needed. Falls back automatically when the web search toolset is unavailable.

## Skill metadata

| | |
|---|---|
| Source | Optional — install with `hermes skills install official/research/searxng-search` |
| Path | `optional-skills/research/searxng-search` |
| Version | `1.0.0` |
| Author | hermes-agent |
| License | MIT |
| Platforms | linux, macos |
| Tags | `search`, `searxng`, `meta-search`, `self-hosted`, `free`, `fallback` |
| Related skills | [`duckduckgo-search`](/docs/user-guide/skills/optional/research/research-duckduckgo-search), [`domain-intel`](/docs/user-guide/skills/optional/research/research-domain-intel) |

## Configuration

Set `SEARXNG_URL` to a reachable SearXNG instance:

```bash
SEARXNG_URL=https://searxng.example.com
SEARXNG_URL=http://localhost:8888
```

If no instance is configured, this skill is unavailable and the agent falls back to other search options.

## Detection Flow

Check what is actually available before choosing an approach:

```bash
curl -s --max-time 5 "${SEARXNG_URL}/search?q=test&format=json" | head -c 200
```

1. If `SEARXNG_URL` is set and the instance responds, use SearXNG.
2. If `SEARXNG_URL` is unset or unreachable, fall back to other available search tools.
3. If the user wants SearXNG specifically, help them set up an instance or find a public one.

## CLI Usage

Use `curl` via `terminal` to call the SearXNG JSON API:

```bash
curl -s --max-time 10 \
  "${SEARXNG_URL}/search?q=python+async+programming&format=json&engines=google,bing&limit=10"
```

Useful query parameters include `q`, `format`, `engines`, `limit`, `categories`, `safesearch`, and `time_range`.
