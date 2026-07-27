# Phase 1: Foundation - Implementation Plan

**Status:** Ready to Execute  
**Created:** 2026-01-XX  
**Estimated Duration:** 3-5 days  
**Dependencies:** None

---

## Overview

Phase 1 focuses on getting the core infrastructure working:
1. Restore and fix Caddy configuration
2. Verify Tailscale integration
3. Fix CasaOS app metadata
4. Ensure all apps are accessible via Tailscale domain

**Success Criteria:**
- ✅ All apps accessible via `https://homelab.taild32764.ts.net/<app>`
- ✅ CasaOS dashboard shows all apps with correct icons
- ✅ "Open" buttons route correctly through Tailscale
- ✅ No 502 errors or blank pages

---

## Task Breakdown

### Task 1.1: Backup Current State

**Objective:** Create restore point before making changes

**Steps:**
```bash
# Backup Caddy configuration
docker exec caddy cat /etc/caddy/Caddyfile > /opt/data/backups/caddy/Caddyfile.$(date +%Y%m%d)

# Backup all app compose files
for app in $(ls /var/lib/casaos/apps/); do
  docker run --rm -v /:/host alpine cat /host/var/lib/casaos/apps/$app/docker-compose.yml \
    > /opt/data/backups/casaos/$app.docker-compose.yml.$(date +%Y%m%d)
done

# Document current container state
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}" \
  > /opt/data/backups/container-state.$(date +%Y%m%d).txt

# Document current network state
docker network ls > /opt/data/backups/network-state.$(date +%Y%m%d).txt
for net in $(docker network ls --format "{{.Name}}"); do
  echo "=== $net ===" >> /opt/data/backups/network-state.$(date +%Y%m%d).txt
  docker network inspect $net >> /opt/data/backups/network-state.$(date +%Y%m%d).txt
done
```

**Deliverables:**
- [ ] Caddy config backed up
- [ ] All app compose files backed up
- [ ] Container state documented
- [ ] Network state documented

**Estimated Time:** 30 minutes

---

### Task 1.2: Rebuild Caddy Configuration

**Objective:** Create clean, working Caddyfile from scratch

**Current Issues:**
- Caddy config is broken (502 errors)
- Unclear routing rules
- Missing or incorrect handlers

**Approach:**
1. Start with minimal config
2. Add apps one by one
3. Test each addition
4. Document as we go

**Step 1: Create Minimal Config**

```caddyfile
{
    auto_https off
}

:8080 {
    # CasaOS (catch-all)
    handle {
        reverse_proxy host.docker.internal:80
    }
}
```

**Test:**
```bash
# Reload Caddy
docker exec caddy caddy reload --config /etc/caddy/Caddyfile

# Test CasaOS access
curl -s http://localhost:8080/ | head -5
# Expected: HTML from CasaOS dashboard
```

**Step 2: Add Vaultwarden (Simple Test)**

```caddyfile
{
    auto_https off
}

:8080 {
    # Vaultwarden API paths (must NOT be stripped)
    handle /api/* {
        reverse_proxy vaultwarden:80
    }
    handle /identity/* {
        reverse_proxy vaultwarden:80
    }
    handle /notifications/* {
        reverse_proxy vaultwarden:80
    }
    handle /icons/* {
        reverse_proxy vaultwarden:80
    }
    handle /webfonts/* {
        reverse_proxy vaultwarden:80
    }
    
    # Vaultwarden UI
    handle /vaultwarden/* {
        reverse_proxy vaultwarden:80
    }
    handle /vaultwarden {
        redir /vaultwarden/ 301
    }
    
    # CasaOS (catch-all)
    handle {
        reverse_proxy host.docker.internal:80
    }
}
```

**Test:**
```bash
# Reload Caddy
docker exec caddy caddy reload --config /etc/caddy/Caddyfile

# Test Vaultwarden
curl -sI http://localhost:8080/vaultwarden/ | head -5
# Expected: HTTP/1.1 200 OK

# Test via Tailscale
docker exec tailscale wget -qO- --timeout=5 https://homelab.taild32764.ts.net/vaultwarden/ | head -5
# Expected: HTML from Vaultwarden
```

**Step 3: Add ARR Stack (Group A)**

```caddyfile
# Add to Caddyfile before the catch-all handle

# Radarr
handle /radarr/* {
    reverse_proxy radarr:7878
}
handle /radarr {
    redir /radarr/ 301
}

# Sonarr
handle /sonarr/* {
    reverse_proxy sonarr:8989
}
handle /sonarr {
    redir /sonarr/ 301
}

# Lidarr
handle /lidarr/* {
    reverse_proxy lidarr:8686
}
handle /lidarr {
    redir /lidarr/ 301
}

# Readarr
handle /readarr/* {
    reverse_proxy readarr:8787
}
handle /readarr {
    redir /readarr/ 301
}

# Prowlarr
handle /prowlarr/* {
    reverse_proxy prowlarr:9696
}
handle /prowlarr {
    redir /prowlarr/ 301
}

# Bazarr
handle /bazarr/* {
    reverse_proxy bazarr:6767
}
handle /bazarr {
    redir /bazarr/ 301
}
```

**Test Each:**
```bash
for app in radarr sonarr lidarr readarr prowlarr bazarr; do
  echo "Testing $app..."
  curl -sI http://localhost:8080/$app/ | grep "HTTP/1.1"
done
# Expected: All return HTTP/1.1 200 OK
```

**Step 4: Add Media Apps (Group B)**

```caddyfile
# Jellyfin
handle_path /jellyfin/* {
    reverse_proxy jellyfin:8096
}

# Kavita
handle_path /kavita/* {
    reverse_proxy big-bear-kavita:5000
}

# Navidrome
handle /navidrome/* {
    reverse_proxy navidrome:4533
}
handle /navidrome {
    redir /navidrome/ 301
}

# Mylar3
handle_path /mylar3/* {
    reverse_proxy mylar3:8090
}

# Overseerr
handle_path /overseerr/* {
    reverse_proxy overseerr:5055
}
```

**Test Each:**
```bash
for app in jellyfin kavita navidrome mylar3 overseerr; do
  echo "Testing $app..."
  curl -sI http://localhost:8080/$app/ | grep "HTTP/1.1"
done
# Expected: All return HTTP/1.1 200 OK
```

**Step 5: Add Management Apps**

```caddyfile
# qBittorrent (via gluetun)
handle_path /qbittorrent/* {
    reverse_proxy qbittorrent:8080
}

# Syncthing
handle_path /syncthing/* {
    reverse_proxy big-bear-syncthing:8384
}

# Home Assistant
handle_path /homeassistant/* {
    reverse_proxy big-bear-home-assistant:8123
}

# ESPHome
handle_path /esphome/* {
    reverse_proxy esphome:6052
}

# Hermes
handle_path /hermes/* {
    reverse_proxy hermes:9119
}

# NZBGet
handle_path /nzbget/* {
    reverse_proxy nzbget:6789
}
```

**Test Each:**
```bash
for app in qbittorrent syncthing homeassistant esphome hermes nzbget; do
  echo "Testing $app..."
  curl -sI http://localhost:8080/$app/ | grep "HTTP/1.1"
done
# Expected: All return HTTP/1.1 200 OK
```

**Deliverables:**
- [ ] Minimal Caddyfile created and tested
- [ ] Vaultwarden added and tested
- [ ] ARR stack added and tested
- [ ] Media apps added and tested
- [ ] Management apps added and tested
- [ ] All apps accessible via localhost:8080

**Estimated Time:** 4-6 hours

---

### Task 1.3: Verify Tailscale Integration

**Objective:** Ensure all apps accessible via Tailscale domain

**Prerequisites:**
- Task 1.2 complete (Caddy working)

**Steps:**

**Step 1: Verify Tailscale Serve Config**
```bash
docker exec tailscale tailscale serve status
# Expected: https://homelab.taild32764.ts.net → http://localhost:8080
```

**Step 2: Test All Apps via Tailscale**
```bash
apps="vaultwarden radarr sonarr lidarr readarr prowlarr bazarr jellyfin kavita navidrome mylar3 overseerr qbittorrent syncthing homeassistant esphome hermes nzbget"

for app in $apps; do
  echo "Testing $app via Tailscale..."
  docker exec tailscale wget -qO- --timeout=5 https://homelab.taild32764.ts.net/$app/ | head -1
  if [ $? -eq 0 ]; then
    echo "  ✅ $app accessible"
  else
    echo "  ❌ $app FAILED"
  fi
done
```

**Step 3: Test from External Device**
- Open browser on phone/laptop connected to Tailscale
- Navigate to `https://homelab.taild32764.ts.net/<app>` for each app
- Verify pages load correctly

**Deliverables:**
- [ ] Tailscale serve config verified
- [ ] All apps accessible via Tailscale domain
- [ ] Tested from external device

**Estimated Time:** 1-2 hours

---

### Task 1.4: Fix CasaOS App Metadata

**Objective:** Ensure all apps show correctly in CasaOS dashboard

**Prerequisites:**
- Task 1.2 complete (Caddy working)
- Task 1.3 complete (Tailscale working)

**Current Issues:**
- Some apps show as "legacy" (need reimport)
- Missing or incorrect metadata
- Icons not loading
- "Open" buttons not routing correctly

**Approach:**
1. Audit all apps for metadata
2. Fix missing fields
3. Ensure icons are accessible
4. Test "Open" button routing

**Step 1: Audit Current Metadata**

```bash
echo "App,Has hostname,Has scheme,Has index,Has icon" > /tmp/casaos-audit.csv

for app_dir in /var/lib/casaos/apps/*/; do
  app=$(basename $app_dir)
  
  has_hostname=$(docker run --rm -v /:/host alpine grep -c "hostname:" /host/var/lib/casaos/apps/$app/docker-compose.yml 2>/dev/null || echo 0)
  has_scheme=$(docker run --rm -v /:/host alpine grep -c "scheme:" /host/var/lib/casaos/apps/$app/docker-compose.yml 2>/dev/null || echo 0)
  has_index=$(docker run --rm -v /:/host alpine grep -c "index:" /host/var/lib/casaos/apps/$app/docker-compose.yml 2>/dev/null || echo 0)
  has_icon=$(docker run --rm -v /:/host alpine grep -c "icon:" /host/var/lib/casaos/apps/$app/docker-compose.yml 2>/dev/null || echo 0)
  
  echo "$app,$has_hostname,$has_scheme,$has_index,$has_icon" >> /tmp/casaos-audit.csv
done

cat /tmp/casaos-audit.csv
```

**Expected Output:**
```
App,Has hostname,Has scheme,Has index,Has icon
bazarr,1,1,1,1
big-bear-dockhand,0,0,0,1
big-bear-home-assistant,1,1,1,1
...
```

**Step 2: Fix Missing Metadata**

For each app missing fields, update the compose file:

```bash
# Example: Fix big-bear-dockhand
docker run --rm -v /:/host alpine sh -c '
cat >> /host/var/lib/casaos/apps/big-bear-dockhand/docker-compose.yml << EOF

x-casaos:
  hostname: homelab.taild32764.ts.net
  scheme: https
  index: /
  icon: https://cdn.jsdelivr.net/gh/selfhst/icons/png/dockhand.png
EOF
'

# Restart the app
docker restart big-bear-dockhand
```

**Step 3: Verify Icon URLs**

```bash
# Extract all icon URLs
docker run --rm -v /:/host alpine sh -c '
for app in /host/var/lib/casaos/apps/*/; do
  name=$(basename $app)
  icon=$(grep "icon:" $app/docker-compose.yml | head -1 | awk "{print \$2}")
  echo "$name: $icon"
done
' > /tmp/icon-urls.txt

# Test each URL
while IFS=: read -r app url; do
  status=$(curl -sI -m 5 "$url" | grep "HTTP/1.1" | awk "{print \$2}")
  echo "$app: $status"
done < /tmp/icon-urls.txt
```

**Step 4: Test "Open" Button Routing**

For each app:
1. Open CasaOS dashboard
2. Click "Open" button
3. Verify it opens `https://homelab.taild32764.ts.net/<app>`
4. Verify page loads correctly

**Deliverables:**
- [ ] All apps audited for metadata
- [ ] Missing metadata added
- [ ] All icon URLs verified (HTTP 200)
- [ ] All "Open" buttons tested and working

**Estimated Time:** 3-4 hours

---

### Task 1.5: Ensure All Containers on web_proxy Network

**Objective:** Verify all web-accessible apps are on web_proxy network

**Prerequisites:**
- Task 1.2 complete (Caddy working)

**Steps:**

**Step 1: Check Current Network Membership**

```bash
echo "Container,On web_proxy" > /tmp/network-audit.csv

for container in $(docker ps --format "{{.Names}}"); do
  on_web_proxy=$(docker inspect $container --format='{{range $k, $v := .NetworkSettings.Networks}}{{$k}} {{end}}' | grep -c "web_proxy" || echo 0)
  echo "$container,$on_web_proxy" >> /tmp/network-audit.csv
done

cat /tmp/network-audit.csv
```

**Step 2: Add Missing Containers to web_proxy**

For containers that should be web-accessible but aren't on web_proxy:

```bash
# Example: Add esphome to web_proxy
docker network connect web_proxy esphome

# Verify
docker inspect esphome --format='{{range $k, $v := .NetworkSettings.Networks}}{{$k}} {{end}}'
# Expected: should include web_proxy
```

**Containers That Should Be on web_proxy:**
- All apps with Caddy routes (see Task 1.2)
- Exclude: gluetun, flaresolverr, tailscale, matter-server, obsidian-livesync

**Deliverables:**
- [ ] All web-accessible apps on web_proxy network
- [ ] Network membership documented

**Estimated Time:** 1-2 hours

---

### Task 1.6: Final Testing & Documentation

**Objective:** Comprehensive testing and documentation

**Steps:**

**Step 1: End-to-End Testing**

Test complete flow for each app:
1. Open browser on Tailscale device
2. Navigate to `https://homelab.taild32764.ts.net/<app>`
3. Verify page loads
4. Verify functionality (login, browse, etc.)

**Step 2: Document Configuration**

Update documentation with:
- Final Caddyfile
- Network topology
- App routing table
- Any deviations from plan

**Step 3: Create Runbook**

Document common operations:
- How to add a new app
- How to troubleshoot 502 errors
- How to reload Caddy
- How to restart services

**Deliverables:**
- [ ] All apps tested end-to-end
- [ ] Configuration documented
- [ ] Runbook created

**Estimated Time:** 2-3 hours

---

## Risk Mitigation

### Risk 1: Breaking Existing Functionality
**Mitigation:**
- Comprehensive backups (Task 1.1)
- Test after each change
- Rollback plan documented

### Risk 2: Incomplete App List
**Mitigation:**
- Audit all running containers
- Cross-reference with Caddy routes
- Verify each app manually

### Risk 3: Network Issues
**Mitigation:**
- Document current network state
- Test connectivity after changes
- Have rollback commands ready

---

## Success Metrics

### Quantitative
- [ ] 100% of apps accessible via Tailscale
- [ ] 0% 502 errors
- [ ] 100% of CasaOS "Open" buttons working
- [ ] 100% of icons loading

### Qualitative
- [ ] Clean, documented configuration
- [ ] Easy to troubleshoot issues
- [ ] Easy to add new apps
- [ ] Clear separation of concerns

---

## Timeline

| Task | Duration | Dependencies |
|------|----------|--------------|
| 1.1 Backup | 30 min | None |
| 1.2 Rebuild Caddy | 4-6 hours | 1.1 |
| 1.3 Verify Tailscale | 1-2 hours | 1.2 |
| 1.4 Fix CasaOS | 3-4 hours | 1.2, 1.3 |
| 1.5 Network Audit | 1-2 hours | 1.2 |
| 1.6 Final Testing | 2-3 hours | 1.2-1.5 |
| **Total** | **11-17 hours** | |

**Estimated Calendar Time:** 3-5 days (assuming 3-4 hours/day)

---

## Next Steps

1. **Review this plan** - Confirm approach and timeline
2. **Set up documentation** - Create Linear project, Obsidian structure
3. **Execute Task 1.1** - Start with backups
4. **Daily check-ins** - Review progress, adjust as needed

---

**Document Version:** 1.0  
**Next Review:** After Task 1.1 completion
