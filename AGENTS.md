# Home Assistant setup guide

## Infrastructure

| Component | Details |
|-----------|---------|
| Hardware | Raspberry Pi 5 8GB, 64GB SD card (A2-rated) |
| OS | Home Assistant OS |
| IP | `<HA_IP>` |
| Access | `http://<HA_IP>:8123` |
| API token | Long-lived token via Profile > Long-Lived Access Tokens |

### Devices

| Device | Integration | Entity |
|--------|------------|--------|
| Sony BRAVIA XR-55X90J | `braviatv` (PSK auth, IP: <TV_IP>) | `media_player.living_room_tv_2` |
| Living Room TV (Cast) | `cast` | `media_player.living_room_tv` |
| WiZ Tunable White x2 | `wiz` | `light.bedroom_light_a`, `light.bedroom_light_b` |
| Philips Hue (10 lights) | `hue` (Bridge) | `light.living_room_*`, `light.kitchen_*`, `light.entryway_light` |
| Google Hub Max | `cast` (IP: <HUB_IP>) | `media_player.chifo_s_hub_max` |
| Dreo Air Circulator | `dreo` (HACS, cloud-dependent) | `fan.air_circulator`, `sensor.air_circulator_temperature` |

### Add-ons

| Add-on | Purpose |
|--------|---------|
| AdGuard Home | DNS-level ad blocking (replaces Pi-hole) |
| Tailscale | VPN, exit node, DNS |
| Advanced SSH & Web Terminal | SSH + web terminal (community, by Frenck) |
| System Monitor | CPU, memory, disk sensors |

### HACS integrations and frontend

| Component | Type | Purpose |
|-----------|------|---------|
| HACS | Integration | Community store for integrations, cards, themes |
| Mushroom Cards | Frontend | Clean, minimal dashboard cards (preferred) |
| Bubble Card | Frontend | Pop-ups, bottom nav bar, media player cards |
| Adaptive Lighting | Integration | Auto color temp by time of day |
| Dreo Smart Device | Integration | Dreo fan control |

---

## HA REST API

Base URL: `http://<HA_IP>:8123/api/`

All requests require: `Authorization: Bearer <TOKEN>`

### Common endpoints

```bash
# Verify API
curl -s -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/

# List all entity states
curl -s -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/states | jq '.[].entity_id'

# Get specific entity
curl -s -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/states/<entity_id>

# Call a service
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/services/<domain>/<service> \
  -d '{"entity_id": "<entity_id>"}'

# Render a Jinja2 template
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/template \
  -d '{"template": "<jinja2>"}'

# List entities for a specific integration
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/template \
  -d '{"template": "{% for e in integration_entities(\"wiz\") %}{{ e }}\n{% endfor %}"}'

# List all config entries (integrations)
curl -s -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/config/config_entries/entry

# Filter config entries by domain
curl -s -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/config/config_entries/entry \
  | jq '[.[] | select(.domain == "<domain>")]'

# Delete a config entry
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/config/config_entries/entry/<entry_id>

# Reload a config entry
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/config/config_entries/entry/<entry_id>/reload

# Check error log
curl -s -H "Authorization: Bearer $TOKEN" \
  http://<HA_IP>:8123/api/error_log
```

### Integration setup via config flow

```bash
# Start a new integration flow
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/config/config_entries/flow \
  -d '{"handler": "<integration_domain>"}'

# Submit a flow step
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/config/config_entries/flow/<flow_id> \
  -d '{"<field>": "<value>"}'
```

### Integration options flow

```bash
# Start options flow for an existing config entry
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/config/config_entries/options/flow \
  -d '{"handler": "<entry_id>"}'

# Submit an options flow step
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/config/config_entries/options/flow/<flow_id> \
  -d '{"<field>": "<value>"}'
```

---

## HomeKit Bridge

### Architecture

Two HomeKit entries are required due to Apple HAP protocol
constraints:

| Entry | Mode | Exposes |
|-------|------|---------|
| HASS Bridge:21064 | Bridge | All lights, switches, fans, temperature sensor |
| HASS Bridge ZX:21065 | Accessory | Sony Bravia TV only |

The bridge uses **exclude mode**: includes domains
`alarm_control_panel`, `climate`, `cover`, `fan`, `humidifier`,
`light`, `lock`, `media_player`, `sensor`, `switch`, `vacuum`,
`water_heater` and excludes specific entities:
`media_player.living_room_tv` (Cast duplicate), backup sensors,
and power sensors.

### Key constraint: TVs must use accessory mode

Apple's HomeKit Accessory Protocol (HAP) requires television
accessories to be paired as **standalone accessories**, not through
a bridge. Attempting to include a TV (`device_class: tv`) in a
bridge will silently fail -- the TV won't appear in Apple Home.

### Common causes of duplicate devices in Apple Home

1. **Multiple integrations for the same device**: Smart TVs
   advertise via Cast, DLNA, and AirPlay simultaneously. HA
   auto-discovers all of them. Fix: delete the redundant config
   entries (keep one).
2. **TV's native HomeKit broadcasting**: Many TVs (Samsung, LG,
   Sony) have built-in AirPlay/HomeKit that advertises directly to
   Apple Home, bypassing HA. Fix: disable HomeKit in the TV's own
   settings.
3. **HomeKit Controller + HomeKit Bridge loop**: If HA imports a
   device via HomeKit Controller and re-exports it via HomeKit
   Bridge, Apple Home sees it twice. Fix: delete the HomeKit
   Controller entry for that device.

### Useful HomeKit services

```bash
# Reload HomeKit Bridge
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/services/homekit/reload

# Reset a specific accessory (forces re-sync to Apple Home)
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://<HA_IP>:8123/api/services/homekit/reset_accessory \
  -d '{"entity_id": "media_player.living_room_tv_2"}'
```

### Re-pairing workflow

When HomeKit Bridge config changes (entity exclusions, mode
changes), Apple Home often caches stale accessories. Clean
re-pair:

1. Remove the bridge in Apple Home (Home Settings > Hubs &
   Bridges)
2. Reload the bridge via API or HA UI
3. Scan the new QR code in HA (Settings > Devices & Services >
   HomeKit Bridge)

---

## Sony Bravia (XR-55X90J)

### Setup

1. On the TV: Settings > Network > Home Network > IP Control
2. Set Authentication to "Normal and Pre-Shared Key"
3. Set a Pre-Shared Key
4. In HA: add `braviatv` integration with the TV's IP and PSK

### Entities created

| Entity | Purpose |
|--------|---------|
| `media_player.living_room_tv_2` | Power, volume, input, source |
| `remote.living_room_tv` | Individual remote button commands |
| `button.living_room_tv_restart` | Restart TV |
| `button.living_room_tv_terminate_apps` | Kill running apps |

### Google Cast vs Bravia

- **Cast** (`media_player.living_room_tv`): only starts a Cast
  session. "Turn on" shows HA wallpaper, not real power on.
- **Bravia** (`media_player.living_room_tv_2`): sends real
  power/volume/input commands via Sony's IRCC-IP and REST API.
  Use this one for HomeKit/Siri control.

---

## AdGuard Home + Tailscale DNS

To use AdGuard as the DNS for your entire Tailnet (replaces
Pi-hole):

1. In the Tailscale admin console
   (https://login.tailscale.com/admin/dns), add a custom
   nameserver pointing to the Pi 5's Tailscale IP
2. Enable "Override local DNS" to force all Tailnet devices
   through AdGuard

### AdGuard API (local)

The web UI runs on the ingress port only. The API is accessible
locally within the Pi:

```bash
# Base URL (from SSH on the Pi)
http://127.0.0.1:45158/control/

# Get DNS config
curl -s http://127.0.0.1:45158/control/dns_info

# Update DNS config
curl -s -X POST http://127.0.0.1:45158/control/dns_config \
  -H "Content-Type: application/json" \
  -d '{"disable_ipv6": true, "cache_optimistic": true}'

# Get filter lists
curl -s http://127.0.0.1:45158/control/filtering/status

# Get query log
curl -s "http://127.0.0.1:45158/control/querylog?limit=50"

# Get stats
curl -s http://127.0.0.1:45158/control/stats
```

### DNS performance tuning

Optimized settings (applied 2026-03-12):

| Setting | Value | Why |
|---------|-------|-----|
| `disable_ipv6` | `true` | Network has local IPv6 addresses (`fd02:`) but no IPv6 internet. Without this, apps try AAAA records first, hang for seconds, then fall back to IPv4. |
| `cache_optimistic` | `true` | Serves stale cache immediately while refreshing upstream in background. Eliminates pause when cached entries expire. |
| `cache_size` | 16 MB | 4x default. Pi has 8GB RAM, plenty to spare. |
| `cache_ttl_min` | 300s (5 min) | Entries stay cached at least 5 min even if upstream says shorter. |
| `cache_ttl_max` | 86400s (24h) | Caps max cache to prevent stale data. |
| `upstream_mode` | `parallel` | Queries Cloudflare and Google DNS-over-HTTPS simultaneously, uses fastest response. |
| `upstream_timeout` | 5s | Fails faster on unresponsive upstream (was 10s). |
| `ratelimit` | 0 (unlimited) | 20/s default was too low for a household with many devices. |
| `bootstrap_dns` | `1.1.1.1`, `8.8.8.8` | Redundancy for resolving DoH upstream hostnames. |

**IPv6 diagnosis**: The Pi has `fd02:` addresses assigned by the
router but `ping6 google.com` returns "Network unreachable". This
caused the classic "first load hangs, second load fast" symptom
where apps prefer IPv6, timeout, then fall back to IPv4.

### Upstream DNS

Uses DNS-over-HTTPS (DoH) for encrypted queries:

- `https://dns.cloudflare.com/dns-query` (avg ~12ms)
- `https://dns.google/dns-query` (avg ~16ms)

### Filter lists

| List | Rules | Status |
|------|-------|--------|
| AdGuard DNS filter | ~161k | Enabled |
| HaGeZi's Pro++ Blocklist | ~228k | Enabled |
| AdAway Default Blocklist | - | Disabled |

### Migrating from Pi-hole

- Import Pi-hole blocklists into AdGuard: Filters > DNS
  Blocklists > Add blocklist > paste the same adlist URLs
- Pi-hole Teleporter backup (v6): export via web UI at
  `https://<pi-ip>/admin` > Settings > Teleporter > Export

### Pi-hole backup (direct file method)

```bash
# Create backup archive of key config files
sudo tar czf /tmp/pihole-backup.tar.gz \
  -C /etc/pihole \
  pihole.toml gravity.db adlists.list dhcp.leases pihole-FTL.db

# Copy to local machine
scp user@<pi-ip>:/tmp/pihole-backup.tar.gz ./
```

**Warning**: `/tmp/` is cleared on reboot. Transfer backups
immediately or save to a persistent path.

---

## SSH access

### Recommended: Advanced SSH & Web Terminal

Community add-on maintained by Frenck (HA core developer).
Preferred over the official Terminal & SSH add-on for its richer
toolset.

| Feature | Official Terminal & SSH | Advanced SSH & Web Terminal |
|---------|----------------------|---------------------------|
| Shell | Bash/ZSH | ZSH + Oh My ZSH |
| Tools | nano, vi | git, rsync, nmap, tmux, curl, vim + more |
| SFTP | No | Yes (requires `username: root`) |
| Docker access | No | Yes (protection mode off) |
| Custom packages | Limited | Full Alpine packages |
| Auth | Key OR password | Key AND/OR password |
| `ha` CLI | Yes | Yes |

### Installation

1. Enable **Advanced Mode**: Profile (bottom-left) > toggle on
2. Add community repo (if not present): Settings > Add-ons >
   Add-on Store > three-dot menu > Repositories > add:
   `https://github.com/hassio-addons/repository`
3. Search and install **Advanced SSH & Web Terminal**
4. Configure (Configuration tab):

```yaml
ssh:
  username: <your-username>
  password: ""
  authorized_keys:
    - "<your-ed25519-public-key>"
  sftp: false
  compatibility_mode: false
  allow_agent_forwarding: false
  allow_remote_port_forwarding: false
  allow_tcp_forwarding: false
zsh: true
share_sessions: true
packages: []
init_commands: []
```

5. Set port to `22` in the **Info** tab under **Network**
6. Enable **Start on boot** and **Watchdog**
7. Click **Start**

### Connecting

```bash
ssh <your-username>@<HA_IP> -p 22
```

Accessible via Tailscale from anywhere on the Tailnet without
exposing port 22 to the internet.

### System monitoring commands (`ha` CLI)

```bash
ha host info          # hostname, OS version, uptime
ha os info            # OS version, update available
ha core info          # HA core version, status
ha supervisor info    # supervisor version, status
ha hardware info      # connected hardware
ha network info       # network configuration
ha backups list       # list backups
ha backups new        # create a new backup
```

Standard Linux commands also work inside the container:

```bash
free -h               # memory usage
df -h                 # disk usage
top -bn1              # CPU/process snapshot
```

### Host-level SSH (port 22222, for debugging only)

Direct root access to the underlying HA OS. Not for daily use.

1. Create a USB drive with a partition labeled `CONFIG`
   (case-sensitive, FAT/ext4/NTFS)
2. Place an `authorized_keys` file (POSIX line endings) with
   your public key
3. Plug in and reboot, or run `ha os import` from the add-on
   SSH session

```bash
ssh root@<HA_IP> -p 22222
```

This gives access to Docker, systemd, journalctl, and the full
host filesystem. Useful commands:

```bash
docker logs homeassistant          # HA core logs
docker logs hassio_supervisor      # supervisor logs
journalctl -f                      # live system journal
```

### Security notes

- Use **Ed25519 keys only** (RSA is no longer supported)
- Keep **protection mode ON** unless you need Docker CLI access
- Do not expose SSH to the internet; use Tailscale for remote
  access
- Only run one SSH add-on to avoid port conflicts

---

## Dashboard

### Architecture

The dashboard uses **Mushroom Cards** for entity controls and
**Bubble Card** for the bottom navigation bar. It is a YAML-mode
dashboard stored at `/config/ui-dashboard-home.yaml`. Changes
appear on browser refresh (Cmd+Shift+R) without restarting HA.

Local backup: `configs/dashboards/ui-dashboard-home.yaml`

### Views

| View | Contents |
|------|----------|
| Home | Room toggles with brightness, media players, system chips |
| Living Room | 4 Hue lights (individual), 8 Hue scenes, TV + full remote |
| Kitchen | 3 Hue lights (individual), 5 Hue scenes |
| Bedroom | 2 WiZ lights, room temperature sensor, Dreo fan (speed + 5 presets), Adaptive Lighting |

### Key patterns

**Light card** (Mushroom):
```yaml
type: custom:mushroom-light-card
entity: light.living_room_a
use_light_color: true
show_brightness_control: true
show_color_temp_control: true
```

**Scene button** (Mushroom):
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

**TV remote button** (Mushroom + Bravia `remote.send_command`):
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

Available Bravia IRCC commands: `Up`, `Down`, `Left`, `Right`,
`Confirm`, `Return`, `Home`, `VolumeUp`, `VolumeDown`, `Mute`,
`ChannelUp`, `ChannelDown`, `Play`, `Pause`, `Stop`, `Netflix`.

**Fan card** (Mushroom):
```yaml
type: custom:mushroom-fan-card
entity: fan.air_circulator
show_percentage_control: true
show_oscillate_control: true
```

**Bottom nav bar** (Bubble Card):
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
```

### Dashboard design notes

- Mushroom Cards preferred over Bubble Card for entity controls
  (cleaner, simpler syntax)
- Bubble Card used for navigation bar and pop-up overlays
- Pure Bubble Card dashboard was tested but reverted; backup at
  `configs/dashboards/ui-dashboard-bubble-card.yaml` and on the
  Pi at `/config/ui-dashboard-home.yaml.bak`
- Cards: browser refresh only. Integrations: full HA restart.
- Frontend cards install via HACS > Frontend. Integrations via
  HACS > Integrations.

---

## Sony Bravia remote commands

Full list of `remote.send_command` IRCC codes for XR-55X90J:

**Navigation**: `Up`, `Down`, `Left`, `Right`, `Confirm`,
`Return`, `Home`

**Volume/Channel**: `VolumeUp`, `VolumeDown`, `Mute`,
`ChannelUp`, `ChannelDown`

**Playback**: `Play`, `Pause`, `Stop`, `Rewind`, `Forward`

**Apps**: `Netflix`, `YouTube`

**Power**: handled by `media_player.living_room_tv_2` turn_on/off,
not via remote commands

**AirPlay note**: Enable AirPlay but disable HomeKit on the TV
(Settings > Apps > AirPlay & HomeKit) to get Apple TV Remote in
Control Center without causing duplicate entries in Apple Home.

---

## Automations

### Reload Dreo on startup

The Dreo integration depends on a cloud API login. After a power
outage, HA may start before DNS (AdGuard) is ready, causing Dreo
to fail with "No response from API". This automation waits 60s
after startup, checks if the fan is unavailable, and reloads the
config entry.

Config entry ID: `<DREO_ENTRY_ID>`

Manual reload via Supervisor API (from SSH):

```bash
curl -s -X POST \
  "http://supervisor/core/api/config/config_entries/entry/<DREO_ENTRY_ID>/reload" \
  -H "Authorization: Bearer ${SUPERVISOR_TOKEN}"
```

---

## Backups

### HA backup via CLI

```bash
ssh root@<HA_IP> -p 22 \
  "ha backups new --name 'my-backup'"
```

### Local config backups

Config files are saved to this repo under `configs/`:

```
configs/
  ha/
    configuration.yaml    — main HA config
    automations.yaml      — HA automations
  dashboards/
    ui-dashboard-home.yaml       — active Mushroom dashboard
    ui-dashboard-bubble-card.yaml — Bubble Card backup
  adguard/
    dns-config.json       — DNS/cache/upstream settings
    filtering-config.json — filter lists and rules
    access-config.json    — access controls
    rewrites-config.json  — DNS rewrites
```

---

## HACS installation

Requires the Advanced SSH add-on with `username: root`:

```bash
ssh root@<HA_IP> -p 22 \
  "wget -O - https://get.hacs.xyz | bash -"
```

Then restart HA and add the integration: Settings > Devices &
Services > + Add Integration > search "HACS". Requires GitHub
account linking for API access.

After HACS is installed, switch the SSH username back to your
preferred user.

---

## Adaptive Lighting

Set-and-forget integration that shifts light color temperature
throughout the day.

### Setup

1. Install via HACS > Integrations > "Adaptive Lighting"
2. Restart HA
3. Settings > Devices & Services > + Add Integration >
   "Adaptive Lighting"
4. Create one switch per room, selecting the lights

### Entities created per room

| Entity | Purpose |
|--------|---------|
| `switch.adaptive_lighting_<name>` | Main on/off toggle |
| `switch.adaptive_lighting_adapt_brightness_<name>` | Brightness adaptation |
| `switch.adaptive_lighting_adapt_color_<name>` | Color adaptation |
| `switch.adaptive_lighting_sleep_mode_<name>` | Sleep mode (very warm) |

### Behavior

- Intercepts `light.turn_on` and overrides color temperature
- Respects manual color changes (stops adapting until next
  on/off cycle)
- Sleep mode sets lights to very warm amber

---

## Tips

- **SD card longevity**: Use A2-rated cards (Samsung EVO Select,
  SanDisk Extreme). Consider moving the HA database to a USB SSD
  for better write endurance.
- **Static IP**: Set a DHCP reservation for the Pi in your router
  to keep `<HA_IP>` stable.
- **Ethernet over WiFi**: HA, AdGuard (DNS), and Tailscale all
  benefit from a stable wired connection. WiFi works but risks
  intermittent DNS outages and device unavailability.
- **New entities auto-sync**: After initial HomeKit pairing, newly
  added HA entities appear in Apple Home automatically (no
  re-pair needed), as long as their domain is included in the
  bridge config.
- **Cloud-dependent integrations**: Dreo (and similar cloud-based
  integrations) fail silently if DNS is not ready at boot. Add
  a startup automation to reload them after a delay. See
  "Automations" section.
- **IPv6 false positives**: Many home routers assign local IPv6
  addresses (`fd02:`, `fe80:`) without providing actual IPv6
  internet. This causes apps to try IPv6 first, hang, then fall
  back to IPv4. Disable IPv6 in AdGuard (`disable_ipv6: true`)
  unless your ISP provides real IPv6 connectivity.
- **Matter bridging**: Matterbridge add-on
  (`https://github.com/Luligu/matterbridge-home-assistant-addon`)
  can expose HA devices as Matter devices to Google Home/Alexa.
  Requires setting the mDNS interface to the physical adapter
  (e.g. `end0`). Still maturing; native Google Home integration
  may be more reliable for now.
