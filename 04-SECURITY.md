# Security Architecture

**Status:** Living Document
**Last Updated:** 2026-07-30
**Owner:** Jon Ross

---

## 1. Security Goals

### 1.1 Network Isolation
- **VPN-isolated traffic**: qBittorrent + ARR stack route through Gluetun VPN — their traffic is masked from ISP
- **Clear-network apps**: Media (Jellyfin, Kavita), management (Caddy, Hermes, CasaOS), smart home (HA, ESPHome) bypass VPN — no unnecessary latency
- **No crosstalk**: App-specific Docker networks (vaultwarden_default, flaresolverr_default) isolate dependencies
- **Tailscale-only external access**: No public ports; all ingress through Tailscale tunnel + device auth

### 1.2 Authentication Layers
| Layer | Scope | Mechanism | Strength | Weakness |
|-------|-------|-----------|----------|----------|
| **Tailscale** | Network-level | Device auth, ACLs, cryptographic identity | Strong | All-or-nothing per-app |
| **App-level** | Per-application | Forms, API keys, SSO | Granular | Multiple credentials |
| **LAN** | Local network | Network trust (no auth) | Weak | Relies on physical/network security |

### 1.3 Data Security
- **Secrets**: Currently in compose files/env vars (improvement path: Docker secrets or Vaultwarden-managed)
- **Configs**: Git-tracked (homelab-architecture repo)
- **Backups**: Strategy TBD (Phase 3 goal)
- **TLS**: Let's Encrypt cert via Tailscale, verified key matching

### 1.4 Traffic Flow (Goal)
```
External user
  │ https://homelab.taild32764.ts.net/<app>
  ▼
Tailscale tunnel (encrypted, device-authenticated)
  │
  ▼
Caddy TLS termination (Tailscale-issued cert)
  │
  ├──→ App on web_proxy/host.docker.internal
  │     (direct Internet — Jellyfin, HA, Hermes, etc.)
  │
  └──→ Gluetun VPN tunnel
        │
        └──→ qBittorrent + *arr apps
              (anonymized traffic)
```

---

## 2. Current Security Posture

### 2.1 ✅ What's Working

| Control | Status | Details |
|---------|--------|---------|
| **External access via Tailscale only** | ✅ | No open ports to public internet |
| **Caddy TLS termination** | ✅ | Tailscale-issued Let's Encrypt cert, verified key pair |
| **Tailscale serve removed** | ✅ | No reliance on broken iptables-based proxy |
| **X-Forwarded-Proto forced** | ✅ | Apps emit `https://` redirects correctly |
| **Gluetun VPN for qBittorrent** | ✅ | qBittorrent traffic fully tunneled |
| **host.docker.internal resolving** | ✅ | `--add-host` configured on Caddy |
| **web_proxy isolation** | ✅ | Web apps on dedicated bridge network |
| **Caddy network-agnostic** | ✅ | 16/18 apps use host.docker.internal — survive network changes |
| **Container-healthy stack** | ✅ | All 29 containers up (verified live) |

### 2.2 ✅ Fixed 2026-07-30

| Control | Status | Details |
|---------|--------|---------|
| **CasaOS app icons** | ✅ | All 18 web apps have icons via selfhst CDN or CasaOS AppStore |
| **CasaOS "Open" URLs** | ✅ | All apps point to `https://homelab.taild32764.ts.net/<app>`. Fixed: readarr (path in hostname), dockhand (LAN IP), vaultwarden (empty hostname, http), seerr (/overseerr), home-assistant (/home-assistant) |
| **Caddy routes added** | ✅ | dockhand (:3003), music-assistant (:8095) |
| **Indentation consistency** | ✅ | Each file matches its own style (2 or 4-space where appropriate) |
| **Backend services** | ✅ | Icons kept; routing metadata stripped from gluetun, flaresolverr, obsidian-livesync, matter-server, tailscale |
| **Duplicate/temp containers removed** | ✅ | 3 old duplicates (syncthing-old, readarr_old, vaultwarden_old) + 5 temp containers removed |
| **Legacy apps imported** | ✅ | hermes, jellyfin, lidarr, prowlarr, radarr, sonarr added to app_order.json (22 total managed) |
| **CasaOS app metadata backup** | ✅ | app_order.json.bak.2026-07-30 |

### 2.3 ❌ Missing / Not Started

| Control | Status | Notes |
|---------|--------|-------|
| **Pi-hole + Unbound** | ❌ | Phase 2 goal, never started |
| **Router DNS integration** | ❌ | Requires DHCP DNS change → Pi-hole |
| **Monitoring & alerting** | ❌ | Phase 3 goal, never started |
| **Automated backups** | ❌ | Phase 3 goal, never started |
| **Recovery runbook** | ❌ | Phase 3 goal |
| **Certificate auto-renewal** | ❌ | Script exists at `/opt/data/scripts/renew-tailscale-cert.sh`, needs host cron |

---

## 3. Gap Analysis: Goals vs Current

### 3.1 DNS Stack

| Goal | Current | Delta | Effort |
|------|---------|-------|--------|
| Pi-hole on :53 for ad-blocking | **Not deployed** | ❌ Missing | Medium |
| Unbound as recursive resolver | **Not deployed** | ❌ Missing | Medium |
| All LAN devices → Pi-hole | **Router DNS still default** | ❌ Not configured | Low |
| Custom domain support | **Not applicable yet** | ❌ Missing | Low |

### 3.2 Ingress

| Goal | Current | Delta | Effort |
|------|---------|-------|--------|
| Caddy reverse proxy | ✅ Running on :443 + :8080 | ✅ Match | — |
| TLS via Tailscale cert | ✅ Verified working | ✅ Match | — |
| Path-based routing `/<app>` | ✅ 18 routes configured | ✅ Match | — |
| Tailscale device auth | ✅ Tunnel + ACLs | ✅ Match | — |
| No Flaresolver needed | ⚠️ **Flaresolverr deployed and running** | ⚠️ Confirm needed | Low |

### 3.3 VPN Isolation

| Goal | Current | Delta | Effort |
|------|---------|-------|--------|
| qBittorrent behind gluetun | ✅ Working | ✅ Match | — |
| **NZBGet behind gluetun** | ❌ **NZBGet on web_proxy, direct Internet** | ❌ Gap | Medium |
| ARR stack intentionally direct | ✅ Confirmed user intent | ✅ Chosen design | — |
| Other apps bypass VPN | ✅ Working (Jellyfin, HA, Caddy, etc.) | ✅ Match | — |

Moving NZBGet behind gluetun requires:
- Recreating NZBGet with `network_mode: container:gluetun` (or joining qbittorrent_default network)
- Publishing NZBGet's web UI port through gluetun (or keeping it LAN-only)
- NZBGet needs to reach qBittorrent's download location — path mapping may need adjustment

### 3.4 Firewall & Exposure

| Goal | Current | Delta |
|------|---------|-------|
| Only 80 (CasaOS), 8080 (Caddy), 443 (Caddy TLS) exposed on LAN | ✅ Confirmed via Docker port maps | ✅ Match |
| No direct access to app ports from LAN | ✅ No container ports published to host | ✅ Match |
| All external through Tailscale | ✅ No public ports | ✅ Match |

### 3.5 Authentication

| Goal | Current | Delta |
|------|---------|-------|
| Tailscale as network-level auth | ✅ Working | ✅ Match |
| App-level auth for individual apps | ✅ Each app auths independently | ✅ Match |
| Unified auth layer (e.g., Authelia) | ❌ Not implemented | Not a stated goal |

---

## 4. Key Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| NZBGet behind gluetun pending | **Medium** — Usenet traffic visible to ISP | Move NZBGet to gluetun netns (medium effort) |
| Flaresolverr kept for Prowlarr | **Low** — Needed for CAPTCHA indexers | ✅ User confirmed keep
| Certificate expires (Oct 25 2026) | **Medium** — TLS breaks, apps unreachable externally | Schedule cert renewal cron on host |
| DNS poisoning / no ad blocking | **Low-Medium** — No network-wide protection | Pi-hole + Unbound deployment (Phase 2) |
| Vaultwarden port 80 unpublished | **Low** — Only affects CasaOS re-import; ingress works via container-name | Publish port 80 when ready to decouple fully |
