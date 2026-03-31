# Profiles (Multi-Instance Hermes)

Introduced in **v0.6.0 (v2026.3.30)**. ([#3681](https://github.com/NousResearch/hermes-agent/pull/3681))

---

## What are profiles?

A profile is a fully isolated Hermes environment. Each profile gets its own directory under `~/.hermes/profiles/<name>/` that mirrors the layout of the default `~/.hermes/` home — with its own `config.yaml`, `.env`, `SOUL.md`, memory, sessions, skills, cron jobs, state database, gateway PID, and logs.

The default profile is `~/.hermes` itself. Existing installs need no migration.

Use cases: a coding assistant, a personal chat bot, a research agent, a staging/production split — each running independently with different models, API keys, bot tokens, and personalities.

---

## Profile directory layout

```
~/.hermes/profiles/
  coder/
    config.yaml       # model, provider, toolsets
    .env              # API keys, bot tokens
    SOUL.md           # personality and instructions
    state.db          # SQLite session database
    sessions/         # JSON session logs
    memories/         # MEMORY.md, USER.md
    skills/           # bundled + hub-installed skills
    cron/             # scheduled jobs
    logs/             # gateway and service logs
    hooks/            # lifecycle hooks
    pairing/          # gateway DM pairing data
```

---

## Quick start

```bash
hermes profile create coder       # creates profile + "coder" command alias
coder setup                       # configure API keys and model for this profile
coder chat                        # start chatting
```

`coder` is an independent agent. Every hermes subcommand is available: `coder gateway start`, `coder doctor`, `coder tools`, etc.

---

## CLI subcommands

### `hermes profile` (bare)

Shows the currently active profile, its path, model, and a summary of all profiles.

### `hermes profile list`

Lists all profiles with their status (active marker, path, model, gateway running/stopped, skill count).

### `hermes profile create <name>`

Creates a new profile. Options:

| Flag | Description |
|------|-------------|
| `--clone` | Copy `config.yaml`, `.env`, and `SOUL.md` from the current active profile. Same API keys and model; fresh sessions and memory. |
| `--clone-all` | Full copy — config, keys, personality, all memories, sessions, skills, cron jobs. A complete snapshot. |
| `--clone-from <source>` | Clone from a named profile instead of the currently active one. |
| `--no-alias` | Skip creating the wrapper script in `~/.local/bin`. |

Bundled skills are seeded into the new profile automatically (unless `--clone-all` was used, which copies them directly).

### `hermes profile use <name>`

Sets a sticky default profile. Plain `hermes` commands will target that profile until you switch back.

```bash
hermes profile use coder
hermes chat                   # targets coder
hermes profile use default    # switch back
```

### `hermes profile show <name>`

Prints detailed information for a single profile: path, model, provider, gateway status, skill count, and whether `.env` and `SOUL.md` are configured.

### `hermes profile alias <name>`

Manages the wrapper script for a profile.

```bash
hermes profile alias coder                  # create (or recreate) the alias
hermes profile alias coder --name dev       # create alias with a custom name
hermes profile alias coder --remove         # remove the alias
```

### `hermes profile rename <old-name> <new-name>`

Renames a profile directory and updates its wrapper script and service name.

### `hermes profile delete <name>`

Stops the gateway, removes the systemd/launchd service, removes the command alias, and deletes all profile data. Prompts for confirmation by typing the profile name.

```bash
hermes profile delete coder
hermes profile delete coder --yes    # skip confirmation
```

### `hermes profile export <name>`

Exports a profile to a `.tar.gz` archive.

```bash
hermes profile export coder                 # creates coder.tar.gz
hermes profile export coder -o backup.tar.gz
```

The archive contains the full profile directory including `.env` (API keys). Remove or redact `.env` before sharing the archive with others.

### `hermes profile import <archive>`

Imports a profile from a `.tar.gz` archive.

```bash
hermes profile import coder.tar.gz
hermes profile import coder.tar.gz --name staging
```

---

## The `-p` / `--profile` flag

Target a specific profile for any single command without changing the sticky default:

```bash
hermes -p coder chat
hermes --profile=coder doctor
hermes -p coder gateway start
```

The flag is pre-parsed before module imports so that `HERMES_HOME` is set correctly before any path constants are resolved.

---

## Generated CLI wrapper scripts

When a profile is created, a shell wrapper is placed at `~/.local/bin/<name>`:

```bash
#!/bin/sh
exec hermes -p coder "$@"
```

This makes `coder chat`, `coder setup`, `coder gateway start`, and every other subcommand work as first-class commands. The wrapper directory must be on your `PATH` (the installer adds `~/.local/bin` automatically).

---

## Tab completion for profile names

Add the completion line to your shell config:

```bash
# Bash — add to ~/.bashrc
eval "$(hermes completion bash)"

# Zsh — add to ~/.zshrc
eval "$(hermes completion zsh)"
```

Completion covers: profile names after `-p` and `--profile`, profile subcommand names, and top-level hermes commands.

---

## Token lock isolation

Each profile runs its own gateway with its own bot credential. If two profiles accidentally share the same bot token, the second gateway is blocked at startup with a clear error naming the conflicting profile. Token locks are enforced for Telegram, Discord, Slack, WhatsApp, and Signal.

---

## Running gateways per profile

```bash
coder gateway start             # start coder's gateway (separate process)
assistant gateway start         # start assistant's gateway (separate process)
coder gateway install           # register hermes-gateway-coder as a system service
assistant gateway install       # register hermes-gateway-assistant as a system service
```

Each profile's gateway logs, PID file, and service are independent.

---

## Updates propagate to all profiles

`hermes update` pulls code once and then syncs new bundled skills to every profile automatically:

```bash
hermes update
# → Code updated (12 commits)
# → Skills synced: default (up to date), coder (+2 new), assistant (+2 new)
```

User-modified skills are never overwritten.

---

## How it works

The profiles system intercepts `--profile`/`-p` from `sys.argv` before any module imports and sets `HERMES_HOME` to the profile's directory. Since all path resolution in the codebase goes through `get_hermes_home()`, every subsystem — config, sessions, memory, skills, state database, gateway PID, logs, cron — automatically scopes to the profile without any per-subsystem awareness.
