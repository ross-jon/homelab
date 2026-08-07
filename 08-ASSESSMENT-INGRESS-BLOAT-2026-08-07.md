# Assessment — Ingress security & container bloat (2026-08-07)

Status: **ASSESSMENT ONLY — no changes made.** Source: live docker/caddy/tailscale inspection (read-only).
Goal: scope tickets for (a) eliminating unsecure-port access, (b) container bloat cleanup.

---

## 1. The "unsecure ports" problem — root causes found

### 1.1 HTTPS ingress IS working — but it's not what you're using
- Caddy terminates TLS on `:443` (Tailscale-issued Let's Encrypt cert, valid **until Oct 25 2026**, verified via SNI handshake). `tailscale serve` is OFF (correct — it's broken on this kernel).
- `https://homelab.taild32764.ts.net/<app>` serves the whole app suite: radarr/sonarr/lidarr/readarr/prowlarr/bazarr/jellyfin/kavita/navidrome/overseerr/seerr/mylar3/qbittorrent/homeassistant/esphome/syncthing/livesync/vaultwarden. Route health tested: all 200/302/307 EXCEPT `/hermes` (see 1.2).
- **Every app ALSO publishes `0.0.0.0:<port>` on the host** (CasaOS compose default). Result: the ENTIRE suite is reachable raw via `http://192.168.86.29:<port>` — no TLS, bypassing Caddy. Verified live: radarr:7878→307, grafana:3000→200, syncthing:8384→200, dockhand:3003→200, flaresolverr:8191→200, influxdb:8086→200, nzbget:6789→401.

### 1.2 `/hermes` route is BROKEN — this is likely today's specific pain
- Caddy → `hermes:9119` returns **502 / SERVFAIL**. Cause: `hermes-hermes-1` is on `hermes_default` network only (NOT `web_proxy`), and the container name is `hermes-hermes-1`, not `hermes` (Docker DNS needs exact name).
- **`/readarr` is ALSO broken** (same bug class): Caddyfile targets `readarr:8787`, container is `readarr-readarr-1` on `web_proxy` with no `readarr` alias → SERVFAIL. Discovered by the INV-002 audit on 2026-08-07 (missed in the manual route sweep).
- Consequence: **Hermes is currently reachable ONLY via raw `http://192.168.86.29:9119` (and `:8642`)** — i.e., exactly the "unsecure port" you're complaining about, for the very tool you're using right now. Readarr raw `http://192.168.86.29:8787` works but bypasses TLS.

### 1.3 Apps with raw ports and NO Caddy route at all
| Port | App | Risk |
|---|---|---|
| 3003 | dockhand (container manager) | **CRITICAL — no auth** (see 1.4) |
| 3000 | grafana | login required, but cleartext over LAN HTTP |
| 8086 | influxdb | token-auth API exposed raw |
| 6789 | nzbget | auth required but likely **default creds** (see 1.4) |
| 8191 | flaresolverr | no auth by design — open CAPTCHA-solver proxy on LAN |
| 8642/9119 | hermes | gateway + UI raw (route broken) |
| 55061/60642/8388/8888 | qBittorrent via gluetun | P2P ports published on LAN despite VPN firewall on + port-forwarding OFF — probably useless and unnecessary |

### 1.4 Auth posture of raw-exposed UIs
- **dockhand :3003 — NO authentication at all** (no auth env, no users in `/app/data`). Anyone on the LAN can open it and manage containers → effectively host compromise. **P0.**
- **nzbget :6789 — `ControlUsername`/`ControlPassword` commented out in config → default `nzbget:tegbzn6789`.** Well-known default on the LAN. **P0/P1.**
- syncthing :8384 — REST returns 403 unauthenticated → auth enforced. OK.
- grafana :3000 — `/api/user` → 401 → anonymous disabled. OK-ish.

### 1.5 Why you still type `http://192.168.86.29:PORT`
- **Tailscale on your phone (jons-s23) has been offline 4 days**; desktop offline 19h. The tailnet FQDN only resolves on tailnet-joined devices → when the phone's tailscale is off, the ONLY paths are raw LAN IPs.
- CasaOS "Open" buttons are configured to `http://192.168.86.29:8080/<app>` (plain HTTP) — the dashboard actively points at the insecure path.
- There is **no LAN TLS story**: no internal CA, no cert for the LAN IP, `:8080` Caddy block is plain HTTP by design.

---

## 2. Bloat — numbers

### 2.1 Images (52 images, 27.94 GB total, 3.93 GB reclaimable)
- **Truly dangling (safe to prune): 3 images, ~3.9 GB** (untagged, 5 days old — stale hermes/livesync rebuilds).
- **Untagged-but-IN-USE** (do NOT prune — CasaOS tag churn makes them look like bloat): HA 3.34GB, music-assistant 2.15GB, seerr 1.32GB, kavita 881MB, dockhand 685MB, matter-server 614MB, couchdb 435MB. The `docker images` list is misleading; these are live.
- **Old tagged images, review candidates**: `linuxserver/readarr:0.3.10-develop` (263MB, 2y — running image is faustvii), `docker/compose:latest` (112MB, 6y), `appropriate/curl` (9.9MB, 8y), `alpine:3.18` (17mo), 4× python, 2× node, 2× curlimages, nginx:alpine, busybox, alpine/git, vaultwarden/server:latest (dup of 1.36.0).

### 2.2 Containers (32 total, 30 running)
- **2 dead**: `hermes_old` (Exited today — recreate leftover), `relaxed_lederberg` (alpine, Created Aug 1 — stray, never ran).
- **Duplicate functionality**: **overseerr AND seerr both running** (~530MB RAM combined) — seerr is a fork of overseerr. Pick one.
- **CasaOS registry mismatch**: 17 apps registered in `app_order.json` vs 30 running → 14 containers show as "legacy" in the CasaOS UI (caddy, couchdb, grafana, influxdb, jellyfin, lidarr, sonarr, prowlarr, radarr, nzbget, overseerr, syncthing, esphome, livesync-daemon). Cosmetic (apps work), but a housekeeping signal.

### 2.3 Networks (14) / Volumes (8) / Disk / RAM
- Orphan networks: `test-app_default`, `hermes_default` (actively harmful — isolates hermes from Caddy), `big-bear-obsidian-livesync_default`.
- 3 orphan volumes, 49MB (negligible).
- Disk: **50.5G / 97.9G (54%)**; `/dev/sdb` = 2.7G **Ubuntu Server 26.04 install USB left mounted at 100%** — unmount/eject.
- RAM: **~3.8GB across 30 containers** on a 7.5GB host (3.5GB free). Top: hermes 572MB, HA 524MB, seerr 362MB, kavita 210MB, prowlarr 191MB, livesync-daemon 176MB, overseerr 170MB.

### 2.4 Update hygiene (not bloat, but housekeeping)
- Several images months old: tailscale v1.98.3, gluetun v3.41.1, qbittorrent release-5.0.4 (16mo), HA 2026.5.1, grafana 11.3.0, kavita 0.9.0.
- **Cert renewal is NOT scheduled** — renewal script exists (`/opt/data/scripts/renew-tailscale-cert.sh`) but no host cron; cert expires **Oct 25 2026**. TLS will silently break if forgotten.

---

## 3. Proposed ticket backlog (scope only — no work done)

### P0 — security
- **T-1** Bind ALL app host ports to `127.0.0.1` (or remove published ports; keep only Caddy's :80/:443/:8080/:10380 + syncthing sync ports 22000/21027 if LAN sync needed). ~30 compose edits. Effort: M. Risk: L-M (restart churn).
- **T-2** Add auth to dockhand (or bind to localhost + route via Caddy with basic-auth) + fix its "Open" metadata.
- **T-3** Set non-default NZBGet creds (or bind 6789 to localhost; it has no Caddy route).

### P1 — ingress correctness
- **T-4** Fix `/hermes` + `/readarr`: connect `hermes-hermes-1` to `web_proxy` (add alias `hermes`) and add alias `readarr` for `readarr-readarr-1` (or fix Caddyfile targets). Then raw 9119/8642/8787 can be localhost-bound. Effort: S.
- **T-5** Decide LAN TLS: (a) internal CA (step-ca/mkcert) so LAN devices trust `https://192.168.86.29`, (b) require Tailscale everywhere + fix phone app, (c) accept LAN HTTP but restrict raw ports. Needs Jon's call.
- **T-6** Schedule cert renewal cron on host (expires Oct 25).

### P2 — bloat
- **T-7** `docker image prune` (3.9GB dangling) + remove confirmed-dead containers `hermes_old`, `relaxed_lederberg`.
- **T-8** Pick one request manager: overseerr or seerr; remove the other (+ its image). ~530MB RAM.
- **T-9** Audit old tagged images (readarr-develop, docker/compose, appropriate/curl, python×4, node×2, etc.) — keep only what CasaOS compose runner actually uses.
- **T-10** Remove orphan networks (test-app_default, big-bear-obsidian-livesync_default, hermes_default after T-4) + unmount the Ubuntu installer USB (/dev/sdb).
- **T-11** CasaOS registry cleanup: import the 14 legacy containers into app_order.json (or accept legacy status) — cosmetic.
- **T-12** Version hygiene pass: tailscale, gluetun, qbittorrent, HA, grafana, kavita.

---

## 4. Decisions needed from Jon (to scope T-5 and T-8 mainly)
1. LAN TLS: internal CA vs Tailscale-everywhere vs accept-HTTP? (Phone tailscale offline 4d is the elephant.)
2. overseerr or seerr? (Both currently deployed.)
3. Is dockhand still needed? If yes, what access model (Tailscale-only? Caddy + auth?).
4. Are qBittorrent's LAN P2P ports needed (8388/8888/55061/60642)? Any LAN torrent peers?
5. Is the Ubuntu installer USB still needed on the server?
