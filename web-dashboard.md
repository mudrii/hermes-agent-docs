
# Web Dashboard

The web dashboard is a browser-based UI for managing your Hermes Agent installation. Instead of editing YAML files or running CLI commands, you can configure settings, manage API keys, and monitor sessions from a clean web interface.

## Quick Start

```bash
hermes dashboard
```

This starts a local web server and opens `http://127.0.0.1:9119` in your browser. The dashboard runs entirely on your machine — no data leaves localhost.

### Options

| Flag | Default | Description |
|------|---------|-------------|
| `--port` | `9119` | Port to run the web server on |
| `--host` | `127.0.0.1` | Bind address |
| `--no-open` | — | Don't auto-open the browser |

```bash
# Custom port
hermes dashboard --port 8080

# Bind to all interfaces (use with caution on shared networks)
hermes dashboard --host 0.0.0.0

# Start without opening browser
hermes dashboard --no-open
```

## Prerequisites

The web dashboard requires FastAPI and Uvicorn. Install them with:

```bash
pip install hermes-agent[web]
```

If you installed with `pip install hermes-agent[all]`, the web dependencies are already included.

When you run `hermes dashboard` without the dependencies, it will tell you what to install. If the frontend hasn't been built yet and `npm` is available, it builds automatically on first launch.

## Pages

### Status

The landing page shows a live overview of your installation:

- **Agent version** and release date
- **Gateway status** — running/stopped, PID, connected platforms and their state
- **Active sessions** — count of sessions active in the last 5 minutes
- **Recent sessions** — list of the 20 most recent sessions with model, message count, token usage, and a preview of the conversation

The status page auto-refreshes every 5 seconds.

### Config

A form-based editor for `config.yaml`. All 150+ configuration fields are auto-discovered from `DEFAULT_CONFIG` and organized into tabbed categories:

- **model** — default model, provider, base URL, reasoning settings
- **terminal** — backend (local/docker/ssh/modal), timeout, shell preferences
- **display** — skin, tool progress, resume display, spinner settings
- **agent** — max iterations, gateway timeout, service tier
- **delegation** — subagent limits, reasoning effort
- **memory** — provider selection, context injection settings
- **approvals** — dangerous command approval mode (ask/yolo/deny)
- And more — every section of config.yaml has corresponding form fields

Fields with known valid values (terminal backend, skin, approval mode, etc.) render as dropdowns. Booleans render as toggles. Everything else is a text input.

**Actions:**

- **Save** — writes changes to `config.yaml` immediately
- **Reset to defaults** — reverts all fields to their default values (doesn't save until you click Save)
- **Export** — downloads the current config as JSON
- **Import** — uploads a JSON config file to replace the current values

:::tip
Config changes take effect on the next agent session or gateway restart. The web dashboard edits the same `config.yaml` file that `hermes config set` and the gateway read from.
:::

### API Keys

Manage the `.env` file where API keys and credentials are stored. Keys are grouped by category:

- **LLM Providers** — OpenRouter, Anthropic, OpenAI, DeepSeek, etc.
- **Tool API Keys** — Browserbase, Firecrawl, Tavily, ElevenLabs, etc.
- **Messaging Platforms** — Telegram, Discord, Slack bot tokens, etc.
- **Agent Settings** — non-secret env vars like `API_SERVER_ENABLED`

Each key shows:
- Whether it's currently set (with a redacted preview of the value)
- A description of what it's for
- A link to the provider's signup/key page
- An input field to set or update the value
- A delete button to remove it

Advanced/rarely-used keys are hidden by default behind a toggle.

### Sessions

Browse and inspect all agent sessions. Each row shows the session title, source platform icon (CLI, Telegram, Discord, Slack, cron), model name, message count, tool call count, and how long ago it was active. Live sessions are marked with a pulsing badge.

- **Search** — full-text search across all message content using FTS5. Results show highlighted snippets and auto-scroll to the first matching message when expanded.
- **Expand** — click a session to load its full message history. Messages are color-coded by role (user, assistant, system, tool) and rendered as Markdown with syntax highlighting.
- **Tool calls** — assistant messages with tool calls show collapsible blocks with the function name and JSON arguments.
- **Delete** — remove a session and its message history with the trash icon.

### Logs

View agent, gateway, and error log files with filtering and live tailing.

- **File** — switch between `agent`, `errors`, and `gateway` log files
- **Level** — filter by log level: ALL, DEBUG, INFO, WARNING, or ERROR
- **Component** — filter by source component: all, gateway, agent, tools, cli, or cron
- **Lines** — choose how many lines to display (50, 100, 200, or 500)
- **Auto-refresh** — toggle live tailing that polls for new log lines every 5 seconds
- **Color-coded** — log lines are colored by severity (red for errors, yellow for warnings, dim for debug)

### Analytics

Usage and cost analytics computed from session history. Select a time period (7, 30, or 90 days) to see:

- **Summary cards** — total tokens (input/output), cache hit percentage, total estimated or actual cost, and total session count with daily average
- **Daily token chart** — stacked bar chart showing input and output token usage per day, with hover tooltips showing breakdowns and cost
- **Daily breakdown table** — date, session count, input tokens, output tokens, cache hit rate, and cost for each day
- **Per-model breakdown** — table showing each model used, its session count, token usage, and estimated cost

### Cron

Create and manage scheduled cron jobs that run agent prompts on a recurring schedule.

- **Create** — fill in a name (optional), prompt, cron expression (e.g. `0 9 * * *`), and delivery target (local, Telegram, Discord, Slack, or email)
- **Job list** — each job shows its name, prompt preview, schedule expression, state badge (enabled/paused/error), delivery target, last run time, and next run time
- **Pause / Resume** — toggle a job between active and paused states
- **Trigger now** — immediately execute a job outside its normal schedule
- **Delete** — permanently remove a cron job

### Skills

Browse, search, and toggle skills and toolsets. Skills are loaded from `~/.hermes/skills/` and grouped by category.

- **Search** — filter skills and toolsets by name, description, or category
- **Category filter** — click category pills to narrow the list (e.g. MLOps, MCP, Red Teaming, AI)
- **Toggle** — enable or disable individual skills with a switch. Changes take effect on the next session.
- **Toolsets** — a separate section shows built-in toolsets (file operations, web browsing, etc.) with their active/inactive status, setup requirements, and list of included tools

:::warning Security
The web dashboard reads and writes your `.env` file, which contains API keys and secrets. It binds to `127.0.0.1` by default — only accessible from your local machine. If you bind to `0.0.0.0`, anyone on your network can view and modify your credentials. The dashboard has no authentication of its own.
:::

## `/reload` Slash Command

The dashboard PR also adds a `/reload` slash command to the interactive CLI. After changing API keys via the web dashboard (or by editing `.env` directly), use `/reload` in an active CLI session to pick up the changes without restarting:

```
You → /reload
  Reloaded .env (3 var(s) updated)
```

This re-reads `~/.hermes/.env` into the running process's environment. Useful when you've added a new provider key via the dashboard and want to use it immediately.

## REST API

The web dashboard exposes a REST API that the frontend consumes. You can also call these endpoints directly for automation:

### GET /api/status

Returns agent version, gateway status, platform states, and active session count.

### GET /api/sessions

Returns the 20 most recent sessions with metadata (model, token counts, timestamps, preview).

### GET /api/config

Returns the current `config.yaml` contents as JSON.

### GET /api/config/defaults

Returns the default configuration values.

### GET /api/config/schema

Returns a schema describing every config field — type, description, category, and select options where applicable. The frontend uses this to render the correct input widget for each field.

### PUT /api/config

Saves a new configuration. Body: `{"config": {...}}`.

### GET /api/env

Returns all known environment variables with their set/unset status, redacted values, descriptions, and categories.

### PUT /api/env

Sets an environment variable. Body: `{"key": "VAR_NAME", "value": "secret"}`.

### DELETE /api/env

Removes an environment variable. Body: `{"key": "VAR_NAME"}`.

### GET /api/sessions/\{session_id\}

Returns metadata for a single session.

### GET /api/sessions/\{session_id\}/messages

Returns the full message history for a session, including tool calls and timestamps.

### GET /api/sessions/search

Full-text search across message content. Query parameter: `q`. Returns matching session IDs with highlighted snippets.

### DELETE /api/sessions/\{session_id\}

Deletes a session and its message history.

### GET /api/logs

Returns log lines. Query parameters: `file` (agent/errors/gateway), `lines` (count), `level`, `component`.

### GET /api/analytics/usage

Returns token usage, cost, and session analytics. Query parameter: `days` (default 30). Response includes daily breakdowns and per-model aggregates.

### GET /api/cron/jobs

Returns all configured cron jobs with their state, schedule, and run history.

### POST /api/cron/jobs

Creates a new cron job. Body: `{"prompt": "...", "schedule": "0 9 * * *", "name": "...", "deliver": "local"}`.

### POST /api/cron/jobs/\{job_id\}/pause

Pauses a cron job.

### POST /api/cron/jobs/\{job_id\}/resume

Resumes a paused cron job.

### POST /api/cron/jobs/\{job_id\}/trigger

Immediately triggers a cron job outside its schedule.

### DELETE /api/cron/jobs/\{job_id\}

Deletes a cron job.

### GET /api/skills

Returns all skills with their name, description, category, and enabled status.

### PUT /api/skills/toggle

Enables or disables a skill. Body: `{"name": "skill-name", "enabled": true}`.

### GET /api/tools/toolsets

Returns all toolsets with their label, description, tools list, and active/configured status.

## CORS

The web server restricts CORS to localhost origins only:

- `http://localhost:9119` / `http://127.0.0.1:9119` (production)
- `http://localhost:3000` / `http://127.0.0.1:3000`
- `http://localhost:5173` / `http://127.0.0.1:5173` (Vite dev server)

If you run the server on a custom port, that origin is added automatically.

## Development

If you're contributing to the web dashboard frontend:

```bash
# Terminal 1: start the backend API
hermes dashboard --no-open

# Terminal 2: start the Vite dev server with HMR
cd web/
npm install
npm run dev
```

The Vite dev server at `http://localhost:5173` proxies `/api` requests to the FastAPI backend at `http://127.0.0.1:9119`.

The frontend is built with React 19, TypeScript, Tailwind CSS v4, and shadcn/ui-style components. Production builds output to `hermes_cli/web_dist/` which the FastAPI server serves as a static SPA.

## Automatic Build on Update

When you run `hermes update`, the web frontend is automatically rebuilt if `npm` is available. This keeps the dashboard in sync with code updates. If `npm` isn't installed, the update skips the frontend build and `hermes dashboard` will build it on first launch.

## Themes

The dashboard supports visual themes that change colors, overlay effects, and overall feel. Switch themes live from the header bar — click the palette icon next to the language switcher.

### Built-in Themes

| Theme | Description |
|-------|-------------|
| **Hermes Teal** | Classic dark teal (default) |
| **Midnight** | Deep blue-violet with cool accents |
| **Ember** | Warm crimson and bronze |
| **Mono** | Clean grayscale, minimal |
| **Cyberpunk** | Neon green on black |
| **Rosé** | Soft pink and warm ivory |

Theme selection is persisted to `config.yaml` under `dashboard.theme` and restored on page load.

### Custom Themes

Create a YAML file in `~/.hermes/dashboard-themes/`:

```yaml
# ~/.hermes/dashboard-themes/ocean.yaml
name: ocean
label: Ocean
description: Deep sea blues with coral accents

colors:
  background: "#0a1628"
  foreground: "#e0f0ff"
  card: "#0f1f35"
  card-foreground: "#e0f0ff"
  primary: "#ff6b6b"
  primary-foreground: "#0a1628"
  secondary: "#152540"
  secondary-foreground: "#e0f0ff"
  muted: "#1a2d4a"
  muted-foreground: "#7899bb"
  accent: "#1f3555"
  accent-foreground: "#e0f0ff"
  destructive: "#fb2c36"
  destructive-foreground: "#fff"
  success: "#4ade80"
  warning: "#fbbf24"
  border: "color-mix(in srgb, #ff6b6b 15%, transparent)"
  input: "color-mix(in srgb, #ff6b6b 15%, transparent)"
  ring: "#ff6b6b"
  popover: "#0f1f35"
  popover-foreground: "#e0f0ff"

overlay:
  noiseOpacity: 0.08
  noiseBlendMode: color-dodge
  warmGlowOpacity: 0.15
  warmGlowColor: "rgba(255,107,107,0.2)"
```

The 21 color tokens map directly to the CSS custom properties used throughout the dashboard. All fields are required for custom themes. The `overlay` section is optional — it controls the grain texture and ambient glow effects.

Refresh the dashboard after creating the file. Custom themes appear in the theme picker alongside built-ins.

### Theme API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/dashboard/themes` | GET | List available themes + active name |
| `/api/dashboard/theme` | PUT | Set active theme. Body: `{"name": "midnight"}` |

---

## v0.11.0 Dashboard Polish

### React-Router Sidebar Layout

The dashboard's flat top-bar tab strip is replaced with a persistent left **sidebar** powered by `react-router`. Pages (Status, Config, API Keys, Sessions, Logs, Analytics, Cron, Skills) keep their state across navigation, deep links work for every page (`/sessions/<id>`, `/logs?level=ERROR`), and back/forward buttons behave correctly. The sidebar collapses to an icon rail on narrow widths.

### Mobile-Responsive Layout

The dashboard now renders cleanly down to ~375px wide. The sidebar collapses to a slide-out drawer behind a hamburger button, the analytics charts and per-day tables become horizontally scrollable, and form-heavy pages (Config, API Keys) stack their two-column layouts vertically. Nothing about the desktop experience changes — only the breakpoints below `md`.

### EN + ZH Language Switcher

A language switcher in the header toggles all dashboard chrome between **English** and **简体中文 (Simplified Chinese)**. The selection is persisted to `dashboard.language` in `config.yaml` and applied immediately via i18n string tables — no reload. Translations cover navigation labels, button text, page descriptions, table headers, status badges, and toast messages. Free-form data (session titles, user messages, log lines, custom skill names) is unchanged.

### Vercel Deployment

The frontend can now be built and deployed as a static site to Vercel (or any static host) and pointed at a remote Hermes Agent backend via `VITE_API_BASE_URL`. The Vercel build skips the FastAPI bundling step and emits only `dist/` for the SPA. Use this when you want a public URL for a self-hosted agent — but remember the **Security** warning above: the dashboard has no built-in authentication, so put it behind your own auth proxy if it leaves localhost.

### One-Click Update + Gateway Restart

The Status page gains two buttons:

- **Update Hermes Agent** — runs `pip install -U hermes-agent` (or the equivalent for your install method) and rebuilds the frontend if `npm` is available. Progress streams into a modal log view; the button is disabled until the update finishes.
- **Restart Gateway** — sends `SIGTERM` to the gateway process (PID shown in the Status panel), waits for it to exit, then re-spawns it with the same arguments. The gateway state pill turns yellow (restarting) then green (running) without a page reload.

### Per-Session API Call Tracking

The Sessions list and the Analytics page now show real per-session **API call counts** sourced from the agent's request ledger, not estimated from message counts. This lets you see, e.g., that a session with 12 user turns issued 47 model API calls because of tool-use loops. The breakdown is exposed via `/api/sessions/{id}` (new `api_calls` field) and aggregated in `/api/analytics/usage`.

### Expanded Theme System

The dashboard's live theme switcher (see **[Themes](#themes)** above) is broadened in v0.11.0 to switch **colors, fonts, layout density, and ambient overlays** in place via CSS custom properties — no reload, no flicker. Custom theme YAML files in `~/.hermes/dashboard-themes/` may now also set `font.body`, `font.mono`, `density: compact|comfortable|cozy`, and the existing `overlay` block. CLI skins (which only style the terminal) remain a separate system; see [Skins → Dashboard Themes vs CLI Skins](./skins.md#dashboard-themes-vs-cli-skins-v0110) for the distinction.
