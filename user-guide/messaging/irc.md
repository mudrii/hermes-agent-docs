---
sidebar_position: 17
title: "IRC"
description: "Set up Hermes Agent as an IRC bot"
---

# IRC Setup

Run Hermes Agent as an IRC bot through the bundled `plugins/platforms/irc/` platform plugin. The adapter uses Python's standard `asyncio` protocol support and connects to one or more IRC channels with a configured bot nickname.

## Configuration

Add the required IRC settings to `~/.hermes/.env`:

```env
IRC_SERVER=irc.libera.chat
IRC_CHANNEL=#hermes
IRC_NICKNAME=hermes-bot
```

Optional connection settings:

```env
IRC_PORT=6697
IRC_USE_TLS=true
IRC_SERVER_PASSWORD=
IRC_NICKSERV_PASSWORD=
```

Control who can talk to the bot:

```env
# Production: allow only known nicks.
IRC_ALLOWED_USERS=alice,bob

# Development only: allow anyone in the joined channel.
IRC_ALLOW_ALL_USERS=true
```

For cron or notification delivery, set a home channel. If omitted, Hermes uses `IRC_CHANNEL`.

```env
IRC_HOME_CHANNEL=#hermes
```

## Run

Start the gateway after saving the environment values:

```bash
hermes gateway
```

You can also use `hermes gateway setup`; the IRC plugin contributes its required and optional environment fields to the setup flow.

## Config File Form

Environment variables override `config.yaml`, but config-only setups are also supported:

```yaml
gateway:
  platforms:
    irc:
      enabled: true
      extra:
        server: irc.libera.chat
        port: 6697
        nickname: hermes-bot
        channel: "#hermes"
        use_tls: true
        allowed_users:
          - alice
          - bob
```

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `IRC_SERVER` | yes | - | IRC server hostname, for example `irc.libera.chat` |
| `IRC_CHANNEL` | yes | - | Channel to join, for example `#hermes`; comma-separate multiple channels |
| `IRC_NICKNAME` | yes | `hermes-bot` | Bot nickname |
| `IRC_PORT` | no | `6697` with TLS, `6667` without TLS | IRC server port |
| `IRC_USE_TLS` | no | true on port `6697` | Use TLS when connecting |
| `IRC_SERVER_PASSWORD` | no | - | Server password for the IRC `PASS` command |
| `IRC_NICKSERV_PASSWORD` | no | - | NickServ password for automatic `IDENTIFY` on connect |
| `IRC_ALLOWED_USERS` | no | - | Comma-separated IRC nicks allowed to talk to the bot |
| `IRC_ALLOW_ALL_USERS` | no | `false` | Dev-only escape hatch; allow anyone in the channel |
| `IRC_HOME_CHANNEL` | no | `IRC_CHANNEL` | Channel for cron / notification delivery |
