# Detailed Architecture

**Status:** Draft  
**Created:** 2026-01-XX  
**Last Updated:** 2026-01-XX

---

## System Overview

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        External Access                       │
│                  (Tailscale Network Only)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ HTTPS (TLS terminated by Tailscale)
                     │ https://homelab.taild32764.ts.net/<app>
                     │
┌────────────────────▼────────────────────────────────────────┐
│                      Tailscale Serve                         │
│  • TLS termination                                          │
│  • Authentication (Tailscale ACLs)                          │
│  • Proxies to: http://localhost:8080                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ HTTP (plain, internal only)
                     │ http://localhost:8080
                     │
┌────────────────────▼────────────────────────────────────────┐
│                        Caddy (Port 8080)                     │
│  • Path-based routing                                       │
│  • Reverse proxy to containers                              │
│  • No TLS (handled by Tailscale)                            │
│  • Config: /DATA/AppData/caddy/Caddyfile                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Docker network: web_proxy
                     │
        ┌────────────┼────────────┬────────────┬────────────┐
        │            │            │            │            │
        ▼            ▼            ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Radarr  │  │ Sonarr  │  │Jellyfin │  │Kavita   │  │Vaultward│
   │ :7878   │  │ :8989   │  │ :8096   │  │ :5000   │  │  :80    │
   └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘
        │            │            │            │            │
        └────────────┴────────────┴────────────┴────────────┘
                     │
                     │ All on web_proxy network
                     │ Container-to-container communication
                     │
┌────────────────────▼────────────────────────────────────────┐
│                      CasaOS (Port 80)                        │
│  • Dashboard UI                                             │
│  • Container management                                     │
│  • App metadata (icons, URLs)                               │
│  • Catch-all route: / → CasaOS                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. Tailscale

**Role:** Secure external access, TLS termination, authentication

**Configuration:**
```bash
# Current setup
tailscale serve --bg --set-path=/ http://localhost:8080

# Result
https://homelab.taild32764.ts.net → http://localhost:8080
```

**Key Points:**
- Tailscale handles all TLS (Caddy sees plain HTTP)
- Authentication is at network level (Tailscale ACLs)
- Apps see requests as "local" (no additional auth needed)
- Only accessible from tailnet (no public exposure)

**Network Mode:** Host  
**Why:** Tailscale needs host network for MagicDNS and serve functionality

---

### 2. Caddy

**Role:** Path-based reverse proxy

**Configuration File:** `/DATA/AppData/caddy/Caddyfile`

**Key Design Decisions:**

#### Decision 1: No TLS in Caddy
- **Why:** Tailscale handles TLS
- **Benefit:** Simpler config, no certificate management
- **Trade-off:** Can't use Caddy for external access without Tailscale

#### Decision 2: Path-Based Routing
- **Why:** Single domain, multiple apps
- **Pattern:** `https://homelab.taild32764.ts.net/<app>`
- **Alternative:** Subdomains (more complex DNS)

#### Decision 3: Two Routing Groups

**Group A: App Knows Base Path**
```caddyfile
handle /radarr/* {
    reverse_proxy radarr:7878
}
handle /radarr {
    redir /radarr/ 301
}
```
- App configured with `base_url: /radarr`
- Caddy does NOT strip prefix
- App sees: `/radarr/api/v1/...`

**Apps in Group A:**
- radarr, sonarr, lidarr, readarr, prowlarr, bazarr
- navidrome

**Group B: Caddy Strips Prefix**
```caddyfile
handle_path /jellyfin/* {
    reverse_proxy jellyfin:8096
}
```
- App served at root (no base path config)
- Caddy strips `/jellyfin` prefix
- App sees: `/api/v1/...` (not `/jellyfin/api/v1/...`)

**Apps in Group B:**
- jellyfin, kavita, vaultwarden, qbittorrent
- overseerr, mylar3, homeassistant, esphome, syncthing, hermes

**Why Two Groups?**
- Some apps support base path configuration (Group A)
- Others don't (Group B)
- Group A is simpler (no path rewriting)
- Group B requires `handle_path` directive

---

### 3. Docker Networks

#### `web_proxy` Network

**Purpose:** Reverse proxy routing  
**Type:** Bridge  
**Subnet:** 172.23.0.0/16 (auto-assigned)  
**Driver:** bridge

**Members:**
```
caddy                    172.23.0.10
radarr                   172.23.0.7
sonarr                   172.23.0.15
lidarr                   172.23.0.14
readarr                  172.23.0.3
prowlarr                 172.23.0.5
bazarr                   172.23.0.2
jellyfin                 172.23.0.8
navidrome                172.23.0.9
big-bear-kavita          172.23.0.11
mylar3                   172.23.0.13
overseerr                172.23.0.4
qbittorrent              (via gluetun)
vaultwarden              172.23.0.6
hermes                   172.23.0.12
big-bear-syncthing       172.23.0.18
big-bear-home-assistant  (not on web_proxy yet)
esphome                  (not on web_proxy yet)
nzbget                   172.23.0.17
```

**Access Pattern:**
```
Tailscale → Caddy (web_proxy:8080) → App (web_proxy:<port>)
```

**Why Separate Network?**
- Isolation from other Docker networks
- Clear membership (only web-accessible apps)
- Easy to audit and manage

#### App-Specific Networks

**Examples:**
- `qbittorrent_default`: qbittorrent + gluetun (VPN tunnel)
- `big-bear-syncthing_default`: syncthing cluster
- `vaultwarden_default`: vaultwarden + db (future)

**Why Separate?**
- Isolate dependencies
- Prevent cross-talk
- Easier troubleshooting

#### Host Network

**Members:**
- tailscale (required for MagicDNS)

**Why Host Network?**
- Tailscale needs direct host access
- Can't work through Docker bridge

---

### 4. CasaOS

**Role:** Dashboard, container management

**Port:** 80 (HTTP)  
**Path:** `/` (catch-all in Caddy)

**Integration Points:**

#### App Metadata
Each app has metadata in `/var/lib/casaos/apps/<app>/docker-compose.yml`:
```yaml
x-casaos:
  hostname: homelab.taild32764.ts.net
  scheme: https
  index: /radarr
  icon: https://cdn.jsdelivr.net/gh/.../icon.svg
  title:
    en_US: Radarr
```

**How It Works:**
1. CasaOS reads metadata from compose files
2. Dashboard shows app cards with icons
3. "Open" button constructs URL: `{scheme}://{hostname}{index}`
4. Example: `https://homelab.taild32764.ts.net/radarr`

**Current Issues:**
- Some apps missing metadata
- Some apps show as "legacy" (need reimport)
- Inconsistent icon URLs

**Fix Strategy:**
1. Audit all apps for metadata
2. Add missing fields
3. Test "Open" button routing
4. Document metadata schema

---

### 5. Pi-hole + Unbound (Planned)

**Role:** Network-wide DNS filtering

**Architecture:**
```
Router (DNS: 192.168.86.29)
    ↓
Pi-hole (Port 53)
    ↓
Unbound (Port 5335)
    ↓
Internet DNS
```

**Pi-hole Configuration:**
- **Port:** 53 (DNS)
- **Web UI:** Port 80 (or subpath via Caddy)
- **Upstream:** Unbound (127.0.0.1#5335)
- **Blocklists:** Default + custom lists

**Unbound Configuration:**
- **Port:** 5335 (internal only)
- **Role:** Recursive DNS resolver
- **Forwarding:** None (pure recursive)
- **Cache:** Aggressive caching for performance

**Integration:**
1. Router DHCP → DNS: 192.168.86.29 (Pi-hole)
2. Pi-hole → Unbound (127.0.0.1#5335)
3. Unbound → Internet (recursive resolution)

**Benefits:**
- Network-wide ad blocking (no client config)
- Privacy (recursive DNS, no upstream logs)
- Performance (aggressive caching)

**Access:**
- **DNS:** Port 53 (all devices)
- **Web UI:** `https://homelab.taild32764.ts.net/pihole` (via Caddy)

---

## Traffic Flow Examples

### Example 1: Accessing Radarr

**User Action:** Open `https://homelab.taild32764.ts.net/radarr`

**Flow:**
```
1. Browser → Tailscale (DNS resolution)
   - Resolves homelab.taild32764.ts.net → 100.x.x.x (Tailscale IP)

2. Browser → Tailscale (HTTPS)
   - TLS handshake
   - Tailscale authenticates device

3. Tailscale → Caddy (HTTP)
   - Proxies to http://localhost:8080
   - Request: GET /radarr

4. Caddy → Radarr (Docker network)
   - Matches: handle /radarr/*
   - Proxies to radarr:7878
   - Request: GET /radarr (no stripping)

5. Radarr → Caddy → Tailscale → Browser
   - Response: HTML/JS/CSS
   - All assets use /radarr/ base path
```

**Why It Works:**
- Radarr configured with `base_url: /radarr`
- All asset paths include `/radarr/`
- No path rewriting needed

### Example 2: Accessing Jellyfin

**User Action:** Open `https://homelab.taild32764.ts.net/jellyfin`

**Flow:**
```
1-3. (Same as Radarr)

4. Caddy → Jellyfin (Docker network)
   - Matches: handle_path /jellyfin/*
   - Strips /jellyfin prefix
   - Proxies to jellyfin:8096
   - Request: GET / (not /jellyfin)

5. Jellyfin → Caddy → Tailscale → Browser
   - Response: HTML/JS/CSS
   - Asset paths: /jellyfin/... (rewritten by Caddy)
```

**Why It Works:**
- Jellyfin doesn't support base path
- Caddy strips `/jellyfin` before proxying
- Caddy rewrites response paths to include `/jellyfin`

### Example 3: DNS Query (Future)

**User Action:** Device requests `example.com`

**Flow:**
```
1. Device → Router (DNS query)
   - Router DHCP DNS: 192.168.86.29

2. Router → Pi-hole (Port 53)
   - Pi-hole checks blocklists
   - If blocked: return 0.0.0.0
   - If not blocked: forward to upstream

3. Pi-hole → Unbound (Port 5335)
   - Unbound checks cache
   - If cached: return result
   - If not cached: recursive resolution

4. Unbound → Internet DNS
   - Query root servers
   - Query TLD servers
   - Query authoritative servers

5. Unbound → Pi-hole → Router → Device
   - Return IP address
   - Pi-hole caches result
```

**Why It Works:**
- Pi-hole blocks ads at DNS level
- Unbound provides privacy (no upstream logs)
- All devices benefit (no client config)

---

## Security Considerations

### Authentication Layers

#### Layer 1: Tailscale (Network Level)
- **Scope:** All external access
- **Mechanism:** Tailscale ACLs, device approval
- **Strength:** Strong (cryptographic identity)
- **Weakness:** All-or-nothing (can't restrict per-app)

#### Layer 2: App-Level (Application Level)
- **Scope:** Individual apps
- **Mechanism:** App-specific (forms, API keys)
- **Strength:** Granular control
- **Weakness:** Multiple credentials to manage

#### Layer 3: Local Network (Trust Level)
- **Scope:** Direct LAN access
- **Mechanism:** Network trust (no auth)
- **Strength:** Convenience
- **Weakness:** Relies on network security

### Network Security

#### Docker Network Isolation
- **web_proxy:** Only web-accessible apps
- **App networks:** Isolated dependencies
- **Host network:** Limited to Tailscale

#### Firewall Rules
- **Exposed ports:** 80, 8080, 443
- **Blocked:** Direct app port access from LAN
- **Rationale:** All access through reverse proxy

### Data Security

#### Secrets Management
- **Current:** Hardcoded in compose files
- **Future:** Docker secrets or Vault
- **Priority:** Low (internal network only)

#### Backup Strategy
- **Configs:** Git repository
- **Data:** Regular backups (TBD)
- **Recovery:** Documented procedures

---

## Performance Considerations

### Caddy Performance
- **Connection pooling:** Enabled by default
- **Keep-alive:** Enabled
- **Compression:** Enabled (gzip)
- **Caching:** No caching (reverse proxy)

### DNS Performance (Pi-hole + Unbound)
- **Pi-hole cache:** 24h TTL
- **Unbound cache:** Aggressive (7 days)
- **Expected latency:** <10ms for cached queries
- **Expected hit rate:** >90%

### Network Performance
- **Docker bridge:** Minimal overhead
- **web_proxy network:** Isolated, no congestion
- **Tailscale:** Encrypted, slight overhead (~5%)

---

## Troubleshooting Guide

### Common Issues

#### Issue: 502 Bad Gateway
**Symptoms:** Browser shows 502 error  
**Cause:** Caddy can't reach container  
**Diagnosis:**
```bash
# Check if container is running
docker ps | grep <app>

# Check if container is on web_proxy
docker inspect <app> | grep -A 5 "Networks"

# Test from Caddy
docker exec caddy wget -qO- http://<app>:<port>/
```

**Fix:**
1. Ensure container is running
2. Add container to web_proxy network
3. Verify port is correct

#### Issue: Blank White Page
**Symptoms:** Page loads but no content  
**Cause:** Assets blocked (auth, CORS, etc.)  
**Diagnosis:**
```bash
# Check browser console (F12)
# Look for 401/403 errors

# Test asset directly
curl -I http://localhost:8080/<app>/asset.js
```

**Fix:**
1. Check app authentication settings
2. Verify CORS headers
3. Check Caddy logs

#### Issue: Redirect Loop
**Symptoms:** Browser shows "too many redirects"  
**Cause:** Incorrect redirect configuration  
**Diagnosis:**
```bash
# Check redirect chain
curl -v http://localhost:8080/<app> 2>&1 | grep Location
```

**Fix:**
1. Check Caddy redirect rules
2. Verify app base path config
3. Ensure no circular redirects

---

## Future Enhancements

### Phase 5: Monitoring & Alerts
- **Tools:** Grafana + Prometheus
- **Metrics:** Container health, DNS performance
- **Alerts:** Downtime, high resource usage

### Phase 6: Automated Backups
- **Tools:** Restic or BorgBackup
- **Schedule:** Daily incremental, weekly full
- **Storage:** Local + offsite (TBD)

### Phase 7: High Availability
- **Tools:** Docker Swarm or Kubernetes
- **Benefit:** Zero-downtime deployments
- **Complexity:** Significant increase

---

## References

- [Caddy Documentation](https://caddyserver.com/docs/)
- [Tailscale Serve](https://tailscale.com/kb/1242/tailscale-serve)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [Unbound Documentation](https://nlnetlabs.nl/documentation/unbound/)

---

**Document Version:** 1.0  
**Next Review:** After Phase 1 completion
