# Extending the CLI

Hermes exposes protected extension hooks on `HermesCLI` so wrapper CLIs can add widgets, keybindings, and layout customizations without overriding the `run()` method. This keeps extensions decoupled from internal changes.

## Extension Points

There are five extension seams available:

| Hook | Purpose | Override when |
|------|---------|---------------|
| `_get_extra_tui_widgets()` | Inject widgets into the layout | You need a persistent UI element (panel, status line, mini-player) |
| `_register_extra_tui_keybindings(kb, *, input_area)` | Add keyboard shortcuts | You need hotkeys (toggle panels, transport controls) |
| `_build_tui_layout_children(**widgets)` | Full control over widget ordering | You need to reorder or wrap existing widgets (rare) |
| `process_command()` | Add custom slash commands | You need `/mycommand` handling |
| `_build_tui_style_dict()` | Custom prompt_toolkit styles | You need custom colors or styling |

## Quick Start: a Wrapper CLI

```python
#!/usr/bin/env python3
from cli import HermesCLI
from prompt_toolkit.layout import FormattedTextControl, Window
from prompt_toolkit.filters import Condition


class MyCLI(HermesCLI):

    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self._panel_visible = False

    def _get_extra_tui_widgets(self):
        cli_ref = self
        return [
            Window(
                FormattedTextControl(lambda: "My custom panel content"),
                height=1,
                filter=Condition(lambda: cli_ref._panel_visible),
            ),
        ]

    def _register_extra_tui_keybindings(self, kb, *, input_area):
        cli_ref = self

        @kb.add("f2")
        def _toggle_panel(event):
            cli_ref._panel_visible = not cli_ref._panel_visible

    def process_command(self, cmd: str) -> bool:
        if cmd.strip().lower() == "/panel":
            self._panel_visible = not self._panel_visible
            state = "visible" if self._panel_visible else "hidden"
            print(f"Panel is now {state}")
            return True
        return super().process_command(cmd)


if __name__ == "__main__":
    cli = MyCLI()
    cli.run()
```

Run it:

```bash
cd ~/.hermes/hermes-agent
source .venv/bin/activate
python my_cli.py
```

## Hook Reference

### _get_extra_tui_widgets()

Returns a list of prompt_toolkit widgets to insert into the TUI layout. Widgets appear between the spacer and the status bar -- above the input area but below the main output.

```python
def _get_extra_tui_widgets(self) -> list:
    return []  # default: no extra widgets
```

Use `ConditionalContainer` or `filter=Condition(...)` to make widgets toggleable.

### _register_extra_tui_keybindings(kb, *, input_area)

Called after Hermes registers its own keybindings and before the layout is built. Add your keybindings to `kb`.

Parameters:
- `kb` -- The `KeyBindings` instance for the prompt_toolkit application
- `input_area` -- The main `TextArea` widget, if you need to read or manipulate user input

Avoid conflicts with built-in keybindings: `Enter` (submit), `Escape Enter` (newline), `Ctrl-C` (interrupt), `Ctrl-D` (exit), `Tab` (auto-suggest accept). Function keys F2+ and Ctrl-combinations are generally safe.

### _build_tui_layout_children(**widgets)

Override this only when you need full control over widget ordering. Most extensions should use `_get_extra_tui_widgets()` instead.

The default layout order:

```
Window(height=0)       # anchor
sudo_widget            # sudo password prompt (conditional)
secret_widget          # secret input prompt (conditional)
approval_widget        # dangerous command approval (conditional)
clarify_widget         # clarify question UI (conditional)
spinner_widget         # thinking spinner (conditional)
spacer                 # fills remaining vertical space
*extra_tui_widgets     # YOUR WIDGETS GO HERE
status_bar             # model/token/context status line
input_rule_top         # border above input
image_bar              # attached images indicator
input_area             # user text input
input_rule_bot         # border below input
voice_status_bar       # voice mode status (conditional)
completions_menu       # autocomplete dropdown
```

## Tips

- **Invalidate the display** after state changes: call `self._invalidate()` to trigger a prompt_toolkit redraw
- **Access agent state**: `self.agent`, `self.model`, `self.conversation_history` are all available
- **Custom styles**: Override `_build_tui_style_dict()` and add entries for your custom style classes
- **Slash commands**: Override `process_command()`, handle your commands, and call `super().process_command(cmd)` for everything else
- **Don't override `run()`** unless absolutely necessary -- the extension hooks exist specifically to avoid that coupling

---

## Plugin CLI Subcommand Registration (v0.8.0)

Plugins can register top-level `hermes <name>` subcommands without touching `main.py`. The `PluginContext.register_cli_command()` method in `hermes_cli/plugins.py` handles the registration.

### How it works

When Hermes starts, `discover_plugins()` loads each enabled plugin and calls its `setup()` entry point, passing a `PluginContext`. The plugin calls `ctx.register_cli_command()` to declare its CLI subcommand. After all plugins are loaded, `main.py` iterates the registered commands and adds them to the argument parser.

### register_cli_command() signature

```python
def register_cli_command(
    self,
    name: str,
    help: str,
    setup_fn: Callable,
    handler_fn: Callable | None = None,
    description: str = "",
) -> None:
    """Register a CLI subcommand (e.g. ``hermes honcho ...``)."""
```

Parameters:

| Parameter | Description |
|-----------|-------------|
| `name` | Subcommand name. The resulting command is `hermes <name>`. |
| `help` | Short help string shown in `hermes --help`. |
| `setup_fn` | Called during argument parser setup. Receives the subparser so the plugin can add its own flags and subcommands. |
| `handler_fn` | Called when the user runs the command. Receives parsed `argparse.Namespace`. Optional — can be set later. |
| `description` | Longer description shown in the subcommand's own `--help`. |

### Example plugin

```python
# my_plugin/__init__.py

def setup(ctx):
    ctx.register_cli_command(
        name="myplugin",
        help="My plugin's CLI commands.",
        setup_fn=_setup_parser,
        handler_fn=_handle_command,
        description="Extended description for hermes myplugin --help.",
    )

def _setup_parser(subparser):
    subparser.add_argument("--verbose", action="store_true")
    sub = subparser.add_subparsers(dest="myplugin_cmd")
    sub.add_parser("run", help="Run the plugin.")
    sub.add_parser("status", help="Show status.")

def _handle_command(args):
    if args.myplugin_cmd == "run":
        print("Running my plugin...")
    elif args.myplugin_cmd == "status":
        print("Status OK")
```

After installing this plugin, users can run:

```bash
hermes myplugin run
hermes myplugin status
hermes myplugin --help
```

### Plugin loading

Plugins are discovered from:
- `~/.hermes/plugins/` (user-installed)
- `./.hermes/plugins/` (project-local, when `HERMES_ENABLE_PROJECT_PLUGINS=true`)
- Pip-installed packages declaring a `hermes_agent.plugins` entry point

Disabled plugins (listed in `config.yaml` under `plugins.disabled`) are skipped during discovery. Use `hermes plugins enable <name>` / `hermes plugins disable <name>` to manage plugin state.
