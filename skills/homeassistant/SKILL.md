---
name: homeassistant
description: >-
  Manage Home Assistant via REST API, hass-cli, and SSH. Use this
  skill whenever the user mentions Home Assistant, HA, smart home,
  lights, switches, fans, automations, dashboards, Lovelace,
  entities, services, integrations, HACS, HomeKit Bridge, Siri,
  Apple Home, or any IoT/home automation task. Also triggers for
  AdGuard Home DNS, Tailscale VPN, or connected devices like Hue,
  WiZ, Bravia TV, Dreo — even if the user doesn't explicitly say
  "Home Assistant".
---

# Home Assistant manager

Manage a Home Assistant instance through its REST API, the
`hass-cli` CLI, and SSH. Prefer the REST API for direct
operations, `hass-cli` for quick interactive queries, and SSH
only for system-level tasks (restart, logs, config validation).

## Setup

### Environment variables

Check for these before any HA operation:

```bash
HASS_SERVER   # e.g. http://<HA_IP>:8123
HASS_TOKEN    # Long-lived access token
```

If unset, ask the user. Tokens are created at:
Profile (bottom-left) > Long-Lived Access Tokens > Create Token.

### hass-cli

Install if not already present:

```bash
uv tool install homeassistant-cli
```

`hass-cli` reads `HASS_SERVER` and `HASS_TOKEN` automatically.
See `references/hass-cli.md` for the full command reference.

## Tool selection

| Task | Tool | Why |
|------|------|-----|
| Read/write entity states | REST API | Direct, no deps |
| Call services | REST API or hass-cli | Both work well |
| List/filter entities | REST API template | Jinja2 filtering |
| Manage config entries | REST API | Only programmatic way |
| Integration setup (flows) | REST API | Step-by-step flow |
| Render templates | REST API | POST `/api/template` |
| Check error log | REST API | GET `/api/error_log` |
| Validate configuration | SSH `ha core check` | No REST equiv |
| Restart HA core | SSH `ha core restart` | No REST equiv |
| Reload services | REST API | Service call |
| View live logs | SSH | Streaming needed |
| Edit dashboards | Local file edit | YAML files |

## REST API

Base: `${HASS_SERVER}/api/`
Auth header: `Authorization: Bearer ${HASS_TOKEN}`

### Entity states

```bash
# List all entity IDs
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/states" | jq '.[].entity_id'

# Get one entity (state + attributes)
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/states/light.living_room_a" \
  | jq '{state, attributes}'
```

### Service calls

```bash
# Turn on a light with brightness
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/services/light/turn_on" \
  -d '{"entity_id": "light.living_room_a", "brightness": 255}'

# Activate a scene
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/services/scene/turn_on" \
  -d '{"entity_id": "scene.living_room_relax"}'

# Set fan speed
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/services/fan/set_percentage" \
  -d '{"entity_id": "fan.air_circulator", "percentage": 50}'

# Reload automations (no restart needed)
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/services/automation/reload"
```

### Templates

Jinja2 templates are the most flexible query mechanism:

```bash
# List entities for an integration
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/template" \
  -d '{"template": "{% for e in integration_entities(\"hue\") %}{{ e }}\n{% endfor %}"}'

# Count entities by domain
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/template" \
  -d '{"template": "{{ states.light | list | length }} lights"}'

# Find unavailable entities
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/template" \
  -d '{"template": "{% for s in states if s.state == \"unavailable\" %}{{ s.entity_id }}\n{% endfor %}"}'
```

### Config entries

```bash
# List all integrations
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/config_entries/entry" \
  | jq '[.[] | {domain, entry_id, title, state}]'

# Filter by domain
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/config_entries/entry" \
  | jq '[.[] | select(.domain == "hue")]'

# Reload a config entry
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/config_entries/entry/<entry_id>/reload"

# Delete a config entry
curl -s -X DELETE -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/config_entries/entry/<entry_id>"
```

### Config flows (set up new integrations)

Config flows are multi-step. Each response returns the next
step's `type`, `step_id`, and `data_schema` (expected fields).
Keep submitting until `type` is `create_entry` or `abort`.

```bash
# Start a new integration flow
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/config/config_entries/flow" \
  -d '{"handler": "wiz"}' | jq .

# Submit the next step (fields from data_schema)
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/config/config_entries/flow/<flow_id>" \
  -d '{"host": "<DEVICE_IP>"}'
```

### Options flows (reconfigure existing integration)

Same multi-step pattern as config flows, but for changing
settings on an already-installed integration.

```bash
# Start options flow
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/config/config_entries/options/flow" \
  -d '{"handler": "<entry_id>"}'

# Submit a step
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/config/config_entries/options/flow/<flow_id>" \
  -d '{"key": "value"}'
```

### Conversation API

Send natural language commands through HA's intent system:

```bash
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/conversation/process" \
  -d '{"text": "Turn on the living room lights", "language": "en"}'
```

### Error log

```bash
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/error_log"
```

See `references/rest-api.md` for the complete endpoint reference
(history, logbook, events, calendars, camera proxy).

## Reload vs restart

Always prefer reload — it's faster and doesn't interrupt running
automations or active connections.

### Reloadable via service call

| Domain | Service |
|--------|---------|
| Automations | `automation/reload` |
| Scripts | `script/reload` |
| Scenes | `scene/reload` |
| Template entities | `template/reload` |
| Groups | `group/reload` |
| Input helpers | `input_boolean/reload`, etc. |
| HomeKit Bridge | `homekit/reload` |
| Themes | `frontend/reload_themes` |
| Core config | `homeassistant/reload_core_config` |
| Single config entry | `homeassistant/reload_config_entry` |

### Requires full restart via SSH

- New integrations added to `configuration.yaml`
- Changes to the `homeassistant:` block
- Min/max/statistics sensor platforms
- MQTT platform-based sensors

```bash
ssh <user>@<ha_ip> "ha core check && ha core restart"
```

## Automation workflow

### Deploy and verify

1. **Edit** the automation YAML locally.
2. **Deploy** to the HA instance:
   ```bash
   scp automations.yaml <user>@<ha_ip>:/config/
   ```
3. **Reload**:
   ```bash
   curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
     "$HASS_SERVER/api/services/automation/reload"
   ```
4. **Trigger** manually:
   ```bash
   curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
     -H "Content-Type: application/json" \
     "$HASS_SERVER/api/services/automation/trigger" \
     -d '{"entity_id": "automation.reload_dreo_on_startup"}'
   ```
5. **Check logs**:
   ```bash
   curl -s -H "Authorization: Bearer $HASS_TOKEN" \
     "$HASS_SERVER/api/error_log"
   ```

### Log indicators

**Success**: `Initialized trigger`, `Running automation actions`,
`Executing step`

**Error**: `Error executing script`,
`Invalid data for call_service`, `TypeError`

## Troubleshooting

### Entity unavailable after restart

Cloud-dependent integrations (Dreo, Tuya) can fail if DNS isn't
ready at boot. Reload the config entry:

```bash
# Find the entry ID
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/config_entries/entry" \
  | jq '.[] | select(.domain == "dreo") | .entry_id'

# Reload it
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/config_entries/entry/<entry_id>/reload"
```

Consider a startup automation to handle this automatically.

### HomeKit Bridge issues

- TVs must use **accessory mode** (HAP protocol constraint).
- After config changes, re-pair: remove bridge in Apple Home,
  reload, scan new QR code.
- Duplicate devices: check for multiple integrations exposing
  the same device (Cast + Bravia, native AirPlay + HA bridge).

```bash
# Reload HomeKit
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/services/homekit/reload"

# Reset a specific accessory
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/services/homekit/reset_accessory" \
  -d '{"entity_id": "media_player.living_room_tv_2"}'
```

### Integration won't set up

Use config flows to debug step by step — the response includes
`data_schema` (required fields) and `errors` (what failed):

```bash
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/config/config_entries/flow" \
  -d '{"handler": "braviatv"}' | jq .
```

### DNS-related slowness

If first page loads hang then resolve on retry, IPv6 is likely
the culprit. Disable it in AdGuard:

```bash
curl -s -X POST http://127.0.0.1:45158/control/dns_config \
  -H "Content-Type: application/json" \
  -d '{"disable_ipv6": true}'
```

## LLM integration (Assist API)

Home Assistant exposes entities to LLMs through the Assist API,
enabling voice control and conversational agents.

### Architecture

1. **Intents**: HA defines intents (`HassTurnOn`, `HassTurnOff`,
   `HassGetState`, etc.) that map to service calls.
2. **Entity exposure**: Entities must be explicitly exposed via
   Settings > Voice assistants > Expose tab.
3. **Tool-based API**: LLM agents receive available tools
   (intents) with typed parameters and call them.

### Custom LLM API

For developers building custom conversation agents, HA supports
registering additional LLM APIs via `llm.async_register_api()`
in a custom component. Each API defines tools with parameters
and handlers, extending what voice assistants can do beyond
built-in intents. See the HA developer docs for the full
registration pattern.

## Dashboards

This setup uses **YAML-mode dashboards** with Mushroom Cards
for entity controls and Bubble Card for the bottom navigation
bar. Edit YAML locally; changes appear on browser refresh
(Cmd+Shift+R) — no HA restart needed.

See `references/dashboards.md` for card patterns, view types,
template syntax, and the bottom nav bar setup.

## SSH operations

For system-level tasks without a REST equivalent:

```bash
# Validate configuration
ssh <user>@<ha_ip> "ha core check"

# Restart
ssh <user>@<ha_ip> "ha core restart"

# View logs
ssh <user>@<ha_ip> "ha core logs"

# System info
ssh <user>@<ha_ip> "ha host info"
ssh <user>@<ha_ip> "ha os info"

# Create backup
ssh <user>@<ha_ip> "ha backups new --name 'pre-change-backup'"
```

## Quick reference

### Common services

| Service | Key parameters |
|---------|---------------|
| `light/turn_on` | `entity_id`, `brightness`, `color_temp` |
| `light/turn_off` | `entity_id` |
| `scene/turn_on` | `entity_id` |
| `fan/turn_on` | `entity_id` |
| `fan/set_percentage` | `entity_id`, `percentage` |
| `media_player/turn_on` | `entity_id` |
| `media_player/volume_set` | `entity_id`, `volume_level` (0-1) |
| `remote/send_command` | `entity_id`, `command` |
| `automation/reload` | (none) |
| `automation/trigger` | `entity_id` |
| `homekit/reload` | (none) |
| `homeassistant/reload_config_entry` | `entry_id` |

### Bravia remote commands

Via `remote/send_command` on `remote.living_room_tv`:

**Navigation**: `Up`, `Down`, `Left`, `Right`, `Confirm`,
`Return`, `Home`

**Volume**: `VolumeUp`, `VolumeDown`, `Mute`

**Playback**: `Play`, `Pause`, `Stop`, `Rewind`, `Forward`

**Apps**: `Netflix`, `YouTube`
