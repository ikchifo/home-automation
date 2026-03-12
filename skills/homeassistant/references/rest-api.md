# REST API reference

Complete endpoint reference for the Home Assistant REST API.
The main SKILL.md covers the most-used endpoints; this file
documents the rest.

Base URL: `${HASS_SERVER}/api/`
Auth: `Authorization: Bearer ${HASS_TOKEN}`

## API check

```bash
# Verify the API is reachable
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/" | jq .
# Returns: {"message": "API running."}
```

## States

```bash
# All states
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/states"

# Single entity
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/states/<entity_id>"

# Set/update state (creates entity if it doesn't exist)
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/states/<entity_id>" \
  -d '{"state": "25", "attributes": {"unit_of_measurement": "°C"}}'
```

## Services

```bash
# List all available services
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/services"

# Call a service
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/services/<domain>/<service>" \
  -d '{"entity_id": "<entity_id>"}'
```

The response is an array of states that changed as a result of
the service call.

## History

```bash
# History for specific entities over a time period
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/history/period/2026-03-12T00:00:00?filter_entity_id=sensor.temperature&end_time=2026-03-12T23:59:59"

# Minimal response (just state changes, no attributes)
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/history/period/2026-03-12T00:00:00?filter_entity_id=sensor.temperature&minimal_response&no_attributes"
```

Parameters:
- `filter_entity_id`: comma-separated entity IDs
- `end_time`: ISO 8601 timestamp
- `minimal_response`: omit attributes from response
- `no_attributes`: exclude attribute changes
- `significant_changes_only`: skip minor changes

## Logbook

```bash
# Recent logbook entries
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/logbook/2026-03-12T00:00:00"

# Filtered by entity
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/logbook/2026-03-12T00:00:00?entity=light.living_room_a"
```

## Events

```bash
# List event types
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/events"

# Fire an event
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/events/<event_type>" \
  -d '{"key": "value"}'
```

## Config

```bash
# Get HA configuration
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config"

# Check configuration validity
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/config/core/check_config"
```

## Calendars

```bash
# List calendars
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/calendars"

# Get events from a calendar
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/calendars/<calendar_entity_id>?start=2026-03-12T00:00:00Z&end=2026-03-13T00:00:00Z"
```

## Camera proxy

```bash
# Get camera snapshot
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/camera_proxy/<camera_entity_id>" \
  -o snapshot.jpg
```

## Intent handling

```bash
# Handle an intent directly
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/intent/handle" \
  -d '{"intent": "HassTurnOn", "data": {"name": "living room lights"}}'
```

## Conversation

```bash
# Process natural language
curl -s -X POST -H "Authorization: Bearer $HASS_TOKEN" \
  -H "Content-Type: application/json" \
  "$HASS_SERVER/api/conversation/process" \
  -d '{"text": "Turn on the living room lights", "language": "en"}'
```

## Error log

```bash
curl -s -H "Authorization: Bearer $HASS_TOKEN" \
  "$HASS_SERVER/api/error_log"
```

## Response patterns

Most endpoints return JSON. Common patterns:

- **GET /states**: array of state objects
  `{entity_id, state, attributes, last_changed, last_updated}`
- **POST /services**: array of changed state objects
- **GET /config/config_entries/entry**: array of config entry
  objects `{entry_id, domain, title, state, ...}`
- **POST /template**: rendered string (plain text, not JSON)
- **GET /error_log**: plain text log output

## Error codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created (state set) |
| 400 | Bad request (invalid JSON, missing fields) |
| 401 | Unauthorized (bad/missing token) |
| 404 | Entity or endpoint not found |
| 405 | Method not allowed |
