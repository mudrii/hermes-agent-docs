# Updating & Uninstalling

## Updating

Update to the latest version with a single command:

```bash
hermes update
```

This pulls the latest code, updates dependencies, and prompts you to configure any new options added since your last update.

### What happens during an update

1. **Git pull** -- pulls the latest code from the `main` branch and updates submodules
2. **Dependency install** -- runs `uv pip install -e ".[all]"` to pick up new or changed dependencies
3. **Config migration** -- detects new config options added since your version and prompts you to set them
4. **Gateway auto-restart** -- if the gateway service is running (systemd on Linux, launchd on macOS), it is automatically restarted so the new code takes effect immediately

Expected output:

```
$ hermes update
Updating Hermes Agent...
  Pulling latest code...
Already up to date.
  Updating dependencies...
  Dependencies updated
  Checking for new config options...
  Config is up to date
  Restarting gateway service...
  Gateway restarted
  Hermes Agent updated successfully!
```

### Checking your current version

```bash
hermes version
```

Check for available updates without installing:

```bash
hermes update --check
```

### Updating from Messaging Platforms

You can update directly from Telegram, Discord, Slack, or WhatsApp by sending:

```
/update
```

This pulls the latest code, updates dependencies, and restarts the gateway. The bot will briefly go offline during the restart (typically 5-15 seconds).

### Manual Update

If you installed manually (not via the quick installer):

```bash
cd /path/to/hermes-agent
export VIRTUAL_ENV="$(pwd)/venv"

git pull origin main
git submodule update --init --recursive

uv pip install -e ".[all]"
uv pip install -e "./tinker-atropos"

hermes config check
hermes config migrate
```

### Rollback instructions

If an update introduces a problem:

```bash
cd /path/to/hermes-agent

# List recent versions
git log --oneline -10

# Roll back to a specific commit
git checkout <commit-hash>
git submodule update --init --recursive
uv pip install -e ".[all]"

# Restart the gateway if running
hermes gateway restart
```

To roll back to a specific release tag:

```bash
git checkout v0.8.0
git submodule update --init --recursive
uv pip install -e ".[all]"
```

Rolling back may cause config incompatibilities if new options were added. Run `hermes config check` after rolling back and remove any unrecognized options if you encounter errors.

### Note for Nix users

Updates are managed through the Nix package manager:

```bash
nix flake update hermes-agent
nix profile upgrade hermes-agent
```

Nix installations are immutable -- rollback is handled by Nix's generation system:

```bash
nix profile rollback
```

See [Nix Setup](./nix-setup.md) for more details.

---

## Uninstalling

```bash
hermes uninstall
```

The uninstaller gives you the option to keep your configuration files (`~/.hermes/`) for a future reinstall.

### Manual Uninstall

```bash
rm -f ~/.local/bin/hermes
rm -rf /path/to/hermes-agent
rm -rf ~/.hermes            # Optional -- keep if you plan to reinstall
```

If you installed the gateway as a system service, stop and disable it first:

```bash
hermes gateway stop
# Linux: systemctl --user disable hermes-gateway
# macOS: launchctl remove ai.hermes.gateway
```
