# Home Assistant

Hermes Agent integrates with [Home Assistant](https://www.home-assistant.io/) in two ways:

1. **Gateway platform** — subscribes to real-time state changes via WebSocket and responds to events
2. **Smart home tools** — four LLM-callable tools for querying and controlling devices via the REST API

Both are activated by setting `HASS_TOKEN`. The `homeassistant` toolset is automatically enabled when `HASS_TOKEN` is set.

This document covers v0.2.0 through v0.8.0 (v2026.4.8). No Home Assistant-specific changes were made after v0.5.0.

---

## Setup

### Step 1: Create a Long-Lived Access Token

1. Open your Home Assistant instance
2. Go to your **Profile** (click your name in the sidebar)
3. Scroll to **Long-Lived Access Tokens**
4. Click **Create Token**, give it a name like "Hermes Agent"
5. Copy the token — it is only shown once

### Step 2: Configure Environment Variables

Add to `~/.hermes/.env`:

```bash
# Required
HASS_TOKEN=your-long-lived-access-token

# Optional (default: http://homeassistant.local:8123)
HASS_URL=http://192.168.1.100:8123
```

### Step 3: Start the Gateway

```bash
hermes gateway              # Foreground
hermes gateway install      # Install as a user service
sudo hermes gateway install --system   # Linux only: system service
```

Home Assistant appears as a connected platform alongside any other messaging platforms (Telegram, Discord, etc.).

---

## Available Tools

Hermes Agent registers four tools for smart home control. These are available in all sessions when `HASS_TOKEN` is set — not just when using the HA gateway platform.

### `ha_list_entities`

List Home Assistant entities, optionally filtered by domain or area.

**Parameters:**
- `domain` (optional) — Filter by entity domain: `light`, `switch`, `climate`, `sensor`, `binary_sensor`, `cover`, `fan`, `media_player`, etc.
- `area` (optional) — Filter by area/room name (matches against friendly names): `living room`, `kitchen`, `bedroom`, etc.

Returns entity IDs, states, and friendly names.

### `ha_get_state`

Get detailed state of a single entity, including all attributes (brightness, color, temperature setpoint, sensor readings, etc.).

**Parameters:**
- `entity_id` (required) — The entity to query, e.g., `light.living_room`, `climate.thermostat`, `sensor.temperature`

Returns: state, all attributes, last changed/updated timestamps.

### `ha_list_services`

List available services (actions) for device control. Shows what actions can be performed on each device type and what parameters they accept.

**Parameters:**
- `domain` (optional) — Filter by domain, e.g., `light`, `climate`, `switch`

### `ha_call_service`

Call a Home Assistant service to control a device.

**Parameters:**
- `domain` (required) — Service domain: `light`, `switch`, `climate`, `cover`, `media_player`, `fan`, `scene`, `script`
- `service` (required) — Service name: `turn_on`, `turn_off`, `toggle`, `set_temperature`, `set_hvac_mode`, `open_cover`, `close_cover`, `set_volume_level`
- `entity_id` (optional) — Target entity, e.g., `light.living_room`
- `data` (optional) — Additional parameters as a JSON object

---

## Gateway Platform: Real-Time Events

The Home Assistant gateway adapter connects via WebSocket and subscribes to `state_changed` events. When a device state changes and matches your filters, it is forwarded to the agent as a message.

### Event Filtering

By default, **no events are forwarded**. You must configure at least one of `watch_domains`, `watch_entities`, or `watch_all` to receive events. Without filters, a warning is logged at startup and all state changes are silently dropped.

This default-closed behavior was added in v0.3.0. Previous versions may have forwarded all events.

Configure which events the agent sees in `~/.hermes/gateway.json` (or under the `platforms` key in `~/.hermes/config.yaml`) under the Home Assistant platform's `extra` section:

```json
{
  "platforms": {
    "homeassistant": {
      "enabled": true,
      "extra": {
        "watch_domains": ["climate", "binary_sensor", "alarm_control_panel", "light"],
        "watch_entities": ["sensor.front_door_battery"],
        "ignore_entities": ["sensor.uptime", "sensor.cpu_usage", "sensor.memory_usage"],
        "cooldown_seconds": 30
      }
    }
  }
}
```

| Setting | Default | Description |
|---------|---------|-------------|
| `watch_domains` | (none) | Only watch these entity domains (e.g., `climate`, `light`, `binary_sensor`) |
| `watch_entities` | (none) | Only watch these specific entity IDs |
| `watch_all` | `false` | Set to `true` to receive all state changes (not recommended for most setups) |
| `ignore_entities` | (none) | Always ignore these entities (applied before domain/entity filters) |
| `cooldown_seconds` | `30` | Minimum seconds between events for the same entity |

Start with a focused set of domains — `climate`, `binary_sensor`, and `alarm_control_panel` cover the most useful automations. Use `ignore_entities` to suppress noisy sensors like CPU temperature or uptime counters.

### Event Formatting

State changes are formatted as human-readable messages based on domain:

| Domain | Format |
|--------|--------|
| `climate` | "HVAC mode changed from 'off' to 'heat' (current: 21, target: 23)" |
| `sensor` | "changed from 21°C to 22°C" |
| `binary_sensor` | "triggered" / "cleared" |
| `light`, `switch`, `fan` | "turned on" / "turned off" |
| `alarm_control_panel` | "alarm state changed from 'armed_away' to 'triggered'" |
| (other) | "changed from 'old' to 'new'" |

### Agent Responses

Outbound messages from the agent are delivered as **Home Assistant persistent notifications** (via `persistent_notification.create`). These appear in the HA notification panel with the title "Hermes Agent". Messages are limited to 4,096 characters.

### Connection Management

- **WebSocket** with 30-second heartbeat for real-time events
- **Automatic reconnection** with backoff: 5s → 10s → 30s → 60s
- **REST API** for outbound notifications (separate session to avoid WebSocket conflicts)
- **Authorization** — HA events are always authorized (no user allowlist needed, since `HASS_TOKEN` authenticates the connection)

---

## Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `HASS_TOKEN` | Yes | — | Long-Lived Access Token from HA profile |
| `HASS_URL` | No | `http://homeassistant.local:8123` | Home Assistant URL |

---

## Security

The Home Assistant tools enforce security restrictions. The following service domains are **blocked** to prevent arbitrary code execution on the HA host:

- `shell_command` — arbitrary shell commands
- `command_line` — sensors/switches that execute commands
- `python_script` — scripted Python execution
- `pyscript` — broader scripting integration
- `hassio` — addon control, host shutdown/reboot
- `rest_command` — HTTP requests from HA server (SSRF vector)

Attempting to call services in these domains returns an error.

Entity IDs are validated against the pattern `^[a-z_][a-z0-9_]*\.[a-z0-9_]+$` to prevent injection attacks.

---

## Example Automations

**Morning Routine:**

```
User: Start my morning routine

Agent:
1. ha_call_service(domain="light", service="turn_on",
     entity_id="light.bedroom", data={"brightness": 128})
2. ha_call_service(domain="climate", service="set_temperature",
     entity_id="climate.thermostat", data={"temperature": 22})
3. ha_call_service(domain="media_player", service="turn_on",
     entity_id="media_player.kitchen_speaker")
```

**Security Check:**

```
User: Is the house secure?

Agent:
1. ha_list_entities(domain="binary_sensor")  → checks door/window sensors
2. ha_get_state(entity_id="alarm_control_panel.home")  → checks alarm status
3. ha_list_entities(domain="lock")  → checks lock states
4. Reports: "All doors closed, alarm is armed_away, all locks engaged."
```

**Reactive Automation (via Gateway Events):**

```
[Home Assistant] Front Door: triggered (was cleared)

Agent automatically:
1. ha_get_state(entity_id="binary_sensor.front_door")
2. ha_call_service(domain="light", service="turn_on",
     entity_id="light.hallway")
3. Sends notification: "Front door opened. Hallway lights turned on."
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| No events arriving despite configuration | Verify `watch_domains` or `watch_entities` are set in `gateway.json`. Check gateway logs for the "no event filters configured" warning. |
| Too many events / agent overwhelmed | Use `ignore_entities` to exclude noisy sensors. Increase `cooldown_seconds`. Narrow `watch_domains` to only needed domains. |
| "Blocked domain" error | The service domain you called is in the blocked list for security reasons. Use HA automations or scripts instead for those actions. |
| Connection drops frequently | Check HA logs for WebSocket errors. Ensure your HA instance is accessible from the machine running Hermes. |
| Tool calls fail with "entity not found" | Verify the entity ID is correct using `ha_list_entities`. Entity IDs must match the pattern `domain.name`. |

---

## Changelog

- **v0.5.0:** Request timeouts added to HA REST API calls ([PR #3258](https://github.com/NousResearch/hermes-agent/pull/3258)).
