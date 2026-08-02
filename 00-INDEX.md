# Homelab Architecture & Implementation Plan

**Status:** Planning Phase  
**Created:** 2026-01-XX  
**Owner:** Jon Ross  
**Last Updated:** 2026-07-30

---

## Executive Summary

### Goals
1. **Unified Access:** All apps accessible via `https://homelab.taild32764.ts.net/<app>`
2. **Network-Wide Ad Blocking:** Pi-hole + Unbound for DNS-level ad blocking
3. **One-Click Dashboard:** CasaOS dashboard routes through Tailscale with seamless auth
4. **Simple & Clean:** Minimal complexity, clear separation of concerns

### Current State
- ✅ 25+ Docker containers running (ARR stack, media apps, utilities)
- ✅ Tailscale configured with `homelab.taild32764.ts.net` domain
- ✅ Caddy reverse proxy on port 8080
- ✅ `web_proxy` Docker network exists
- ⚠️ Caddy configuration broken (needs rebuild)
- ⚠️ CasaOS app metadata inconsistent
- ❌ No network-wide DNS filtering
- ❌ No unified authentication layer

### Target State
- ✅ Clean, documented reverse proxy setup
- ✅ All apps accessible via Tailscale domain
- ✅ Pi-hole + Unbound for ad blocking
- ✅ CasaOS dashboard with proper routing
- ✅ Comprehensive documentation in Obsidian/GitHub
- ✅ Linear integration for task tracking

---

## Architecture Overview

### High-Level Flow
```
User Browser
    ↓
Tailscale (TLS termination, auth)
    ↓
Caddy (reverse proxy, path routing)
    ↓
Docker Containers (apps on web_proxy network)
```

### Key Components

#### 1. Tailscale
- **Role:** External access, TLS termination, authentication
- **Domain:** `homelab.taild32764.ts.net`
- **Config:** `tailscale serve` proxies to `http://localhost:8080`
- **Auth:** Tailscale handles auth; apps see requests as "local"

#### 2. Caddy
- **Role:** Path-based reverse proxy
- **Port:** 8080 (HTTP only, TLS handled by Tailscale)
- **Config:** `/DATA/AppData/caddy/Caddyfile`
- **Routing:** Path-based (`/radarr`, `/sonarr`, etc.)

#### 3. Docker Networks
- **`web_proxy`:** All web-accessible apps
- **Container networks:** App-specific (e.g., `qbittorrent_default`)
- **Host network:** Tailscale, some utilities

#### 4. CasaOS
- **Role:** Dashboard, container management
- **Port:** 80 (root path `/`)
- **Integration:** Apps show as cards with icons and "Open" buttons

#### 5. Pi-hole + Unbound (NEW)
- **Role:** Network-wide DNS filtering
- **Pi-hole:** Ad blocking, DNS sinkhole
- **Unbound:** Recursive DNS resolver (upstream for Pi-hole)
- **Integration:** Router DNS → Pi-hole → Unbound → Internet

---

## Network Topology

### Docker Networks

#### `web_proxy` (Bridge Network)
**Purpose:** Reverse proxy routing for all web apps  
**Subnet:** 172.23.0.0/16 (auto-assigned)  
**Members:**
- caddy (172.23.0.10)
- radarr, sonarr, lidarr, readarr, prowlarr, bazarr
- jellyfin, navidrome, kavita, mylar3, overseerr
- qbittorrent, vaultwarden, hermes, syncthing
- homeassistant, esphome, nzbget

**Access Pattern:**
```
Tailscale → Caddy (web_proxy:8080) → App (web_proxy:<port>)
```

#### App-Specific Networks
**Purpose:** Isolation for apps with dependencies  
**Examples:**
- `qbittorrent_default`: qbittorrent + gluetun (VPN)
- `big-bear-syncthing_default`: syncthing cluster
- `vaultwarden_default`: vaultwarden + db (if added)

#### Host Network
**Purpose:** Direct host access  
**Members:**
- tailscale (needs host network for Tailscale magic)
- Some utilities (if needed)

---

## Routing Rules

### Caddy Configuration

#### Path-Based Routing
```caddyfile
:8080 {
    # ARR Stack (Group A: app knows its base path)
    handle /radarr/* {
        reverse_proxy radarr:7878
    }
    handle /radarr {
        redir /radarr/ 301
    }
    
    handle /sonarr/* {
        reverse_proxy sonarr:8989
    }
    handle /sonarr {
        redir /sonarr/ 301
    }
    
    # ... (similar for lidarr, readarr, prowlarr, bazarr)
    
    # Media Apps (Group B: Caddy strips prefix)
    handle_path /jellyfin/* {
        reverse_proxy jellyfin:8096
    }
    
    handle_path /kavita/* {
        reverse_proxy big-bear-kavita:5000
    }
    
    # Management Apps
    handle /vaultwarden/* {
        reverse_proxy vaultwarden:80
    }
    
    handle /qbittorrent/* {
        reverse_proxy qbittorrent:8080
    }
    
    # CasaOS (catch-all)
    handle {
        reverse_proxy host.docker.internal:80
    }
}
```

#### Routing Groups

**Group A: App Knows Base Path**
- Apps configured with `base_url` or `UrlBase`
- Caddy does NOT strip prefix
- Examples: radarr, sonarr, lidarr, readarr, prowlarr, bazarr, navidrome

**Group B: Caddy Strips Prefix**
- Apps served at root, no base path config
- Caddy uses `handle_path` to strip prefix
- Examples: jellyfin, kavita, vaultwarden, qbittorrent

---

## Security Model

### Authentication Layers

#### 1. Tailscale Authentication
- **Scope:** All external access
- **Mechanism:** Tailscale ACLs, device approval
- **Scope:** Network-level, not app-level

#### 2. App-Level Authentication
- **Scope:** Individual apps
- **Mechanism:** App-specific (forms, API keys, etc.)
- **Note:** Tailscale auth passes through; apps see "local" requests

#### 3. Local Network Access
- **Scope:** Direct LAN access (192.168.86.29)
- **Mechanism:** No auth required (trusted network)
- **Use Case:** Local devices, troubleshooting

### Network Security

#### Docker Network Isolation
- Apps on `web_proxy` can talk to each other
- App-specific networks isolate dependencies
- Host network limited to Tailscale

#### Firewall Rules
- Only expose: 80 (CasaOS), 8080 (Caddy), 443 (Tailscale)
- Block direct access to app ports from LAN
- All external access goes through Tailscale

---

## Implementation Plan

### Phase 1: Foundation (Week 1)
**Goal:** Get core infrastructure working

#### Tasks
1. **Restore Caddy Configuration**
   - [ ] Backup current state
   - [ ] Rebuild Caddyfile from scratch
   - [ ] Test each route individually
   - [ ] Document routing decisions

2. **Verify Tailscale Integration**
   - [ ] Confirm `tailscale serve` config
   - [ ] Test HTTPS access to all apps
   - [ ] Verify auth passthrough
   - [ ] Document Tailscale setup

3. **Fix CasaOS Metadata**
   - [ ] Audit all app metadata
   - [ ] Fix icons and URLs
   - [ ] Test "Open" button routing
   - [ ] Document metadata schema

**Deliverables:**
- Working reverse proxy
- All apps accessible via Tailscale
- CasaOS dashboard functional

### Phase 2: DNS & Ad Blocking (Week 2)
**Goal:** Implement network-wide ad blocking

#### Tasks
1. **Deploy Pi-hole**
   - [ ] Create Pi-hole container
   - [ ] Configure web interface
   - [ ] Set up blocklists
   - [ ] Test ad blocking

2. **Deploy Unbound**
   - [ ] Create Unbound container
   - [ ] Configure as recursive resolver
   - [ ] Integrate with Pi-hole
   - [ ] Test DNS resolution

3. **Network Integration**
   - [ ] Update router DNS settings
   - [ ] Configure DHCP DNS option
   - [ ] Test all devices
   - [ ] Monitor performance

**Deliverables:**
- Network-wide ad blocking
- Recursive DNS resolution
- Documentation for router config

### Phase 3: Optimization (Week 3)
**Goal:** Polish and optimize

#### Tasks
1. **Performance Tuning**
   - [ ] Optimize Caddy config
   - [ ] Tune Pi-hole settings
   - [ ] Monitor resource usage
   - [ ] Document baselines

2. **Monitoring & Alerts**
   - [ ] Set up health checks
   - [ ] Configure alerts
   - [ ] Create dashboards
   - [ ] Document monitoring

3. **Backup & Recovery**
   - [ ] Document backup strategy
   - [ ] Test restore procedures
   - [ ] Create recovery runbook
   - [ ] Schedule regular backups

**Deliverables:**
- Optimized performance
- Monitoring dashboards
- Backup/recovery procedures

### Phase 4: Documentation & Handoff (Week 4)
**Goal:** Complete documentation and knowledge transfer

#### Tasks
1. **Architecture Documentation**
   - [ ] Finalize architecture docs
   - [ ] Create network diagrams
   - [ ] Document all decisions
   - [ ] Publish to Obsidian/GitHub

2. **Operational Documentation**
   - [ ] Create runbooks
   - [ ] Document troubleshooting
   - [ ] Create maintenance guides
   - [ ] Document common issues

3. **Knowledge Transfer**
   - [ ] Review docs with Jon
   - [ ] Answer questions
   - [ ] Update based on feedback
   - [ ] Final sign-off

**Deliverables:**
- Complete documentation
- Operational runbooks
- Knowledge transfer complete

---

## Documentation Strategy

### Obsidian (Knowledge Base)
**Location:** `/opt/data/docs/homelab-architecture/`

**Structure:**
```
homelab-architecture/
├── 00-INDEX.md (this file)
├── 01-ARCHITECTURE.md (detailed architecture)
├── 02-NETWORK.md (current live topology + gap analysis)
├── 03-ROUTING.md (routing rules — TBD)
├── 04-SECURITY.md (security posture + goal comparison) ✅
├── 05-IMPLEMENTATION.md (implementation details)
├── 06-OPERATIONS.md (runbooks, troubleshooting)
├── 07-IMPLEMENTATION-LOG-CASAOS-REIMPORT.md (Phase 2: CasaOS app re-import)
├── diagrams/
│   ├── network-topology.png
│   ├── routing-flow.png
│   └── security-layers.png
└── decisions/
    ├── 001-caddy-vs-traefik.md
    ├── 002-tailscale-serve.md
    └── 003-pihole-unbound.md
```

### GitHub (Version Control)
**Repository:** `ross-jon/hermes-backup` (or new repo)

**Structure:**
```
homelab/
├── docs/ (synced from Obsidian)
├── configs/
│   ├── caddy/
│   │   └── Caddyfile
│   ├── pihole/
│   │   └── setup.conf
│   └── unbound/
│       └── unbound.conf
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── health-check.sh
└── README.md
```

### Linear (Task Management)
**Project:** Homelab Infrastructure

**Structure:**
```
Epics:
├── Phase 1: Foundation
│   ├── Task: Restore Caddy
│   ├── Task: Verify Tailscale
│   └── Task: Fix CasaOS
├── Phase 2: DNS & Ad Blocking
│   ├── Task: Deploy Pi-hole
│   ├── Task: Deploy Unbound
│   └── Task: Network Integration
├── Phase 3: Optimization
│   ├── Task: Performance Tuning
│   ├── Task: Monitoring
│   └── Task: Backup & Recovery
└── Phase 4: Documentation
    ├── Task: Architecture Docs
    ├── Task: Operational Docs
    └── Task: Knowledge Transfer
```

### Decision Records
**Format:** Architecture Decision Records (ADRs)

**Template:**
```markdown
# ADR-001: Use Caddy as Reverse Proxy

## Status
Accepted

## Context
We need a reverse proxy to route traffic from Tailscale to Docker containers.

## Decision
Use Caddy instead of Traefik or Nginx.

## Rationale
- Simpler configuration (Caddyfile vs complex YAML)
- Automatic HTTPS (though we don't need it with Tailscale)
- Good Docker integration
- Active community

## Consequences
- Positive: Easy to maintain, good documentation
- Negative: Less features than Traefik
- Risks: May need to migrate if requirements change

## References
- https://caddyserver.com/docs/
```

---

## Next Steps

### Immediate Actions
1. **Review this document** - Confirm goals and approach
2. **Set up documentation structure** - Create Obsidian/GitHub/Linear
3. **Phase 1 kickoff** - Start with Caddy restoration

### Questions for Jon
1. Do you want a separate GitHub repo for homelab configs?
2. Should we use Linear's GitHub integration for task tracking?
3. Any specific monitoring tools you prefer (Grafana, Netdata, etc.)?
4. Do you want automated backups, or manual?

---

## Appendix

### Current Container Inventory
See `references/app-state.md` in `homelab-operations` skill

### Network Details
See `02-NETWORK.md` (to be created)

### Routing Table
See `03-ROUTING.md` (to be created)

---

**Document Version:** 1.0  
**Next Review:** After Phase 1 completion
