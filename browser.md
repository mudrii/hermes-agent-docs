# Browser Automation

Hermes Agent includes a full browser automation toolset with multiple backend options. The browser toolset must be enabled in your configuration before the tools are available to the model.

## Backend Options

| Backend | How to enable | Use case |
|---------|--------------|----------|
| **Browserbase cloud** | Set `BROWSERBASE_API_KEY` and `BROWSERBASE_PROJECT_ID` | Managed cloud browsers with anti-bot, CAPTCHA solving, residential proxies |
| **Browser Use cloud** | Set `BROWSER_USE_API_KEY` | Alternative cloud browser provider |
| **Local Chrome via CDP** | Run `/browser connect` in CLI | Attach to your own Chrome instance; free, real-time |
| **Local Chromium** | Install `agent-browser` CLI | Local Chromium driven by `agent-browser`; no credentials required |

If both Browserbase and Browser Use credentials are set, Browserbase takes priority.

## Overview

Pages are represented as **accessibility trees** — text-based snapshots of page content. Interactive elements are assigned ref IDs (like `@e1`, `@e2`) that the agent uses for clicking and typing. This makes browser automation LLM-friendly without requiring pixel coordinates.

Key capabilities:

- **Multi-provider cloud execution** — Browserbase or Browser Use, no local browser install needed
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

```bash
# Add to ~/.hermes/.env
BROWSERBASE_API_KEY=***
BROWSERBASE_PROJECT_ID=your-project-id-here
```

Get credentials at [browserbase.com](https://browserbase.com).

### Browser Use cloud mode

```bash
# Add to ~/.hermes/.env
BROWSER_USE_API_KEY=***
```

Get your API key at [browser-use.com](https://browser-use.com).

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

If no cloud credentials are set and `/browser connect` is not used, Hermes uses the browser tools through a local Chromium installation driven by `agent-browser`:

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
```

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

The screenshot is saved persistently and the file path is returned alongside the AI analysis. On messaging platforms (Telegram, Discord, Slack, WhatsApp), you can ask the agent to share the screenshot — it will be sent as a native photo attachment via the `MEDIA:` mechanism.

Screenshots are stored in `~/.hermes/browser_screenshots/` and automatically cleaned up after 24 hours.

### browser_console

Get browser console output (log/warn/error messages) and uncaught JavaScript exceptions from the current page. Use `clear=True` to clear the console after reading so subsequent calls only show new messages.

```
Check the browser console for any JavaScript errors
```

### browser_close

Close the browser session and release resources. Call this when done to free up cloud session quota.

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
7. browser_close()
```

### Researching Dynamic Content

```
User: What are the top trending repos on GitHub right now?

Agent workflow:
1. browser_navigate("https://github.com/trending")
2. browser_snapshot(full=true)  → reads trending repo list
3. Returns formatted results
4. browser_close()
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

- Each task gets an isolated browser session
- Sessions are automatically cleaned up after inactivity (default: 5 minutes, set by `BROWSER_INACTIVITY_TIMEOUT`)
- A background thread checks every 30 seconds for stale sessions
- Emergency cleanup runs on process exit to prevent orphaned sessions
- Browserbase sessions are released via the API (`REQUEST_RELEASE` status)

## Security Considerations

- Cloud browser sessions execute in sandboxed environments provided by Browserbase or Browser Use
- CDP mode (`/browser connect`) operates on your real Chrome instance — the agent has full access to your running browser including cookies, local storage, and open tabs
- When using CDP mode, only connect to trusted Chrome instances; the agent can interact with any open page
- Browser automation involves real HTTP requests; web services may log interactions
- Session recording stores video files locally in `~/.hermes/browser_recordings/`; clean these up if they contain sensitive information

## Limitations

- **Text-based interaction** — relies on accessibility tree, not pixel coordinates
- **Snapshot size** — large pages may be truncated or LLM-summarized at 8000 characters
- **Session timeout** — cloud sessions expire based on your provider's plan settings
- **Cost** — cloud sessions consume provider credits; use `browser_close` when done. Use `/browser connect` for free local browsing.
- **No file downloads** — cannot download files from the browser
