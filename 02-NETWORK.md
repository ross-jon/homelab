# Network Architecture

**Status:** Living Document
**Last Updated:** 2026-07-30
**Owner:** Jon Ross

---

## 1. Design Goals (from Jon, 2026-07-30)

### 1.1 DNS Stack (Core)
```
All Devices → Pi-hole (:53) → Unbound (:53) → Internet (recursive)
```
- **Pi-hole** on `0.0.0.0:53` — ad-blocking, DNS filtering, single entry point
- **Unbound** as upstream — pure recursive resolver, no external DNS dependency
- Every LAN device queries Pi-hole; Pi-hole queries Unbound
- Result: clean ad-blocking + privacy (no upstream logs) + custom domain support

### 1.2 Ingress (Proxy + TLS)
```
Browser → Tailscale tunnel → Caddy (:443 TLS) → App containers
                                  ↓
                        Also :8080 (LAN, plain HTTP)
```
- **Caddy** terminates TLS (via Tailscale-issued Let's Encrypt cert)
- Path-based routing: `https://homelab.taild32764.ts.net/<app>`
- Tailscale provides device auth + encrypted tunnel — no longer does HTTP proxy
- No Flaresolver needed for base ARR scraping (note: currently deployed)

### 1.3 VPN Isolation (Gluetun)
```
qBittorrent + NZBGet → Gluetun (VPN tunnel) → Internet
ARR stack + all other apps → direct Internet access
```
- **Gluetun** runs Mullvad/ProtonVPN (configured)
- **qBittorrent** binds to Gluetun's network namespace (`network_mode: container:gluetun`)
- **NZBGet** should also route through gluetun (current: direct Internet — pending migration)
- **ARR stack** intentionally has direct Internet access (no VPN overhead needed for indexer queries)
- Everything else (Jellyfin, Kavita, Caddy, Hermes, HA, etc.) bypasses VPN

### 1.4 External Access (Tailscale)
- Single domain: `homelab.taild32764.ts.net`
- Path-based routing to all apps via `/<app>`
- Device auth via Tailscale ACLs
- No public exposure — Tailscale only

---

## 2. Current Live Network Topology

### 2.1 Docker Networks

| Network | Driver | Members | Purpose |
|---------|--------|---------|---------|
| `web_proxy` | bridge | caddy, bazarr, flaresolverr, jellyfin, lidarr, mylar3, navidrome, nzbget, overseerr, prowlarr, radarr, readarr, sonarr, big-bear-kavita, big-bear-seerr, big-bear-syncthing, vaultwarden | Web-accessible apps reachable by Caddy (container-name DNS) |
| `qbittorrent_default` | bridge | qbittorrent (via gluetun netns) | VPN-isolated apps |
| Host | n/a | tailscale, homeassistant, esphome | Apps needing host network access |
| Other bridge nets | bridge | vaultwarden_default, flaresolverr_default, etc. | App-specific dependency isolation |

### 2.2 Container Network Modes

```
┌─────────────────────────────────────────────────────────┐
│                     HOST NETWORK                         │
│  ┌──────────┐  ┌──────────────┐  ┌────────┐            │
│  │ tailscale│  │ homeassistant│  │ esphome│            │
│  └──────────┘  └──────────────┘  └────────┘            │
├─────────────────────────────────────────────────────────┤
│                   web_proxy (172.23.0.0/16)              │
│  ┌─────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐      │
│  │caddy│ │radarr│ │sonarr│ │lidarr│ │prowlarr  │      │
│  │.10  │ │.7    │ │.15   │ │.14   │ │.5        │      │
│  └─────┘ └──────┘ └──────┘ └──────┘ └──────────┘      │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │
│  │readarr │ │bazarr  │ │jellyfin│ │kavita  │           │
│  │.3      │ │.2      │ │.8      │ │.11     │           │
│  └────────┘ └────────┘ └────────┘ └────────┘           │
│  ┌──────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │navidrome │ │mylar3  │ │seerr   │ │syncth. │        │
│  │.9        │ │.13     │ │.19     │ │.18     │        │
│  └──────────┘ └────────┘ └────────┘ └────────┘        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│  │overseerr │ │vaultwd.  │ │nzbget    │               │
│  │.4        │ │.6        │ │.17       │               │
│  └──────────┘ └──────────┘ └──────────┘               │
├─────────────────────────────────────────────────────────┤
│              qbittorrent_default (gluetun netns)         │
│  ┌──────────┐ ┌──────────────┐                          │
│  │ gluetun  │ │ qbittorrent  │  (shares netns)          │
│  └──────────┘ └──────────────┘                          │
├─────────────────────────────────────────────────────────┤
│              Other bridge networks                       │
│  vaultwarden_default, flaresolverr_default, hermes_default│
└─────────────────────────────────────────────────────────┘
```

### 2.3 Current Routing (Caddy)

**TLS Entry (`:443`):** `homelab.taild32764.ts.net` — Tailscale-issued cert
**LAN Entry (`:8080`):** plain HTTP, for CasaOS dashboard

Both `import (routes)` — the same snippet handles all apps:

| App | Caddy Target | Group | Behind VPN? |
|-----|-------------|-------|-------------|
| vaultwarden | `vaultwarden:80` (container name) | A (no strip) | ❌ |
| jellyfin | `host.docker.internal:8096` | B (strip) | ❌ |
| kavita | `host.docker.internal:5000` | A (no strip) | ❌ |
| navidrome | `host.docker.internal:4533` | A (no strip) | ❌ |
| radarr | `host.docker.internal:7878` | A (no strip) | ❌ (intentional) |
| sonarr | `host.docker.internal:8989` | A (no strip) | ❌ (intentional) |
| lidarr | `host.docker.internal:8686` | A (no strip) | ❌ (intentional) |
| readarr | `host.docker.internal:8787` | A (no strip) | ❌ (intentional) |
| prowlarr | `host.docker.internal:9696` | A (no strip) | ❌ (intentional) |
| bazarr | `host.docker.internal:6767` | A (no strip) | ❌ (intentional) |
| mylar3 | `host.docker.internal:8090` | B (strip) | ❌ |
| overseerr | `host.docker.internal:5055` | B (strip) | ❌ |
| seerr | `host.docker.internal:5056` | B (strip) | ❌ |
| qbittorrent | `host.docker.internal:8181` | B (strip) | ✅ (gluetun) |
| **nzbget** | `host.docker.internal:6789` | B (strip) | ❌ **(should be gluetun)** |
| homeassistant | `host.docker.internal:8123` | B (strip) | ❌ |
| esphome | `host.docker.internal:6052` | B (strip) | ❌ |
| syncthing | `host.docker.internal:8384` | B (strip) | ❌ |
| hermes | `host.docker.internal:9119` | B (strip) | ❌ |
| **dockhand** | `host.docker.internal:3003` | B (strip) | ❌ (new route) |
| **music-assistant** | `host.docker.internal:8095` | B (strip) | ❌ (new route) |
| CasaOS root | `host.docker.internal:80` | catch-all | ❌ |

### 2.4 Host.docker.internal Resolution
- `--add-host host.docker.internal:host-gateway` on Caddy → resolves to `172.17.0.1` ✅
- Caddy uses this for 16/18 apps (network-agnostic routing)
- Exception: **vaultwarden** (port 80 not host-published) — uses container-name `vaultwarden:80`

---

## 3. Access Patterns

### External (via Tailscale)
```
https://homelab.taild32764.ts.net/<app>/
    → Tailscale tunnel (device auth)
    → Host port 443 (Caddy TLS)
    → Caddy path-based routing
    → App container
```

### LAN (direct)
```
http://192.168.86.29:8080/<app>/
    → Caddy plain HTTP
    → App container
```

### CasaOS (one-click Open)
```
Constructed from app metadata:
    scheme://hostname/index
    → https://homelab.taild32764.ts.net/<app>
```
