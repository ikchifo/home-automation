# Home Assistant plugin for Claude Code

A Claude Code plugin for managing Home Assistant instances via
the REST API, `hass-cli`, and SSH.

## Install

Add this repo as a marketplace, then install the plugin:

```
/plugin marketplace add ikchifo/home-automation
/plugin install homeassistant@home-automation
```

Or test locally:

```bash
git clone https://github.com/ikchifo/home-automation.git
claude --plugin-dir ./home-automation/plugins/homeassistant
```

### Standalone skill (no plugin system)

```bash
git clone https://github.com/ikchifo/home-automation.git /tmp/ha
cp -r /tmp/ha/plugins/homeassistant/skills/homeassistant \
  ~/.claude/skills/homeassistant
```

## Setup

Set these environment variables:

```bash
export HASS_SERVER=http://<your-ha-ip>:8123
export HASS_TOKEN=<your-long-lived-access-token>
```

Create a token in Home Assistant at:
Profile > Long-Lived Access Tokens > Create Token.

Optionally install the CLI:

```bash
uv tool install homeassistant-cli
```

## What it does

- Query and control entities via the HA REST API
- Call services (lights, scenes, fans, media players)
- Manage integrations through config flows and options
  flows
- Deploy and verify automations
- Troubleshoot unavailable entities, HomeKit Bridge, DNS
- Send natural language commands via the Conversation API
- Edit YAML-mode dashboards (Mushroom Cards, Bubble Card)

## Plugin structure

```
plugins/
  homeassistant/
    .claude-plugin/
      plugin.json            — plugin manifest
    skills/
      homeassistant/
        SKILL.md             — main skill
        references/
          rest-api.md        — REST API endpoint reference
          hass-cli.md        — CLI command reference
          dashboards.md      — dashboard card patterns
```

## Other repo contents

Config backups and setup guide for a Home Assistant instance
on a Raspberry Pi 5.

```
AGENTS.md                    — full setup guide and reference
configs/
  ha/                        — core HA config and automations
  dashboards/                — Lovelace YAML dashboards
  adguard/                   — AdGuard DNS and filtering config
scripts/
  hooks/                     — git hooks (secrets scanner)
```

## Contributing

After cloning, activate the pre-commit hook:

```bash
git config core.hooksPath scripts/hooks
```
