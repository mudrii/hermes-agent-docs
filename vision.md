# Vision

Hermes Agent supports **multimodal vision** -- you can paste images from your clipboard directly into the CLI, send images via messaging platforms, or point the agent at image URLs for analysis.

Vision capabilities have been present since v0.2.0. Image paste in CLI was added in v0.3.0. Vision support across messaging platforms (Telegram, Discord, Matrix) was expanded in v0.4.0. Native Windows image paste was added in v0.8.0.

## Two Vision Surfaces

### Image Paste (CLI)

Paste images from your clipboard into the CLI for the main model to see directly. Images are sent as base64-encoded content blocks in the OpenAI vision format.

### `vision_analyze` Tool

A dedicated tool that downloads an image from a URL and analyzes it using the auxiliary vision provider (which can be a different, cheaper model than your main model). Implemented in `tools/vision_tools.py`.

## CLI Image Paste

### How It Works

1. Copy an image to your clipboard (screenshot, browser image, etc.)
2. Attach it using one of the methods below
3. Type your question and press Enter
4. The image appears as a `[image #1]` badge above the input
5. On submit, the image is sent to the model as a vision content block

You can attach multiple images before sending -- each gets its own badge. Press `Ctrl+C` to clear all attached images.

Images are saved to `~/.hermes/images/` as PNG files with timestamped filenames.

### Paste Methods

| Method | Description | Works in |
|--------|-------------|----------|
| `/paste` | Most reliable. Explicitly reads clipboard. | Everywhere |
| `Ctrl+V` / `Cmd+V` | Works when clipboard has both text and image | Most terminals |
| `Alt+V` | Alt key passes through most terminal emulators | Most terminals (not VSCode) |
| `Ctrl+V` (raw) | Linux desktop only -- Ctrl+V is not paste there | Linux X11/Wayland terminals |

### Platform Compatibility

| Environment | `/paste` | Ctrl+V text+image | Alt+V |
|---|:---:|:---:|:---:|
| macOS Terminal / iTerm2 | Yes | Yes | Yes |
| Windows (native) | Yes | Yes | No |
| Linux X11 desktop | Yes | Yes | Yes |
| Linux Wayland desktop | Yes | Yes | Yes |
| WSL2 (Windows Terminal) | Yes | Yes | Yes |
| VSCode Terminal (local) | Yes | Yes | No |
| VSCode Terminal (SSH) | No | No | No |
| SSH terminal (any) | No | No | No |

### Platform-Specific Setup

**macOS** -- No setup required. Uses `osascript`. Optionally install `pngpaste` for faster performance: `brew install pngpaste`.

**Linux (X11)** -- Install `xclip`: `sudo apt install xclip`

**Linux (Wayland)** -- Install `wl-clipboard`: `sudo apt install wl-clipboard`

**Windows (native)** -- No extra setup required. Hermes uses PowerShell via .NET `System.Windows.Forms.Clipboard`. Tries `powershell` (Windows PowerShell 5.1) first, then `pwsh` (PowerShell 7+). Added in v0.8.0.

**WSL2** -- No extra setup required. Hermes detects WSL2 automatically and uses `powershell.exe` to access the Windows clipboard.

### SSH and Remote Sessions

Clipboard paste does not work over SSH. The Hermes CLI runs on the remote host, and clipboard tools read the remote machine's clipboard, not your local one.

Workarounds:
1. Upload the image file via `scp` or VSCode drag-and-drop, then reference by path
2. Use a URL -- the agent can use `vision_analyze` on any image URL
3. Use X11 forwarding (`ssh -X`) to forward your local clipboard
4. Send images via a messaging platform (Telegram, Discord, Slack, WhatsApp)

## vision_analyze Tool

The `vision_analyze` tool analyzes images from URLs with custom prompts. It uses the centralized auxiliary vision router, which tries OpenRouter, Nous Portal, and the active provider in sequence.

| Parameter | Description |
|-----------|-------------|
| `image_url` | URL of the image to analyze |
| `user_prompt` | Question or analysis request about the image |

The tool downloads images from URLs and converts them to base64 for API compatibility. Automatic temporary file cleanup is handled after analysis.

### Auxiliary Vision Provider

The vision tool uses the auxiliary provider system (not the main model) for analysis. Configure in `config.yaml`:

```yaml
auxiliary:
  vision:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    download_timeout: 30   # HTTP timeout for image download (seconds)
```

The download timeout for fetching images is separate from the LLM API call timeout. It can also be set via the `HERMES_VISION_DOWNLOAD_TIMEOUT` environment variable.

### Vision Auto-Detection (v0.8.0)

In v0.8.0, the vision auto-detection chain was simplified. The resolution order is now:

1. **OpenRouter** — if `OPENROUTER_API_KEY` is set
2. **Nous Portal** — if authenticated via `~/.hermes/auth.json`
3. **Active provider** — the user's configured main chat provider (DeepSeek, Google AI Studio, Alibaba, named custom endpoints, etc.)
4. Stop

Previously the chain included five fixed backends (OpenRouter, Nous, Codex, Anthropic, custom). The new chain is shorter and more predictable, and uses `resolve_provider_client()` for step 3, which handles all provider types including named custom providers.

### Supported Providers for Vision

Any provider reachable through the active provider resolution works for vision tasks. Explicitly supported as known-good vision backends in auto mode: OpenRouter and Nous Portal. Google AI Studio (Gemini), added in v0.8.0 as a first-class provider, works as the active provider fallback.

### Platform Delivery

When images are sent via messaging platforms:

- **Telegram** -- inline images and photo messages are automatically analyzed
- **Discord** -- DM vision + attachment analysis (v0.4.0)
- **Matrix** -- vision support with image caching

## Supported Models

Image paste works with any vision-capable model. The image is sent as a base64-encoded data URL in the OpenAI vision content format:

```json
{
  "type": "image_url",
  "image_url": {
    "url": "data:image/png;base64,..."
  }
}
```

Most modern models support this format, including GPT-4 Vision, Claude (with vision), Gemini, and open-source multimodal models served through OpenRouter.

## Debugging

Enable debug logging for vision tools:

```bash
export VISION_TOOLS_DEBUG=true
```

## Why Terminals Can't Paste Images

Terminals are text-based interfaces. When you press Ctrl+V, the terminal reads the clipboard for text content and wraps it in bracketed paste escape sequences. If the clipboard contains only an image (no text), the terminal has nothing to send. There is no standard terminal escape sequence for binary image data.

Hermes works around this by calling OS-level tools (`osascript`, `powershell.exe`, `xclip`, `wl-paste`) directly via subprocess to read the clipboard independently of the terminal paste event.
