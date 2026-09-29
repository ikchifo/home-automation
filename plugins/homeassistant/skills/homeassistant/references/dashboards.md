# Dashboard patterns

Patterns for YAML-mode Lovelace dashboards using Mushroom Cards
and Bubble Card.

## YAML-mode dashboards

YAML dashboards are stored at `/config/ui-dashboard-<name>.yaml`
on the HA instance. Edit locally and refresh the browser
(Cmd+Shift+R) — no HA restart needed.

## Storage-mode dashboards

UI-managed dashboards are not stored in the YAML dashboard files.
Connect to `${HASS_SERVER}/api/websocket` and wait for `auth_required`.
Send `{"type":"auth","access_token":"<TOKEN>"}` and wait for
`auth_ok`. Send messages with unique integer `id` values:

```json
{"id":1,"type":"lovelace/dashboards/list"}
{"id":2,"type":"lovelace/config","url_path":"<dashboard-path>"}
{"id":3,"type":"lovelace/config/save","url_path":"<dashboard-path>","config":{}}
```

For `config/save`, replace `{}` with the full `result` object from
`lovelace/config` after changing only the intended cards. Confirm
`success: true`, then read the config again to verify the change.
Do not overwrite other views or cards. Refresh the display if it
does not show the new layout.

To control multiple lights with one light card, create a light
group helper first (Settings > Devices & services > Helpers >
Create helper > Group > Light group). The config-flow API can
also create it: start `/api/config/config_entries/flow` with
`{"handler":"group"}`, submit `{"next_step_id":"light"}` to
the returned flow URL, then submit the form's `name`, `entities`,
`hide_members`, and `all` fields. Check the new `light.*` entity
before using it in a dashboard card.

## Mushroom Cards

Preferred for entity controls. Clean, minimal, touch-friendly.
Install via HACS > Frontend.

### Light card

```yaml
type: custom:mushroom-light-card
entity: light.living_room_a
use_light_color: true
show_brightness_control: true
show_color_temp_control: true
```

### Scene button

```yaml
type: custom:mushroom-template-card
entity: scene.living_room_relax
primary: Relax
icon: mdi:sofa
icon_color: orange
layout: vertical
tap_action:
  action: call-service
  service: scene.turn_on
  target:
    entity_id: scene.living_room_relax
```

### Fan card

```yaml
type: custom:mushroom-fan-card
entity: fan.air_circulator
show_percentage_control: true
show_oscillate_control: true
```

### Entity card

```yaml
type: custom:mushroom-entity-card
entity: sensor.air_circulator_temperature
name: Room Temperature
icon: mdi:thermometer
layout: horizontal
```

### Template card (dynamic content)

```yaml
type: custom:mushroom-template-card
primary: All Lights
secondary: >-
  {{ states.light
     | selectattr('state', 'eq', 'on')
     | list | length }} on
icon: mdi:lightbulb-group
icon_color: >-
  {% if states.light
     | selectattr('state', 'eq', 'on')
     | list | length > 0 %}amber
  {% else %}grey{% endif %}
```

### TV remote button

Uses Bravia `remote.send_command`:

```yaml
type: custom:mushroom-template-card
primary: VOL +
icon: mdi:volume-plus
layout: vertical
tap_action:
  action: call-service
  service: remote.send_command
  target:
    entity_id: remote.living_room_tv
  data:
    command: VolumeUp
```

## Bubble Card

Used for navigation bars and pop-up overlays. Install via HACS >
Frontend.

### Bottom nav bar

```yaml
type: custom:bubble-card
card_type: horizontal-buttons-stack
styles: |
  .horizontal-buttons-stack-container {
    z-index: 1;
    position: fixed;
    bottom: 0;
    left: 0;
    width: 100%;
  }
1_link: /home-dashboard/home
1_name: Home
1_icon: mdi:home
2_link: /home-dashboard/living-room
2_name: Living Room
2_icon: mdi:sofa
3_link: /home-dashboard/kitchen
3_name: Kitchen
3_icon: mdi:countertop
4_link: /home-dashboard/bedroom
4_name: Bedroom
4_icon: mdi:bed
```

## View structure

```yaml
views:
  - title: Home
    path: home
    cards:
      - type: horizontal-stack
        cards:
          - type: custom:mushroom-light-card
            entity: light.living_room_a
      # Bottom nav bar (last card in every view)
      - type: custom:bubble-card
        card_type: horizontal-buttons-stack
```

## Template syntax tips

Always use type filters when comparing values:

```yaml
# Correct
{{ states('sensor.temp') | float > 25 }}

# Wrong (string comparison)
{{ states('sensor.temp') > 25 }}
```

Test templates in Developer Tools > Template before adding to
dashboards.

## Design guidelines

- Mushroom Cards for entity controls (lights, fans, sensors)
- Bubble Card for navigation and overlays only
- Use `layout: vertical` for compact grid layouts
- Use `layout: horizontal` for list-style layouts
- Use `use_light_color: true` to show actual bulb color
- Group related cards in `horizontal-stack` or `vertical-stack`
- Keep each view focused on one room or function
