# Nix & NixOS Setup

Hermes Agent ships a Nix flake with three levels of integration:

| Level | Who it's for | What you get |
|-------|-------------|--------------|
| `nix run` / `nix profile install` | Any Nix user (macOS, Linux) | Pre-built binary with all deps -- then use the standard CLI workflow |
| NixOS module (native) | NixOS server deployments | Declarative config, hardened systemd service, managed secrets |
| NixOS module (container) | Agents that need self-modification | Everything above, plus a persistent Ubuntu container where the agent can `apt`/`pip`/`npm install` |

The `curl | bash` installer manages Python, Node, and dependencies itself. The Nix flake replaces all of that -- every Python dependency is a Nix derivation built by uv2nix, and runtime tools (Node.js, git, ripgrep, ffmpeg) are wrapped into the binary's PATH. There is no runtime pip, no venv activation, no `npm install`.

For non-NixOS users, this only changes the install step. Everything after (`hermes setup`, `hermes gateway install`, config editing) works identically to the standard install.

For NixOS module users, the entire lifecycle is different: configuration lives in `configuration.nix`, secrets go through sops-nix/agenix, the service is a systemd unit, and CLI config commands are blocked.

## Prerequisites

- Nix with flakes enabled -- Determinate Nix recommended (enables flakes by default)
- API keys for the services you want to use (at minimum: an OpenRouter or Anthropic key)

## Quick Start (Any Nix User)

No clone needed. Nix fetches, builds, and runs everything:

```bash
# Run directly (builds on first use, cached after)
nix run github:NousResearch/hermes-agent -- setup
nix run github:NousResearch/hermes-agent -- chat

# Or install persistently
nix profile install github:NousResearch/hermes-agent
hermes setup
hermes chat
```

After `nix profile install`, `hermes`, `hermes-agent`, and `hermes-acp` are on your PATH. The workflow is identical to the standard installation -- `hermes setup` walks through provider selection, `hermes gateway install` sets up a launchd (macOS) or systemd user service, and config lives in `~/.hermes/`.

Building from a local clone:

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
nix build
./result/bin/hermes setup
```

## NixOS Module

The flake exports `nixosModules.default` -- a full NixOS service module that declaratively manages user creation, directories, config generation, secrets, documents, and service lifecycle. This module requires NixOS.

### Add the Flake Input

```nix
# /etc/nixos/flake.nix (or your system flake)
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    hermes-agent.url = "github:NousResearch/hermes-agent";
  };

  outputs = { nixpkgs, hermes-agent, ... }: {
    nixosConfigurations.your-host = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        hermes-agent.nixosModules.default
        ./configuration.nix
      ];
    };
  };
}
```

### Minimal Configuration

```nix
{ config, ... }: {
  services.hermes-agent = {
    enable = true;
    settings.model.default = "anthropic/claude-sonnet-4";
    environmentFiles = [ config.sops.secrets."hermes-env".path ];
    addToSystemPackages = true;
  };
}
```

`nixos-rebuild switch` creates the `hermes` user, generates `config.yaml`, wires up secrets, and starts the gateway.

The `environmentFiles` line assumes you have sops-nix or agenix configured. The file should contain at least one LLM provider key. If you don't have a secrets manager yet, you can use a plain file:

```bash
echo "OPENROUTER_API_KEY=sk-or-your-key" | sudo install -m 0600 -o hermes /dev/stdin /var/lib/hermes/env
```

```nix
services.hermes-agent.environmentFiles = [ "/var/lib/hermes/env" ];
```

Setting `addToSystemPackages = true` puts the `hermes` CLI on your system PATH and sets `HERMES_HOME` system-wide so the interactive CLI shares state with the gateway service.

### Verify It Works

```bash
systemctl status hermes-agent
journalctl -u hermes-agent -f
hermes version
hermes config
```

### Choosing a Deployment Mode

| | Native (default) | Container |
|---|---|---|
| How it runs | Hardened systemd service on the host | Persistent Ubuntu container with `/nix/store` bind-mounted |
| Security | `NoNewPrivileges`, `ProtectSystem=strict`, `PrivateTmp` | Container isolation, unprivileged user inside |
| Agent can self-install packages | No | Yes -- `apt`, `pip`, `npm` installs persist across restarts |
| When to choose | Standard deployments, maximum security | Agent needs runtime package installation, mutable environment |

Enable container mode:

```nix
services.hermes-agent = {
  enable = true;
  container.enable = true;
};
```

Container mode auto-enables `virtualisation.docker.enable` via `mkDefault`. For Podman, set `container.backend = "podman"`.

## Configuration

### Declarative Settings

The `settings` option accepts an arbitrary attrset rendered as `config.yaml`. Supports deep merging across multiple module definitions:

```nix
# base.nix
services.hermes-agent.settings = {
  model.default = "anthropic/claude-sonnet-4";
  toolsets = [ "all" ];
  terminal = { backend = "local"; timeout = 180; };
};

# personality.nix
services.hermes-agent.settings = {
  display = { compact = false; personality = "kawaii"; };
  memory = { memory_enabled = true; user_profile_enabled = true; };
};
```

Nix-declared keys always win over keys in an existing `config.yaml` on disk, but user-added keys that Nix doesn't touch are preserved.

Run `nix build .#configKeys && cat result` to see every leaf config key.

### Escape Hatch: Bring Your Own Config

```nix
services.hermes-agent.configFile = /etc/hermes/config.yaml;
```

Bypasses `settings` entirely -- no merge, no generation.

## Secrets Management

Never put API keys in `settings` or `environment` -- values in Nix expressions end up in `/nix/store`, which is world-readable. Always use `environmentFiles` with a secrets manager.

### sops-nix

```nix
{
  sops = {
    defaultSopsFile = ./secrets/hermes.yaml;
    age.keyFile = "/home/user/.config/sops/age/keys.txt";
    secrets."hermes-env" = { format = "yaml"; };
  };

  services.hermes-agent.environmentFiles = [
    config.sops.secrets."hermes-env".path
  ];
}
```

### agenix

```nix
{
  age.secrets.hermes-env.file = ./secrets/hermes-env.age;
  services.hermes-agent.environmentFiles = [
    config.age.secrets.hermes-env.path
  ];
}
```

### OAuth / Auth Seeding

```nix
services.hermes-agent = {
  authFile = config.sops.secrets."hermes/auth.json".path;
  # authFileForceOverwrite = true;  # overwrite on every activation
};
```

## Documents

The `documents` option installs files into the agent's working directory:

```nix
services.hermes-agent.documents = {
  "SOUL.md" = ''
    You are a helpful research assistant specializing in NixOS packaging.
  '';
  "USER.md" = ./documents/USER.md;
};
```

`SOUL.md` is the agent's system prompt/personality. `USER.md` provides context about the user.

## MCP Servers

### Stdio Transport (Local Servers)

```nix
services.hermes-agent.mcpServers = {
  filesystem = {
    command = "npx";
    args = [ "-y" "@modelcontextprotocol/server-filesystem" "/data/workspace" ];
  };
  github = {
    command = "npx";
    args = [ "-y" "@modelcontextprotocol/server-github" ];
    env.GITHUB_PERSONAL_ACCESS_TOKEN = "\${GITHUB_TOKEN}";
  };
};
```

### HTTP Transport

```nix
services.hermes-agent.mcpServers.remote-api = {
  url = "https://mcp.example.com/v1/mcp";
  headers.Authorization = "Bearer \${MCP_REMOTE_API_KEY}";
  timeout = 180;
};
```

### HTTP with OAuth

```nix
services.hermes-agent.mcpServers.my-oauth-server = {
  url = "https://mcp.example.com/mcp";
  auth = "oauth";
};
```

Tokens are stored in `$HERMES_HOME/mcp-tokens/<server-name>.json`.

## Managed Mode

When Hermes runs via the NixOS module, these CLI commands are blocked:

| Blocked command | Why |
|---|---|
| `hermes setup` | Config is declarative |
| `hermes config edit` | Config is generated from `settings` |
| `hermes config set` | Config is generated from `settings` |
| `hermes gateway install` | systemd service is managed by NixOS |
| `hermes gateway uninstall` | systemd service is managed by NixOS |

Detection uses `HERMES_MANAGED=true` env var (set by systemd) and `.managed` marker file in `HERMES_HOME`.

## Container Architecture

When `container.enable = true`, Hermes runs inside a persistent Ubuntu container with the Nix-built binary bind-mounted read-only from the host.

Key mount points:
- `/nix/store` -- Hermes binary + all Nix deps (read-only)
- `/data` -- all state, config, workspace (read-write, maps to `/var/lib/hermes`)
- `/home/hermes` -- persistent agent home (read-write)
- `/usr`, `/usr/local`, `/tmp` -- writable layer for `apt`/`pip`/`npm`

The writable layer persists across restarts and code updates but is lost when the container identity hash changes (image upgrade, volume changes, option changes).

## Development

### Dev Shell

```bash
cd hermes-agent
nix develop
hermes setup
hermes chat
```

The dev shell provides Python 3.11, uv, Node.js 20, ripgrep, git, openssh, ffmpeg.

### direnv

```bash
cd hermes-agent
direnv allow    # one-time
```

### Flake Checks

```bash
nix flake check
```

Checks: `package-contents`, `entry-points-sync`, `cli-commands`, `managed-guard`, `bundled-skills`, `config-roundtrip`.

## Options Reference

### Core

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enable` | bool | false | Enable the hermes-agent service |
| `package` | package | hermes-agent | The hermes-agent package to use |
| `user` | str | "hermes" | System user |
| `group` | str | "hermes" | System group |
| `createUser` | bool | true | Auto-create user/group |
| `stateDir` | str | "/var/lib/hermes" | State directory |
| `workingDirectory` | str | "${stateDir}/workspace" | Agent working directory |
| `addToSystemPackages` | bool | false | Add CLI to system PATH and set HERMES_HOME |

### Configuration

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `settings` | attrs | {} | Declarative config rendered as config.yaml |
| `configFile` | null or path | null | Path to existing config.yaml (overrides settings) |

### Secrets & Environment

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `environmentFiles` | listOf str | [] | Paths to env files with secrets |
| `environment` | attrsOf str | {} | Non-secret env vars (visible in Nix store) |
| `authFile` | null or path | null | OAuth credentials seed |
| `authFileForceOverwrite` | bool | false | Always overwrite auth.json |

### Documents

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `documents` | attrsOf (either str path) | {} | Workspace files installed on activation |

### MCP Servers

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `mcpServers` | attrsOf submodule | {} | MCP server definitions |
| `mcpServers.<name>.command` | null or str | null | Server command (stdio) |
| `mcpServers.<name>.args` | listOf str | [] | Command arguments |
| `mcpServers.<name>.env` | attrsOf str | {} | Environment variables |
| `mcpServers.<name>.url` | null or str | null | Server URL (HTTP) |
| `mcpServers.<name>.headers` | attrsOf str | {} | HTTP headers |
| `mcpServers.<name>.auth` | null or "oauth" | null | Authentication method |
| `mcpServers.<name>.enabled` | bool | true | Enable/disable server |
| `mcpServers.<name>.timeout` | null or int | null | Tool call timeout (default: 120) |
| `mcpServers.<name>.sampling` | null or submodule | null | Sampling config |

### Service Behavior

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `extraArgs` | listOf str | [] | Extra args for hermes gateway |
| `extraPackages` | listOf package | [] | Extra packages on service PATH (native only) |
| `restart` | str | "always" | systemd Restart= policy |
| `restartSec` | int | 5 | systemd RestartSec= value |

### Container

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `container.enable` | bool | false | Enable OCI container mode |
| `container.backend` | enum ["docker" "podman"] | "docker" | Container runtime |
| `container.image` | str | "ubuntu:24.04" | Base image |
| `container.extraVolumes` | listOf str | [] | Extra volume mounts |
| `container.extraOptions` | listOf str | [] | Extra args for docker create |

## Updating

```bash
nix flake update hermes-agent --flake /etc/nixos
sudo nixos-rebuild switch
```

In container mode, the `current-package` symlink is updated and the agent picks up the new binary on restart. No container recreation needed.

## Troubleshooting

### Service Logs

```bash
journalctl -u hermes-agent -f
docker logs -f hermes-agent    # container mode
```

### Container Inspection

```bash
systemctl status hermes-agent
docker ps -a --filter name=hermes-agent
docker exec -it hermes-agent bash
```

### Force Container Recreation

```bash
sudo systemctl stop hermes-agent
docker rm -f hermes-agent
sudo rm /var/lib/hermes/.container-identity
sudo systemctl start hermes-agent
```

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Cannot save configuration: managed by NixOS | CLI guards active | Edit configuration.nix and nixos-rebuild switch |
| Container recreated unexpectedly | extraVolumes/extraOptions/image changed | Expected -- writable layer resets |
| hermes version shows old version | Container not restarted | systemctl restart hermes-agent |
| Permission denied on /var/lib/hermes | State dir is 0750 hermes:hermes | Use docker exec or sudo -u hermes |
| nix-collect-garbage removed hermes | GC root missing | Restart service (preStart recreates GC root) |
