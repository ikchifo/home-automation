# Home Assistant plugin for Claude Code

A Claude Code plugin for managing Home Assistant instances via
the REST API, `hass-cli`, and SSH.

## Install

### As a plugin (recommended)

Test locally:

```bash
git clone https://github.com/chifo/home-automation.git
claude --plugin-dir ./home-automation
```

Or install from a marketplace if this repo has been added as
one (see
[plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)):

```
/plugin install homeassistant@<marketplace-name>
```

### As a standalone skill

Copy into your Claude Code skills directory:

```bash
git clone https://github.com/chifo/home-automation.git /tmp/ha
cp -r /tmp/ha/skills/homeassistant ~/.claude/skills/
```

## Setup

Set these environment variables before using the skill:

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
- Manage integrations through config flows and options flows
- Deploy and verify automations
- Troubleshoot unavailable entities, HomeKit Bridge, and DNS
- Send natural language commands via the Conversation API
- Edit YAML-mode dashboards (Mushroom Cards, Bubble Card)

The skill prefers the REST API for direct operations,
`hass-cli` for quick queries, and SSH only for system-level
tasks like restart and log streaming.

## Plugin structure

```
.claude-plugin/
  plugin.json              — plugin manifest
skills/
  homeassistant/
    SKILL.md               — main skill
    references/
      rest-api.md          — full REST API endpoint reference
      hass-cli.md          — CLI command reference
      dashboards.md        — Mushroom/Bubble Card patterns
```

## Other repo contents

This repo also includes config backups and a setup guide for
a Home Assistant instance on a Raspberry Pi 5.

```
AGENTS.md                  — full setup guide and reference
configs/
  ha/                      — core HA config and automations
  dashboards/              — Lovelace YAML dashboards
  adguard/                 — AdGuard DNS and filtering config
scripts/
  hooks/                   — git hooks (secrets scanner)
```

## Contributing

After cloning, activate the pre-commit hook:

```bash
git config core.hooksPath scripts/hooks
```

This scans staged files for private IPs, emails, tokens, and
other sensitive data before each commit.
