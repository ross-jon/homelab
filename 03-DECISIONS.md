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
