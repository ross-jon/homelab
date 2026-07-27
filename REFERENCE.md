# Homelab Reference — Operational Details

**Last updated:** 2026-07-27  
**Source of truth:** this repo (synced Obsidian ↔ GitHub). Memory holds only pointers.

---

## 1. External Ingress (VERIFIED WORKING)

- **Caddy** terminates TLS on `:443` using a Tailscale-issued Let's Encrypt cert.
  - Cert: `/DATA/AppData/tailscale/certs/homelab.taild32764.ts.net.{crt,key}`
  - Mounted into caddy at `/etc/caddy/certs` (read-only)
  - Domain: `homelab.taild32764.ts.net` → resolves to Tailscale IP `100.103.106.49`
- **`tailscale serve` is REMOVED** — it was broken (containerized Tailscale cannot
  initialize iptables due to kernel secureboot/lockdown; "Operation not permitted"
  even with NET_ADMIN). Caddy now does all TLS + routing.
- Caddy also listens on `:8080` (plain HTTP) for LAN / CasaOS dashboard access.
- **Recreate caddy** (started via `docker run`, NOT CasaOS compose):
  ```bash
  docker run -d --name caddy --network web_proxy \
    --add-host host.docker.internal:host-gateway \
    --restart unless-stopped \
    -p 8080:8080 -p 443:443 -p 10380:10380 \
    -v /DATA/AppData/caddy/Caddyfile:/etc/caddy/Caddyfile \
    -v /DATA/AppData/caddy/data:/data \
    -v /DATA/AppData/tailscale/certs:/etc/caddy/certs:ro \
    caddy:alpine caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
  ```
- **Cert renewal:** Let's Encrypt cert expires ~Oct 25 2026. Renew via
  `/opt/data/scripts/renew-tailscale-cert.sh` (runs `tailscale cert` + restarts caddy).
  MUST be scheduled on HOST cron (not hermes container). Example:
  `0 3 1 * * /opt/data/scripts/renew-tailscale-cert.sh >> /opt/data/logs/cert-renew.log 2>&1`

### Key ingress gotchas (learned the hard way)
1. `tailscale cert` MUST generate cert+key in ONE command. Running it twice with
   different `--key-file` paths produced a cert whose key did NOT match (TLS
   "internal error"). Verify with openssl pubkey comparison.
2. `host.docker.internal` is NOT auto-resolved on Linux user-defined networks.
   Must pass `--add-host host.docker.internal:host-gateway` to caddy.
   Apps using it: qbittorrent/gluetun:8181, homeassistant:8123, esphome:6052,
   syncthing:8384.
3. Caddy snippet order: snippets (`(routes)`, `(fwd)`) must be DEFINED before
   they are `import`ed, or Caddy errors "File to import not found".
4. Test with SNI = domain (not IP). Connecting to the IP with SNI=IP fails (no cert
   for IP). Real browsers set SNI=domain, so this only affects testing.

---

## 2. ARR Stack + Media Apps

| App | Port | Network | Notes |
|-----|------|---------|-------|
| Radarr | 7878 | web_proxy | Group A (owns /radarr base path) |
| Sonarr | 8989 | web_proxy | Group A |
| Lidarr | 8686 | web_proxy | Group A |
| Readarr | 8787 | web_proxy | Faustvii fork (rreading-glasses); MetadataProvider src https://api.bookinfo.pro via /config/development.metadataSource. Runs as nobody(65534) → /DATA/AppData/readarr/config must be chown 65534:65533 or DB migration fails (readonly). |
| Prowlarr | 9696 | web_proxy | + 3 indexers (TPB, YTS, LimeTorrents); FlareSolverr(8191) |
| Bazarr | 6767 | web_proxy | Group A |
| qBittorrent | 8080 (behind gluetun) | gluetun netns | Published via gluetun :8181. Caddy → host.docker.internal:8181 |
| Jellyfin | 8096 | web_proxy | Group B |
| Navidrome | 4533 | web_proxy | Group A |
| Kavita | 5000 | web_proxy | Group B (no strip, handle /kavita/*) |
| Mylar3 | 8090 | web_proxy | Group B |
| Overseerr | 5055 | web_proxy | Group B |
| Seerr | 5055 | web_proxy | big-bear-seerr; Group B |
| Vaultwarden | 80 | web_proxy | /vaultwarden (no strip) + API paths |
| Syncthing | 8384 | web_proxy | Group B → host.docker.internal:8384 |
| Home Assistant | 8123 | host | network_mode: host → host.docker.internal:8123 |
| ESPHome | 6052 | host | network_mode: host → host.docker.internal:6052 |
| Hermes | 9119 | web_proxy | Group B |

### Download paths
- qB save path: `/DATA/Downloads/qBittorrent` (shared, 1000:1000)
- *arr mount `/DATA/Downloads` → `/downloads`
- Remote Path Mapping (RPM): host `172.17.0.1`, remotePath `/DATA/Downloads/qBittorrent`, localPath `/downloads/qBittorrent`
- Credentials: in Vault Warden
- Obsidian vault: `/DATA/Documents/Notes` (Daoist framework, 1418 notes)

---

## 3. Network Topology

- **web_proxy**: Docker network all web-facing app containers + Caddy join. Enables
  Caddy → container-name DNS resolution. Apps NOT on it (HA, esphome host-mode) are
  reached via `host.docker.internal`.
- **gluetun netns**: qBittorrent shares gluetun's network namespace (VPN tunnel).
- **Tailscale**: container in host network mode; provides device auth + tunnel to
  `homelab.taild32764.ts.net`. Certs issued via `tailscale cert`.

---

## 4. Progress Log

| Date | Task | Status | Notes |
|------|------|--------|-------|
| 2026-07-27 | Phase 1.1 Backup | ✅ | All compose files, container/network state backed up |
| 2026-07-27 | Phase 1.2 Rebuild Caddy | ✅ | New Caddyfile, all 18 routes verified |
| 2026-07-27 | Phase 1.3 Tailscale integration | ✅ | Option B: Caddy TLS, tailscale serve removed |
| 2026-07-27 | CasaOS metadata (1.4) | ⏳ | Pending — icons + web URLs in CasaOS |
| 2026-07-27 | Network membership (1.5) | ⚠️ | kavita/navidrome/seerr added to web_proxy; verify all |
| 2026-07-27 | Cert renewal | ⚠️ | Script written; needs HOST cron scheduling |

---

## 5. Key Findings

- **Original "legacy CasaOS apps" issue**: was a red herring / already resolved by
  compose labels. The REAL blocker was the broken `tailscale serve` ingress.
- **`tailscale serve` is fundamentally broken in this containerized setup** (iptables
  kernel lockdown). Do NOT attempt to fix it via iptables module loading — kernel
  lockdown blocks it ("Operation not permitted" even with NET_ADMIN).
- **Option B (Caddy TLS)** is the robust ingress. All 18 apps verified 200/30x.
- **Vaultwarden URLs in Bitwarden**: should point to
  `https://homelab.taild32764.ts.net/vaultwarden` (Task 1.4 / Vaultwarden URL audit).
