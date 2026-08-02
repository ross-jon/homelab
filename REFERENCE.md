# Homelab Reference — Operational Details

**Last updated:** 2026-07-30  
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
| 2026-07-27 | CasaOS re-import — Phase 1 Caddy decouple | ✅ | 16/18 apps on `host.docker.internal`; vaultwarden (unpub) + overseerr (port-misconfig, being deleted by user) sole exceptions. External TLS verified working by user post-change. All 18 routes green. |
| 2026-07-27 | CasaOS re-import — Phase 2 UI re-import | ⏳ | Re-import apps via CasaOS for proper icons + clickable URLs; reconnect `web_proxy` for *arr name resolution. overseerr excluded (user deleting). |
| 2026-07-27 | CasaOS metadata (1.4) — Phase 2 re-import | ⏳ | UI re-import pending user CasaOS access; 2 exceptions (vaultwarden unpub, overseerr port-map) to handle |
| 2026-07-27 | Network membership (1.5) | ⚠️ | kavita/navidrome/seerr added to web_proxy; verify all |
| 2026-07-27 | Cert renewal | ⚠️ | Script written; needs HOST cron scheduling |
| 2026-07-27 | Circadian lighting engine | ✅ | `light_sync.py` deployed to HA; dual-light, equinox-anchored; look-vs-power separation wired |
| 2026-07-27 | Context tool live HA snapshot | ✅ | End-to-end: snapshot → validate → export → commit on real HA data (21 domains, 2 automations) |
| 2026-07-27 | Prowlarr→*arr sync fix | ✅ | Root cause: stale bridge IPs + missing urlBase + IPv6 dead-end. All 4 apps recreated with --sysctl IPv6 disable; sync URLs corrected |
| 2026-07-27 | *arr Batch 2 recreate | ✅ | sonarr/radarr/lidarr recreated with IPv6 sysctl; indexers retained |
| 2026-07-27 | Readarr indexer category mismatch | ⚠️ | Readarr has 0 book indexers — general indexers don't return book cats. Pending: user needs book indexers |
| 2026-07-27 | Circadian automation entity_id drift | ⚠️ | Live entity_id != YAML `id` due to UI edit. Option A chosen (YAML as truth) but not yet executed (blocked by HA modify approval) |
| 2026-07-27 | HA circadian live test | ⚠️ | Approved but not yet run. Ties into entity_id reconciliation |

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
- **Servarr IPv6 dead-end**: .NET prefers AAAA DNS; Docker bridge has no IPv6 route.
  Fix via `--sysctl net.ipv6.conf.all.disable_ipv6=1` + `default=1` on container
  recreate. Affects ALL *arr apps (each makes direct .NET outbound), not just Prowlarr.
- **Prowlarr `fields[]` must be set, not top-level keys**: `baseUrl` and `apiKey`
  and `prowlarrUrl` must live inside the `fields[]` array for a `PUT /api/v1/applications`
  to persist. Sending them as flat keys is silently ignored.
- **Prowlarr `apiKey` cannot be sent as `********`** — the masked value from a partial
  export **wipes** the real key when PUT. Always read the real key from
  `/config/config.xml` before updating.
- **`urlBase` MUST be in every sync URL**: `baseUrl` should be
  `http://readarr:8787/readarr` NOT `http://readarr:8787`. Missing it causes sync to
  fail (301 redirect loop / 401).
- **Readarr 0 indexers is EXPECTED** for general indexers — they don't return book
  categories. The fix is adding book-specific indexers to Prowlarr, not a config bug.
- **linuxserver *arr images use `catatonit`**: entrypoint is
  `/usr/bin/catatonit -- /entrypoint.sh`. Recreating with bare `--entrypoint /usr/bin/catatonit`
  (no program arg) crash-loops with "missing program name". Omit `--entrypoint` entirely
  when faithfully recreating.
- **Context tool `validate` works**: on its first live run it caught real
  automation entity_id drift that would have caused silent task failure. Run it before
  every task injection.
- **CasaOS re-import vs Caddy routing (Phase 1 DONE)**: re-importing apps via CasaOS UI moves them to
  `bridge` network. Caddy now uses `host.docker.internal:<host_port>` for 16/18 apps (network-agnostic),
  so re-importing those 16 will NOT break ingress. Remaining exceptions still on container-name:
  `vaultwarden` (port 80 not host-published) and `overseerr` (host port published `5555→5555` but app
  listens on 5055 — needs port-map fix before it can be decoupled). After re-import, reconnect
  `web_proxy` so Prowlarr→*arr name resolution keeps working.
- **Circadian automation entity_id reconciliation**: YAML as source of truth (Option A)
  chosen but not executed — needs HA API modify approval (delete storage automation +
  reload). **Pending user go-ahead.**
- **HA circadian live test**: script ready, approved, not yet run. Ties into the
  automation entity_id cleanup (run after reconcile to avoid targeting the wrong entity).

---

## 6. CasaOS App Registration (Task 1.4 — Phase 1 Caddy decouple ✅, Phase 2 UI re-import ⏳)

### Current state
- All 18 apps are **accessible via the domain** (ingress works).
- CasaOS shows the ARR stack + friends as **"legacy"** because their running
  containers lack the `icon` label and aren't registered in CasaOS's app DB
  (they were recreated via `docker run`, bypassing CasaOS registration).
- qbittorrent shows properly because its container has the `icon` label.

### The architecture conflict (largely defused in Phase 1)
- CasaOS installs apps on the **`bridge`** network (default).
- Caddy now reaches 16/18 apps via **`host.docker.internal:<host_port>`** (network-agnostic),
  NOT container name — so re-importing those to `bridge` no longer breaks ingress.
- **Remaining exception (1 app)**: `vaultwarden` (port 80 not host-published) still uses
  container-name routing. `overseerr` is being deleted by the user (its Caddy route blocks removed).
  After re-import of each app, run `docker network connect web_proxy <app>` so Prowlarr→*arr
  name resolution keeps working (Caddy for the 16 host.docker.internal apps does NOT need this).

### Decision (Option B chosen — Phase 1 executed 2026-07-27)
- **Phase 1 ✅**: Caddy rewritten to `host.docker.internal:<host_port>` for 16/18 apps.
  All 18 routes re-verified green after reload. Apps decoupled from `web_proxy` membership
  for ingress purposes.
- **Phase 2 (pending, manual)**: re-import apps via CasaOS UI for proper icons + clickable URLs.
  For each re-imported app, run `docker network connect web_proxy <app>` to preserve
  Prowlarr→*arr name resolution. Host port mapping used (verified):
  - ARR/media: host = container port (radarr 7878, sonarr 8989, lidarr 8686, readarr 8787,
    prowlarr 9696, bazarr 6767, jellyfin 8096, navidrome 4533, kavita 5000, mylar3 8090)
  - seerr: host **5056** (container 5055)
  - hermes: host **9119** (NOT 8642 — 8642 is a dead mapping; app binds 9119)
  - qbittorrent: host **8181** (via gluetun) — already on host.docker.internal
  - homeassistant: host **8123** / esphome: host **6052** / syncthing: host **8384** — already on host.docker.internal
  - **vaultwarden**: port 80 NOT host-published → stays container-name `vaultwarden:80` (cosmetic legacy in CasaOS)
  - **overseerr**: being DELETED by user — excluded from re-import; its Caddy route blocks removed.

### Vaultwarden URLs (user's explicit ask)
- Each app's Vaultwarden entry should use `https://homelab.taild32764.ts.net/<app>`
  (not LAN IP). Requires Vaultwarden API token (in user's Vault) to audit/update.

## 2026-08-02 — Obsidian LiveSync (G2 sync decision → option B)
- **Decision:** LiveSync (CouchDB) chosen for vault sync — resolves parked G2 (was: "Fix Syncthing / LiveSync / Both"). Picked B (LiveSync for Obsidian) over C: Syncthing container still broken (lost mounts).
- **Deployed:** CouchDB 3.5.2 container `couchdb` on web_proxy, data at `/DATA/AppData/obsidian-livesync/data`; DB `obsidian-livesync`; Caddy root URI `https://homelab.taild32764.ts.net:10380/` (path `/livesync/` also routes, but plugin requires root URI for its wizard).
- **Seeded:** via official `self-hosted-livesync-cli` (built from source, image `livesync-cli`, repo `/opt/data/livesync-src`): `mirror /DATA/Documents/Notes` → `sync`. 3,101 docs; decryption round-trip verified; `.livesync/ignore` excludes `.git/` + imports vault .gitignore (PAT, creds stay out).
- **Creds:** `/opt/data/.secrets/couchdb_livesync` (admin pw + E2E passphrase); Setup URI at `.secrets/livesync_setup_uri.txt`.
- **Pending:** Jon installs plugin on desktop+phone with the Setup URI (devices pull existing seed — no reset-remote needed). Vault git backup unchanged (durability net remains).
