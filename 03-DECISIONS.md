# Decisions (rendered from decisions.yaml)

_Machine source: `$HOMELAB_CONTEXT/decisions.yaml` — edit there, re-render._

## DEC-001 — All public ingress terminated by Caddy using Tailscale-issued TLS
- **Status:** active · **Tags:** networking, security
- **Rationale:** Single TLS termination point; avoids per-app cert management and exposes one controlled entry.
- **Created:** 2026-07-27T22:05:03.266223+00:00

## DEC-002 — ARR stack (Sonarr/Radarr) and qBittorrent run behind the gluetun VPN container
- **Status:** active · **Tags:** privacy, networking
- **Rationale:** Keeps torrent traffic off the bare WAN IP; only VPN egress used for p2p.
- **Created:** 2026-07-27T22:05:03.406013+00:00

## DEC-003 — OpenCode Go backend is the preferred LLM backend; default model MiMo-V2.5
- **Status:** active · **Tags:** ai, tooling
- **Rationale:** Operator preference for a local/self-hosted inference path.
- **Created:** 2026-07-27T22:05:03.547859+00:00

## DEC-004 — Bulk storage is a single 6TB Seagate USB drive at /mnt/storage (no RAID)
- **Status:** active · **Tags:** storage
- **Rationale:** Cost/simplicity; accepted single-drive risk for media bulk store.
- **Created:** 2026-07-27T22:05:03.686101+00:00

## DEC-007 — Caddy upstream resolution fixed via Docker network aliases, not Caddyfile target edits
- **Status:** active · **Tags:** ingress, networking
- **Rationale:** Caddyfile is bind-mounted and fragile to edit (docker cp/sed fail; reload risk). docker network connect --alias is live, zero-file-risk, no reload. Verified 2026-08-07: hermes (web_proxy + alias hermes) and readarr (alias readarr) both 200 over TLS after alias fix. NOTE: DEC-005/DEC-006 are RESERVED for pending Jon decisions (JON-47 LAN trust boundary, JON-48 access UX).
- **Alternatives considered:** edit Caddyfile reverse_proxy targets to exact container names
- **Created:** 2026-08-07T21:53:49.779338Z

## DEC-008 — Tailscale TLS cert renewal runs as Hermes cron job 'cert-renew' (monthly, no_agent, docker-exec script), not a host cron
- **Status:** active · **Tags:** security, ops
- **Rationale:** Host cron is not schedulable from the sandbox; the renewal script is docker-socket-safe (docker exec tailscale cert + docker restart caddy) and cert files persist on the host volume (/DATA/AppData/tailscale/certs). Daily INV-003 audit is the safety net if renewal ever fails. Verified 2026-08-07: script runs clean from sandbox, caddy healthy after.
- **Alternatives considered:** host cron via /etc/cron.d (needs host shell), manual monthly renewal
- **Created:** 2026-08-07T21:55:00.000000Z
