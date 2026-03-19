# Skins & Themes

Skins control the **visual presentation** of the Hermes CLI: banner colors, spinner faces and verbs, response-box labels, branding text, and the tool activity prefix. Skins are purely cosmetic — they have no effect on the agent's behavior, capabilities, or conversational personality.

The distinction between skins and personality:

- **Personality** (via `SOUL.md` or `/personality`) changes the agent's tone and wording
- **Skin** (via `/skin` or `display.skin`) changes the CLI's visual appearance

## Changing Skins

### In the CLI

```bash
/skin                # show the current skin and list available skins
/skin ares           # switch to the "ares" built-in skin
/skin mytheme        # switch to a custom skin from ~/.hermes/skins/mytheme.yaml
```

The `/skin` command updates the active CLI theme immediately for the current session.

### Persistent default

Set the default skin in `~/.hermes/config.yaml`:

```yaml
display:
  skin: default
```

## Built-in Skins

| Skin | Description | Agent branding |
|------|-------------|----------------|
| `default` | Classic Hermes — gold and kawaii | `Hermes Agent` |
| `ares` | War-god theme — crimson and bronze | `Ares Agent` |
| `mono` | Monochrome — clean grayscale | `Hermes Agent` |
| `slate` | Cool blue — developer-focused | `Hermes Agent` |
| `poseidon` | Ocean-god theme — deep blue and seafoam | `Poseidon Agent` |
| `sisyphus` | Sisyphean theme — austere grayscale with persistence | `Sisyphus Agent` |
| `charizard` | Volcanic theme — burnt orange and ember | `Charizard Agent` |

Built-in skins are loaded from `hermes_cli/skin_engine.py`.

## What a Skin Can Customize

| Area | Config keys |
|------|-------------|
| Banner and response colors | `colors.banner_border`, `colors.banner_title`, `colors.banner_accent`, `colors.banner_dim`, `colors.banner_text`, `colors.response_border` |
| UI accent colors | `colors.ui_accent`, `colors.ui_label`, `colors.ui_ok`, `colors.ui_error`, `colors.ui_warn` |
| Prompt and input styling | `colors.prompt`, `colors.input_rule`, `colors.session_label`, `colors.session_border` |
| Spinner animation | `spinner.waiting_faces`, `spinner.thinking_faces`, `spinner.thinking_verbs`, `spinner.wings` |
| Branding text | `branding.agent_name`, `branding.welcome`, `branding.goodbye`, `branding.response_label`, `branding.prompt_symbol`, `branding.help_header` |
| Tool activity prefix | `tool_prefix` (default: `"┊"`) |
| Tool emoji overrides | `tool_emojis` (per-tool emoji overrides for spinners and progress) |
| Banner ASCII art | `banner_logo` (Rich-markup ASCII art replacing the default logo) |
| Banner hero art | `banner_hero` (Rich-markup hero art replacing the default caduceus) |

## Creating Custom Skins

Place YAML files in `~/.hermes/skins/`. User skins inherit missing values from the built-in `default` skin, so you only need to specify the values you want to change.

### Minimal custom skin

```yaml
name: cyberpunk
description: Neon terminal theme

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  banner_accent: "#FF1493"

spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "Cyber Agent"
  response_label: " ⚡ Cyber "

tool_prefix: "▏"
```

Save this as `~/.hermes/skins/cyberpunk.yaml` and activate with:

```bash
/skin cyberpunk
```

### Full skin example (all keys)

```yaml
name: my-skin
description: My custom skin

colors:
  banner_border: "#FFD700"          # Panel border color
  banner_title: "#FFA500"           # Panel title text color
  banner_accent: "#FFFF00"          # Section headers (Available Tools, etc.)
  banner_dim: "#B8860B"             # Dim/muted text (separators, labels)
  banner_text: "#FFF8DC"            # Body text (tool names, skill names)
  ui_accent: "#FFBF00"             # General UI accent
  ui_label: "#4dd0e1"              # UI labels
  ui_ok: "#4caf50"                 # Success indicators
  ui_error: "#ef5350"              # Error indicators
  ui_warn: "#ffa726"               # Warning indicators
  prompt: "#FFF8DC"                # Prompt text color
  input_rule: "#CD7F32"            # Input area horizontal rule
  response_border: "#FFD700"       # Response box border (ANSI)
  session_label: "#DAA520"         # Session label color
  session_border: "#8B8682"        # Session ID dim color

spinner:
  waiting_faces: ["(-.-)Zzz...", "(-.-)Zz...", "(-.-)Z...", "(-.-)..."]
  thinking_faces: ["( ˘ω˘)", "( ˘ω˘ )", "( ˘ω˘  )", "( ˘ω˘   )"]
  thinking_verbs:
    - "thinking"
    - "processing"
    - "reasoning"
  wings:                            # Optional left/right spinner decorations
    - ["✦", "✦"]                   # Each entry is [left, right] pair
    - ["✧", "✧"]

branding:
  agent_name: "My Agent"           # Banner title, status display
  welcome: "Welcome! I am your agent."  # Shown at CLI startup
  goodbye: "See you! ✦"            # Shown on exit
  response_label: " ✦ My Agent "   # Response box header label
  prompt_symbol: "✦ ❯ "           # Input prompt symbol
  help_header: "(✦) Available Commands"  # /help header text

tool_prefix: "│"                    # Tool output line prefix (default: ┊)

# Per-tool emoji overrides (used in spinners and progress)
tool_emojis:
  terminal: "✦"
  web_search: "🔮"

# Rich-markup ASCII art for the banner (optional)
# banner_logo: "[bold]MY AGENT[/]"
# banner_hero: "[dim]hero art here[/]"
```

## Operational Notes

- Unknown skins fall back to `default` automatically with no error
- `/skin` activates a skin immediately for the current session only
- Setting `display.skin` in `config.yaml` persists the skin across sessions
- Built-in skins load from `hermes_cli/skin_engine.py`
- User skin files are loaded from `~/.hermes/skins/`

## Banner Customization

The `branding.agent_name` key controls the agent name displayed in the banner and response box. Each built-in skin uses this to present a different identity:

- `default` shows `Hermes Agent`
- `ares` shows `Ares Agent`
- `poseidon` shows `Poseidon Agent`
- `sisyphus` shows `Sisyphus Agent`
- `charizard` shows `Charizard Agent`

Custom skins can set any agent name they like. This is purely cosmetic — the underlying Hermes Agent system is unchanged.

## Separating Skin from Personality

A common pattern is to pair a matching skin with a personality for immersive themed sessions:

```bash
/skin poseidon
/personality creative
```

But they remain independent settings. You can use `ares` skin with `helpful` personality, or `default` skin with a custom `SOUL.md`.

For conversational personality changes, see the Personality documentation. For terminal appearance, use skins.
