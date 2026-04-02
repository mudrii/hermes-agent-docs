# Profile Commands Reference

This page covers all commands related to Hermes profiles. For general CLI commands, see [CLI Commands Reference](./cli-commands.md).

## `hermes profile`

```bash
hermes profile <subcommand>
```

Top-level command for managing profiles. Running `hermes profile` without a subcommand shows help.

| Subcommand | Description |
|------------|-------------|
| `list` | List all profiles. |
| `use` | Set the active (default) profile. |
| `create` | Create a new profile. |
| `delete` | Delete a profile. |
| `show` | Show details about a profile. |
| `alias` | Regenerate the shell alias for a profile. |
| `rename` | Rename a profile. |
| `export` | Export a profile to a tar.gz archive. |
| `import` | Import a profile from a tar.gz archive. |

## `hermes profile list`

```bash
hermes profile list
```

Lists all profiles. The currently active profile is marked with `*`.

Example:

```bash
$ hermes profile list
  default
* work
  dev
  personal
```

## `hermes profile use`

```bash
hermes profile use <name>
```

Sets `<name>` as the active profile. All subsequent `hermes` commands (without `-p`) will use this profile.

| Argument | Description |
|----------|-------------|
| `<name>` | Profile name to activate. Use `default` to return to the base profile. |

Example:

```bash
hermes profile use work
hermes profile use default
```

## `hermes profile create`

```bash
hermes profile create <name> [options]
```

Creates a new profile. Profile names must match `^[a-z0-9][a-z0-9_-]{0,63}$` (alphanumeric, hyphens, underscores, max 64 chars). Each new profile gets bootstrapped directories: memories, sessions, skills, skins, logs, plans, workspace, and cron.

| Argument / Option | Description |
|-------------------|-------------|
| `<name>` | Name for the new profile. |
| `--clone` | Copy `config.yaml`, `.env`, and `SOUL.md` from the current profile. |
| `--clone-all` | Copy everything (config, memories, skills, sessions, state) from the current profile. Runtime files (gateway.pid, processes.json) are stripped. |
| `--clone-from <profile>` | Clone from a specific profile instead of the current one. Used with `--clone` or `--clone-all`. |

Examples:

```bash
# Blank profile -- needs full setup
hermes profile create mybot

# Clone config only from current profile
hermes profile create work --clone

# Clone everything from current profile
hermes profile create backup --clone-all

# Clone config from a specific profile
hermes profile create work2 --clone --clone-from work
```

## `hermes profile delete`

```bash
hermes profile delete <name> [options]
```

Deletes a profile and removes its shell alias. This permanently deletes the profile's entire directory including all config, memories, sessions, and skills. Cannot delete the currently active profile.

| Argument / Option | Description |
|-------------------|-------------|
| `<name>` | Profile to delete. |
| `--yes`, `-y` | Skip confirmation prompt. |

## `hermes profile show`

```bash
hermes profile show <name>
```

Displays details about a profile including its home directory, configured model, active platforms, and disk usage.

Example:

```bash
$ hermes profile show work
Profile:    work
Home:       ~/.hermes/profiles/work
Model:      anthropic/claude-sonnet-4
Platforms:  telegram, discord
Skills:     12 installed
Disk:       48 MB
```

## `hermes profile alias`

```bash
hermes profile alias <name> [options]
```

Regenerates the shell alias script at `~/.local/bin/<name>`. Useful if the alias was accidentally deleted or if you need to update it after moving your Hermes installation.

| Argument / Option | Description |
|-------------------|-------------|
| `<name>` | Profile to create/update the alias for. |
| `--remove` | Remove the wrapper script instead of creating it. |
| `--name <alias>` | Custom alias name (default: profile name). |

Examples:

```bash
hermes profile alias work
# Creates/updates ~/.local/bin/work

hermes profile alias work --name mywork
# Creates ~/.local/bin/mywork

hermes profile alias work --remove
# Removes the wrapper script
```

## `hermes profile rename`

```bash
hermes profile rename <old-name> <new-name>
```

Renames a profile. Updates the directory and shell alias.

| Argument | Description |
|----------|-------------|
| `<old-name>` | Current profile name. |
| `<new-name>` | New profile name. |

Example:

```bash
hermes profile rename mybot assistant
# ~/.hermes/profiles/mybot -> ~/.hermes/profiles/assistant
# ~/.local/bin/mybot -> ~/.local/bin/assistant
```

## `hermes profile export`

```bash
hermes profile export <name> [options]
```

Exports a profile as a compressed tar.gz archive. The default profile (~/.hermes) excludes infrastructure files (repo checkout, worktrees, databases, binaries, auth.json, .env, node_modules) to keep the export portable.

| Argument / Option | Description |
|-------------------|-------------|
| `<name>` | Profile to export. |
| `-o`, `--output <path>` | Output file path (default: `<name>.tar.gz`). |

## `hermes profile import`

```bash
hermes profile import <archive> [options]
```

Imports a profile from a tar.gz archive.

| Argument / Option | Description |
|-------------------|-------------|
| `<archive>` | Path to the tar.gz archive to import. |
| `--name <name>` | Name for the imported profile (default: inferred from archive). |

## `hermes -p` / `hermes --profile`

```bash
hermes -p <name> <command> [options]
hermes --profile <name> <command> [options]
```

Global flag to run any Hermes command under a specific profile without changing the sticky default. This overrides the active profile for the duration of the command. The flag is pre-parsed before any module imports to ensure `HERMES_HOME` is set correctly.

Examples:

```bash
hermes -p work chat -q "Check the server status"
hermes --profile dev gateway start
hermes -p personal skills list
hermes -p work config edit
```

## `hermes completion`

```bash
hermes completion <shell>
```

Generates shell completion scripts. Includes completions for profile names and profile subcommands.

| Argument | Description |
|----------|-------------|
| `<shell>` | Shell to generate completions for: `bash` or `zsh`. |

After installation, tab completion works for:
- `hermes profile <TAB>` -- subcommands (list, use, create, etc.)
- `hermes profile use <TAB>` -- profile names
- `hermes -p <TAB>` -- profile names

## See also

- [CLI Commands Reference](./cli-commands.md)
- [FAQ](./faq.md)
