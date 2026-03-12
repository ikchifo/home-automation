# hass-cli reference

Command-line interface for Home Assistant. Wraps the REST API
with a convenient CLI.

## Installation

```bash
uv tool install homeassistant-cli
```

Or: `pip install homeassistant-cli`

## Configuration

hass-cli reads these environment variables:

```bash
export HASS_SERVER=http://<HA_IP>:8123
export HASS_TOKEN=your_long_lived_token
```

Or pass them as flags: `--server` and `--token`.

## State management

```bash
# List all entity states
hass-cli state list

# Get a specific entity
hass-cli state get light.living_room_a

# Get state with specific output format
hass-cli --output json state get light.living_room_a

# Set entity state
hass-cli state edit sensor.example --json '{"state": "25"}'

# Delete an entity
hass-cli state delete sensor.old_entity

# Watch for state changes
hass-cli state watch light.living_room_a
```

## Service calls

```bash
# Call a service
hass-cli service call light.turn_on \
  --arguments entity_id=light.living_room_a

# With multiple arguments
hass-cli service call light.turn_on \
  --arguments entity_id=light.living_room_a,brightness=200

# Reload automations
hass-cli service call automation.reload

# Trigger an automation
hass-cli service call automation.trigger \
  --arguments entity_id=automation.reload_dreo_on_startup
```

## Events

```bash
# Fire an event
hass-cli event fire my_custom_event --json '{"key": "value"}'

# Watch events in real-time
hass-cli event watch state_changed

# Watch specific entity changes
hass-cli event watch state_changed \
  --json '{"entity_id": "light.living_room_a"}'
```

## Device and area management

```bash
# List devices
hass-cli device list

# List areas
hass-cli area list

# Create an area
hass-cli area create "Living Room"

# Delete an area
hass-cli area delete "Old Room"
```

## Templates

```bash
# Render a Jinja2 template
hass-cli template "{{ states('sensor.temperature') }}"

# Multiline template
hass-cli template "
{% for light in states.light %}
  {{ light.entity_id }}: {{ light.state }}
{% endfor %}
"
```

## Raw API access

```bash
# GET request
hass-cli raw get /api/config

# POST request
hass-cli raw post /api/services/light/turn_on \
  --json '{"entity_id": "light.living_room_a"}'
```

## Discovery

```bash
# Discover HA instances on the network
hass-cli discover
```

## Output formatting

```bash
# JSON output
hass-cli --output json state list

# Table output (default)
hass-cli --output table state list

# Specific columns
hass-cli --columns entity_id,state state list
```

## Useful patterns

```bash
# Find all unavailable entities
hass-cli state list | rg unavailable

# List all lights and their states
hass-cli --output json state list \
  | jq '.[] | select(.entity_id | startswith("light."))'

# Check if HA is reachable
hass-cli raw get /api/ && echo "OK" || echo "UNREACHABLE"
```
