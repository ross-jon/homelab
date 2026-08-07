# Checks — Invariant Registry & Latest Audit

_Machine source: `$HOMELAB_CONTEXT/invariants.yaml` + `checks/latest.json`.

## Registry

| ID | Category | Check | Cadence | Target |
| --- | --- | --- | --- | --- |
| INV-001 | security | port_allowlist | nightly | PASS |
| INV-002 | ingress | route_health | nightly | PASS |
| INV-003 | security | cert_expiry | daily | PASS |
| INV-004 | ops | casaos_registration | weekly | PASS |
| INV-005 | ops | image_age | weekly | PASS |
| INV-006 | data | backup_status | weekly | PASS |

## Latest Audit

| ID | Status | Detail |
| --- | --- | --- |
| INV-001 | FAIL | 25 ports exposed beyond allowlist: bazarr:6767, big-bear-dockhand:3003, big-bear-kavita:5000, big-bear-seerr:5056, big-bear-syncthing:8384, flaresolverr:8191, gluetun:55061, gluetun:60642, gluetun:8181, gluetun:8388, gluetun:8888, grafana:3000, hermes-hermes-1:8642, hermes-hermes-1:9119, influxdb:8086, jellyfin:8096, lidarr:8686, mylar3:8090, navidrome:4533, nzbget:6789, overseerr:5555, prowlarr:9696, radarr:7878, readarr-readarr-1:8787, sonarr:8989 |
| INV-002 | PASS | all 16 Caddy upstreams resolve |
| INV-003 | PASS | cert expires in 78d, renewal scheduled (hermes cron cert-renew) |
| INV-004 | PASS | all 30 containers registered or legacy-accepted |
| INV-005 | FAIL | 3 images older than 180d: grafana (654d); qbittorrent (485d); matter-server (358d) |
| INV-006 | FAIL | no backup job/script found (CON-002 SPOF unmitigated) |
