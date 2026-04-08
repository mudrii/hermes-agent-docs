# Browser Automation

Hermes Agent includes a full browser automation toolset with multiple backend options. The `/browser` CLI command was introduced in v0.4.0 ([#2273](https://github.com/NousResearch/hermes-agent/pull/2273), [#1814](https://github.com/NousResearch/hermes-agent/pull/1814)) alongside the underlying browser tool system. The browser toolset must be enabled in your configuration before the tools are available to the model.

## `/browser` CLI Command (v0.4.0)

The `/browser` slash command provides an interactive browser workflow from the CLI without needing to phrase tool calls explicitly. It is available in both CLI and gateway modes.

```
/browser                          Start an interactive browser session
/browser connect                  Attach to Chrome at ws://localhost:9222
/browser connect ws://host:port   Attach to a specific CDP endpoint
/browser status                   Show current connection and active sessions
/browser disconnect               Detach CDP connection, return to local/cloud mode
```

**Interactive session workflow:**

1. Type `/browser` and provide a starting URL or task description
2. The agent calls `browser_navigate` to load the page
3. Call `browser_snapshot` to read the accessibility tree and get ref IDs
4. Use `browser_click`, `browser_type`, `browser_press`, etc. to interact
5. Call `browser_vision` for screenshot-based visual analysis when the accessibility tree is insufficient
6. Session cleanup is automatic — `browser_close` is a no-op in v0.8.0

The underlying mechanism uses the `agent-browser` CLI (a Node.js wrapper around Chromium) in local mode, or CDP (Chrome DevTools Protocol) when connected via `/browser connect`. Setting `CAMOFOX_URL` routes the same tool surface through the local Camofox REST backend instead. The accessibility tree representation uses ref IDs (`@e1`, `@e2`, ...) for element addressing. Cloud backends (Browserbase, Browser Use, Firecrawl) are selected via `browser.cloud_provider` in config.yaml and bypass the local browser path entirely.

---

## Backend Options

| Backend | How to enable | Use case |
|---------|--------------|----------|
| **Browserbase cloud** | Set `browser.cloud_provider: browserbase` in config.yaml + `BROWSERBASE_API_KEY` | Managed cloud browsers with anti-bot, CAPTCHA solving, residential proxies |
| **Browser Use cloud** | Set `browser.cloud_provider: browser-use` in config.yaml + `BROWSER_USE_API_KEY` | Alternative cloud browser provider |
| **Firecrawl cloud** | Set `browser.cloud_provider: firecrawl` in config.yaml + `FIRECRAWL_API_KEY` | Cloud browser via Firecrawl's CDP (v0.8.0) |
| **Camofox local** | Set `CAMOFOX_URL=http://localhost:9377` | Local anti-detection browsing with persistent sessions and VNC debugging |
| **Local Chrome via CDP** | Run `/browser connect` in CLI or set `BROWSER_CDP_URL` env var | Attach to your own Chrome instance; free, real-time |
| **Local Chromium** | Install `agent-browser` CLI (default when no cloud provider is configured) | Local Chromium driven by `agent-browser`; no credentials required |

The cloud provider is selected via the `browser.cloud_provider` key in `~/.hermes/config.yaml`. When the key is absent or empty, local mode is used. When `BROWSER_CDP_URL` is set (e.g. via `/browser connect`), it takes priority over both cloud providers and local mode. When `CAMOFOX_URL` is set, all browser operations route through Camofox instead of the local `agent-browser` CLI.

## Overview

Pages are represented as **accessibility trees** — text-based snapshots of page content. Interactive elements are assigned ref IDs (like `@e1`, `@e2`) that the agent uses for clicking and typing. This makes browser automation LLM-friendly without requiring pixel coordinates.

Key capabilities:

- **Multi-provider cloud execution** — Browserbase or Browser Use, no local browser install needed
- **Camofox local anti-detection mode** — local browser sessions backed by a dedicated Camofox service
- **Local Chrome integration** — attach to your running Chrome via CDP (Chrome DevTools Protocol)
- **Built-in stealth** — random fingerprints, CAPTCHA solving, residential proxies (Browserbase)
- **Session isolation** — each task gets its own browser session
- **Automatic cleanup** — inactive sessions are closed after 5 minutes by default
- **Vision analysis** — screenshot plus AI analysis for visual understanding

## Setup

### Enable the toolset

The `browser` toolset must be enabled in your config or via CLI:

```bash
hermes config set toolsets '["hermes-cli", "browser"]'
```

Or add to `~/.hermes/config.yaml`:

```yaml
toolsets:
  - hermes-cli
  - browser
```

### Browserbase cloud mode

1. Set the cloud provider in config:

```yaml
# ~/.hermes/config.yaml
browser:
  cloud_provider: browserbase
```

2. Add credentials to `~/.hermes/.env`:

```bash
BROWSERBASE_API_KEY=***
BROWSERBASE_PROJECT_ID=your-project-id-here
```

Get credentials at [browserbase.com](https://browserbase.com).

### Browser Use cloud mode

1. Set the cloud provider in config:

```yaml
# ~/.hermes/config.yaml
browser:
  cloud_provider: browser-use
```

2. Add your API key to `~/.hermes/.env`:

```bash
BROWSER_USE_API_KEY=***
```

Get your API key at [browser-use.com](https://browser-use.com).

### Firecrawl cloud mode (v0.8.0)

1. Set the cloud provider in config:

```yaml
# ~/.hermes/config.yaml
browser:
  cloud_provider: firecrawl
```

2. Add credentials to `~/.hermes/.env`:

```bash
FIRECRAWL_API_KEY=***
# Optional: session TTL in seconds (default: 300)
FIRECRAWL_BROWSER_TTL=300
```

Get credentials at [firecrawl.dev](https://firecrawl.dev).

All browser tools route through Firecrawl's cloud browser via CDP when this provider is active.

### Camofox local mode

Set `CAMOFOX_URL` in `~/.hermes/.env`:

```bash
CAMOFOX_URL=http://localhost:9377
```

Hermes then routes `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, `browser_vision`, and related browser tools through the Camofox REST service. The integration supports:

- persistent browser sessions keyed to the current task
- optional managed persistence via `browser.camofox.managed_persistence`
- VNC URL discovery returned from Camofox health/status so you can inspect the live browser visually

The upstream service can be run from the published Camofox browser image:

```bash
docker run -p 9377:9377 -e CAMOFOX_PORT=9377 jo-inc/camofox-browser
```

### Local Chrome via CDP (/browser connect)

Connect Hermes browser tools to your own running Chrome instance via CDP. This is useful when you want to see what the agent is doing in real-time, use pages that require your own cookies/sessions, or avoid cloud browser costs.

```
/browser connect              # Connect to Chrome at ws://localhost:9222
/browser connect ws://host:port  # Connect to a specific CDP endpoint
/browser status               # Check current connection
/browser disconnect            # Detach and return to cloud/local mode
```

If Chrome isn't already running with remote debugging enabled, Hermes will attempt to auto-launch it with `--remote-debugging-port=9222`.

To start Chrome manually with CDP enabled:

```bash
# macOS
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --remote-debugging-port=9222

# Linux
google-chrome --remote-debugging-port=9222
```

When connected via CDP, all browser tools (`browser_navigate`, `browser_click`, etc.) operate on your live Chrome instance instead of spinning up a cloud session.

### Local browser mode

If no cloud credentials are set, `CAMOFOX_URL` is unset, and `/browser connect` is not used, Hermes uses the browser tools through a local Chromium installation driven by `agent-browser`:

```bash
npm install -g agent-browser
# Or install locally in the repo:
npm install
```

### Optional Environment Variables

```bash
# Residential proxies for better CAPTCHA solving (default: "true")
BROWSERBASE_PROXIES=true

# Advanced stealth with custom Chromium — requires Scale Plan (default: "false")
BROWSERBASE_ADVANCED_STEALTH=false

# Session reconnection after disconnects — requires paid plan (default: "true")
BROWSERBASE_KEEP_ALIVE=true

# Custom session timeout in milliseconds (default: project default)
BROWSERBASE_SESSION_TIMEOUT=600000

# Inactivity timeout before auto-cleanup in seconds (default: 300)
BROWSER_INACTIVITY_TIMEOUT=300

# Override CDP endpoint directly (set by /browser connect, or manually)
BROWSER_CDP_URL=ws://localhost:9222

# Route browser tools through a local Camofox service
CAMOFOX_URL=http://localhost:9377

# Vision model for browser_vision screenshot analysis
AUXILIARY_VISION_MODEL=

# Model for page snapshot text summarization
AUXILIARY_WEB_EXTRACT_MODEL=
```

### config.yaml Browser Settings

```yaml
browser:
  cloud_provider: browserbase    # "browserbase" | "browser-use" | "firecrawl" (empty = local mode)
  record_sessions: false          # Auto-record browser sessions as WebM video
  inactivity_timeout: 120         # Seconds before auto-cleanup (env var override available)
  allow_private_urls: false       # Allow RFC1918 / loopback URLs on remote backends
  camofox:
    managed_persistence: false
```

`browser.allow_private_urls` only affects remote or cloud browser paths. Local backends (Camofox, local Chromium, and direct CDP-attached browsers) are not subject to the private-URL SSRF block.

## Available Tools

### browser_navigate

Navigate to a URL. Must be called before any other browser tool. Initializes the browser session.

```
Navigate to https://github.com/NousResearch
```

For simple information retrieval, prefer `web_search` or `web_extract` — they are faster and cheaper. Use browser tools when you need to interact with a page (click buttons, fill forms, handle dynamic content).

### browser_snapshot

Get a text-based snapshot of the current page's accessibility tree. Returns interactive elements with ref IDs like `@e1`, `@e2` for use with `browser_click` and `browser_type`.

- **`full=false`** (default): Compact view showing only interactive elements
- **`full=true`**: Complete page content including non-interactive text

Snapshots over 8000 characters are automatically summarized by an LLM.

### browser_click

Click an element identified by its ref ID from a snapshot.

```
Click @e5 to press the "Sign In" button
```

### browser_type

Type text into an input field. Clears the field first, then types the new text.

```
Type "hermes agent" into the search field @e3
```

### browser_scroll

Scroll the page up or down to reveal more content.

### browser_press

Press a keyboard key. Useful for submitting forms or navigation.

```
Press Enter to submit the form
```

Supported keys: `Enter`, `Tab`, `Escape`, `ArrowDown`, `ArrowUp`, and more.

### browser_back

Navigate back to the previous page in browser history.

### browser_get_images

List all images on the current page with their URLs and alt text.

### browser_vision

Take a screenshot and analyze it with vision AI. Use this when text snapshots don't capture important visual information — especially useful for CAPTCHAs, complex layouts, or visual verification.

Parameters:
- **`question`** (required): What you want to know about the page visually
- **`annotate`** (optional, default `false`): When `true`, overlay numbered `[N]` labels on interactive elements. Each `[N]` maps to ref `@eN` for subsequent browser commands. Useful for QA and spatial reasoning about page layout.

The screenshot is saved persistently and the file path is returned alongside the AI analysis. On messaging platforms (Telegram, Discord, Slack, WhatsApp), you can ask the agent to share the screenshot — it will be sent as a native photo attachment via the `MEDIA:` mechanism.

Screenshots are stored in `~/.hermes/browser_screenshots/` and automatically cleaned up after 24 hours.

The vision model used for analysis is controlled by the `AUXILIARY_VISION_MODEL` environment variable. For page snapshot text summarization, the `AUXILIARY_WEB_EXTRACT_MODEL` variable is used.

### browser_console

Get browser console output (log/warn/error messages) and uncaught JavaScript exceptions from the current page. Use `clear=True` to clear the console after reading so subsequent calls only show new messages.

```
Check the browser console for any JavaScript errors
```

### browser_close

Close the browser session and release resources. This tool schema is still exposed for compatibility, but in v0.8.0 the underlying function was removed — cleanup happens automatically via inactivity-based auto-cleanup and the per-session reaper. You do not need to call `browser_close` explicitly.

## Practical Examples

### Filling Out a Web Form

```
User: Sign up for an account on example.com with my email john@example.com

Agent workflow:
1. browser_navigate("https://example.com/signup")
2. browser_snapshot()  → sees form fields with refs
3. browser_type(ref="@e3", text="john@example.com")
4. browser_type(ref="@e5", text="SecurePass123")
5. browser_click(ref="@e8")  → clicks "Create Account"
6. browser_snapshot()  → confirms success
```

### Researching Dynamic Content

```
User: What are the top trending repos on GitHub right now?

Agent workflow:
1. browser_navigate("https://github.com/trending")
2. browser_snapshot(full=true)  → reads trending repo list
3. Returns formatted results
```

## Session Recording

Automatically record browser sessions as WebM video files:

```yaml
# ~/.hermes/config.yaml
browser:
  record_sessions: true  # default: false
```

When enabled, recording starts automatically on the first `browser_navigate` and saves to `~/.hermes/browser_recordings/` when the session closes. Works in both local and cloud (Browserbase) modes. Recordings older than 72 hours are automatically cleaned up.

## Stealth Features (Browserbase)

| Feature | Default | Notes |
|---------|---------|-------|
| Basic Stealth | Always on | Random fingerprints, viewport randomization, CAPTCHA solving |
| Residential Proxies | On | Routes through residential IPs |
| Advanced Stealth | Off | Custom Chromium build, requires Scale Plan |
| Keep Alive | On | Session reconnection after network hiccups |

If paid features aren't available on your plan, Hermes automatically falls back — first disabling `keepAlive`, then proxies — so browsing still works on free plans.

## Session Management

- Each task gets an isolated browser session identified by `task_id`
- Sessions are automatically cleaned up after inactivity (default: 300 seconds / 5 minutes, set by `BROWSER_INACTIVITY_TIMEOUT`)
- A background daemon thread (`browser-cleanup`) checks every 30 seconds for stale sessions
- Emergency cleanup runs on process exit via `atexit` to prevent orphaned sessions
- Cloud sessions are released via the provider's API on cleanup
- Each session gets its own socket directory under `/tmp/agent-browser-<session_name>` (or `$TMPDIR` on Linux) to avoid concurrency conflicts between parallel subagents
- On macOS, `/tmp` is used directly instead of `$TMPDIR` to avoid the 104-byte Unix socket path limit
- Session activity is tracked per-task with `_session_last_activity` under a thread-safe lock

## Security Considerations

- Cloud browser sessions execute in sandboxed environments provided by Browserbase or Browser Use
- CDP mode (`/browser connect`) operates on your real Chrome instance — the agent has full access to your running browser including cookies, local storage, and open tabs
- When using CDP mode, only connect to trusted Chrome instances; the agent can interact with any open page
- Browser automation involves real HTTP requests; web services may log interactions
- Session recording stores video files locally in `~/.hermes/browser_recordings/`; clean these up if they contain sensitive information

## v0.8.0 Changes

| Change | PR | Release |
|--------|-----|---------|
| Firecrawl cloud browser provider | [#5628](https://github.com/NousResearch/hermes-agent/pull/5628) | v0.8.0 |
| `agent-browser` paths with spaces preserved correctly | [#6077](https://github.com/NousResearch/hermes-agent/pull/6077) | v0.8.0 |
| `browser_close` function removed — auto-cleanup handles it | [#5792](https://github.com/NousResearch/hermes-agent/pull/5792) | v0.8.0 |
| Browser Use set as default managed cloud provider | [#5750](https://github.com/NousResearch/hermes-agent/pull/5750) | v0.8.0 |
| JS evaluation via `browser_console` `expression` parameter | [#5303](https://github.com/NousResearch/hermes-agent/pull/5303) | v0.8.0 |

## v0.4.0 and v0.5.0 Changes

| Change | PR | Release |
|--------|-----|---------|
| `/browser` interactive CLI command | [#2273](https://github.com/NousResearch/hermes-agent/pull/2273), [#1814](https://github.com/NousResearch/hermes-agent/pull/1814) | v0.4.0 |
| SSRF protection added to `browser_navigate` | [#3058](https://github.com/NousResearch/hermes-agent/pull/3058) | v0.5.0 |
| Browser command timeout configurable via `browser.command_timeout` in config.yaml | [#2801](https://github.com/NousResearch/hermes-agent/pull/2801) | v0.5.0 |
| `browser_vision` respects `auxiliary.vision.timeout` config | [#2901](https://github.com/NousResearch/hermes-agent/pull/2901) | v0.5.0 |
| 402 insufficient credits error handled gracefully in vision tool | [#2802](https://github.com/NousResearch/hermes-agent/pull/2802) | v0.5.0 |
| Race condition fix in browser session creation | [#1721](https://github.com/NousResearch/hermes-agent/pull/1721) | v0.4.0 |
| macOS Homebrew paths added to browser PATH resolution | [#2713](https://github.com/NousResearch/hermes-agent/pull/2713) | v0.5.0 |

---

## Limitations

- **Text-based interaction** — relies on accessibility tree, not pixel coordinates
- **Snapshot size** — large pages may be truncated or LLM-summarized at 8000 characters
- **Session timeout** — cloud sessions expire based on your provider's plan settings
- **Cost** — cloud sessions consume provider credits. Auto-cleanup handles session teardown; use `/browser connect` for free local browsing.
- **No file downloads** — cannot download files from the browser
- **SSRF protection** — `browser_navigate` blocks requests to RFC-1918 addresses and localhost (added v0.5.0, [#3058](https://github.com/NousResearch/hermes-agent/pull/3058))
