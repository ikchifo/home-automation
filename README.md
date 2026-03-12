# home-automation

Local config backups and reference docs for my Home Assistant
setup running on a Raspberry Pi 5 (8GB).

## What's running

- **Home Assistant OS**
- **AdGuard Home** for DNS-level ad blocking
- **Tailscale** for VPN and remote access
- **HomeKit Bridge** for Siri/Apple Home control
- **HACS** with Mushroom Cards, Bubble Card, Adaptive Lighting,
  and Dreo fan integration

## Repo structure

```
AGENTS.md              — full setup guide and reference
configs/
  ha/                  — core HA config and automations
  dashboards/          — Lovelace YAML dashboards
  adguard/             — AdGuard DNS and filtering config
skills/
  homeassistant/       — Claude Code skill for HA management
```

## Devices

Philips Hue lights (10), WiZ lights (2), Sony Bravia TV,
Dreo fan, Google Hub Max.

## Key docs

See [AGENTS.md](AGENTS.md) for detailed setup instructions,
API reference, troubleshooting notes, and dashboard patterns.
