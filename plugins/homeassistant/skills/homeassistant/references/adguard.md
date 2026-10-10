# AdGuard Home reference

AdGuard Home commonly runs as a Home Assistant add-on on the
same host. Its REST API (`/control/*`) is reachable from the
host's SSH add-on session on AdGuard's own HTTP port — an
instance-specific value, not the ingress port. Record it in the
instance's notes once found (it also appears in the add-on logs
at startup, and in `GET /control/status` as `http_port`).

```bash
ADGUARD="http://127.0.0.1:<adguard_http_port>/control"
```

## Configure via API

`POST /control/dns_config` takes partial updates: fields you
omit keep their current values.

```bash
# Read current DNS settings
curl -s $ADGUARD/dns_info | jq .

# Change one setting
curl -s -X POST $ADGUARD/dns_config \
  -H "Content-Type: application/json" \
  -d '{"disable_ipv6": true}'

# Replace the upstream list
curl -s -X POST $ADGUARD/dns_config \
  -H "Content-Type: application/json" \
  -d '{"upstream_dns": ["https://dns.google/dns-query", "1.1.1.1"]}'
```

Upstream syntax notes:

- `https://host/dns-query` — DoH
- `h3://host/dns-query` — DoH over HTTP/3; requires working
  outbound QUIC (UDP 443). If the path blocks it, the upstream
  silently never answers — check `GET /control/stats`
  (`top_upstreams_responses`) for upstreams with 0 responses.
- `[/<internal-domain>/]<dns_server_ip>` — route only matching
  domains to that server
- `upstream_mode: "parallel"` races all upstreams and takes the
  first answer. Plain UDP (e.g. `1.1.1.1`) almost always wins,
  so DoH entries are cosmetic unless parallel mode is off.

## User rules (allowlist/blocklist overrides)

AdGuard Home v0.107.79 removed `POST /control/filtering/add_rule`
(it returns 404). Set user rules wholesale instead:

```bash
# Replace all user rules (array of plain strings)
curl -s -X POST $ADGUARD/filtering/set_rules \
  -H "Content-Type: application/json" \
  -d '{"rules": ["@@||graph.instagram.com^"]}'

# Read current rules
curl -s $ADGUARD/filtering/status | jq .user_rules
```

An exception rule (`@@||domain^`) overrides any blocklist entry.
Check the add-on version with `GET /control/status` before using
older endpoint names.

## Diagnosing with the query log

`GET /control/querylog?limit=N` returns recent queries with
`question.name`, `reason` (`NotFilteredDefault`,
`FilteredBlackList`, ...), the matched `rules` (including
`filter_list_id`), and `elapsed_ms`.

```bash
# What did a filter block recently?
curl -s "$ADGUARD/querylog?limit=500" | jq '[.data[]
  | select(.reason != "NotFilteredDefault")
  | {q: .question.name, reason, rules}]' | head -40

# Map filter_list_id to a filter list name
curl -s $ADGUARD/filtering/status \
  | jq '.filters[] | {id, name, rules_count, enabled}'

# Upstream health: response counts and average latency
curl -s $ADGUARD/stats | jq '.top_upstreams_responses, .top_upstreams_avg_time'
```

Watch for aggressive blocklists (e.g. HaGeZi Pro++) blocking app
API endpoints — apps then misbehave (media fails to load until a
manual retry) while the ad CDNs themselves are left alone.

## Tailscale + AdGuard DNS architecture

A common setup makes AdGuard the tailnet's global nameserver.
That routes every client's DNS through the WireGuard tunnel to
the HA host. On iOS this creates wake-stalls: iOS suspends the
VPN when locked, and tailscaled needs 1–5 s to re-contact a peer
after suspension (`magicsock: new contact: peer=... usec=...`
in the add-on logs). Symptoms: apps hang on first load after
unlock, real-time calls dip for seconds, reload/retry fixes it.
The DNS server itself is fine — cached answers return in ~0 ms.

Fixes:

- Home LAN: point router DHCP DNS at the HA host's LAN IP so
  home clients hit AdGuard directly (no tunnel hop)
- Tailnet: remove the global nameserver but keep MagicDNS —
  `<tailnet>.ts.net` names still resolve via each client's local
  tailscaled, and general DNS falls back to the network's own DNS
- Away devices then use their network's DNS (no ad blocking
  off-LAN unless a client sets it explicitly)

Keep a domain-specific forward rule so AdGuard can answer
tailnet names for LAN clients that ask it directly:

```json
"upstream_dns": ["[/<tailnet>.ts.net/]100.100.100.100", ...]
```

## ISP router: LAN DNS restrictions

Some ISP gateways (e.g. Bell Giga Hub / Sagemcom) refuse LAN IPs
as DNS servers in the web UI, blocking local AdGuard setups. The
router's local JSON-RPC API can still set it — see community
tooling like `fernando-granco/Bell-Gigahub-Local-DNS` (DHCP
method points clients at your DNS server directly; relay keeps
the router as forwarder). Caveats: firmware-dependent (relay mode
on old firmware cuts the DNS server off the network), ISP remote
management may revert the change, and there is no secondary DNS
in DHCP mode. Verify with a DNS health check before and after.

## ISP gateway: Wi-Fi config over JSON-RPC

The same JSON-RPC API used for DNS settings can read and write
Wi-Fi radio config on Sagemcom-style gateways — handy when the
web UI hides channel selection:

- Read radios: `getValue` on `Device/WiFi/Radios` (channel,
  width, standards, auto-channel flags for each band)
- Set channel: `setValue` on `Device/WiFi/Radios/Radio[N]/Channel`.
  Instance paths need NUMERIC indices — the alias form shown in
  `Name` fields (`Radio[RADIO5G]`) returns `XMO_UNKNOWN_PATH_ERR`;
  check the `Device/WiFi/Radios` dump for order (one 5690:
  2.4GHz=1, 5GHz=2, 6GHz=3)
- Changing the channel briefly drops that band's clients. Save
  the original channel, read back the radio status after the
  write, and roll back if it does not return `UP`
- DFS channels (e.g. 100) force a 30+ minute channel evacuation
  on radar detection, dropping every client on that band —
  prefer non-DFS channels for latency-sensitive traffic
- Neighbor scan: setValue
  `Device/WiFi/NeighboringWiFiDiagnostic/DiagnosticsState` to
  `Requested`; poll the node until `COMPLETED` (~12s); results
  land in `Result`. Some units return zero entries (quiet RF
  environment or firmware never populates it)
