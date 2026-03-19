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
| Banner and response colors | `colors.banner_border`, `colors.banner_title`, `colors.banner_accent`, `colors.response_border` |
| Spinner animation | `spinner.waiting_faces`, `spinner.thinking_faces`, `spinner.thinking_verbs`, `spinner.wings` |
| Branding text | `branding.agent_name`, `branding.welcome`, `branding.response_label`, `branding.prompt_symbol` |
| Tool activity prefix | `tool_prefix` |

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
  banner_border: "#FFD700"          # hex color for banner box border
  banner_title: "#FFA500"           # hex color for the banner title text
  banner_accent: "#FFFF00"          # hex color for accents in the banner
  response_border: "#888888"        # hex color for the response box border

spinner:
  waiting_faces: ["(-.-)Zzz...", "(-.-)Zz...", "(-.-)Z...", "(-.-)..."]
  thinking_faces: ["( ˘ω˘)", "( ˘ω˘ )", "( ˘ω˘  )", "( ˘ω˘   )"]
  thinking_verbs:
    - "thinking"
    - "processing"
    - "reasoning"
  wings:
    - ["✦", "✦"]
    - ["✧", "✧"]

branding:
  agent_name: "My Agent"
  welcome: "Welcome! I am your agent."
  response_label: " My Agent "
  prompt_symbol: "❯"

tool_prefix: "│"
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
