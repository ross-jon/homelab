# Linear Project Setup - Homelab Infrastructure

**Project Name:** Homelab Infrastructure  
**Project Key:** HOMELAB  
**Repository:** https://github.com/ross-jon/homelab

---

## Epics

### Epic 1: Foundation (Phase 1)
**Description:** Get core infrastructure working - Caddy, Tailscale, CasaOS  
**Timeline:** Week 1  
**Status:** In Progress

### Epic 2: DNS & Ad Blocking (Phase 2)
**Description:** Implement Pi-hole + Unbound for network-wide ad blocking  
**Timeline:** Week 2  
**Status:** Not Started

### Epic 3: Optimization (Phase 3)
**Description:** Performance tuning, monitoring, backup & recovery  
**Timeline:** Week 3  
**Status:** Not Started

### Epic 4: Documentation & Handoff (Phase 4)
**Description:** Complete documentation, operational runbooks, knowledge transfer  
**Timeline:** Week 4  
**Status:** Not Started

---

## Tasks

### Epic 1: Foundation

#### Task 1.1: Backup Current State
**Priority:** High  
**Estimate:** 30 minutes  
**Status:** Ready

**Description:**
Create comprehensive backup of current state before making changes.

**Subtasks:**
- [ ] Backup Caddy configuration
- [ ] Backup all app compose files
- [ ] Document current container state
- [ ] Document current network state
- [ ] Store backups in /opt/data/backups/

**Acceptance Criteria:**
- All configs backed up with timestamps
- Container state documented
- Network state documented
- Backups verified

---

#### Task 1.2: Rebuild Caddy Configuration
**Priority:** High  
**Estimate:** 4-6 hours  
**Status:** Ready  
**Dependencies:** Task 1.1

**Description:**
Create clean, working Caddyfile from scratch with proper routing for all apps.

**Subtasks:**
- [ ] Create minimal Caddyfile with CasaOS catch-all
- [ ] Add Vaultwarden routes and test
- [ ] Add ARR stack routes (radarr, sonarr, lidarr, readarr, prowlarr, bazarr)
- [ ] Add media app routes (jellyfin, kavita, navidrome, mylar3, overseerr)
- [ ] Add management app routes (qbittorrent, syncthing, homeassistant, esphome, hermes, nzbget)
- [ ] Test all routes via localhost:8080
- [ ] Document routing decisions

**Acceptance Criteria:**
- All apps accessible via http://localhost:8080/<app>
- No 502 errors
- Routing documented
- Caddyfile committed to GitHub

---

#### Task 1.3: Verify Tailscale Integration
**Priority:** High  
**Estimate:** 1-2 hours  
**Status:** Ready  
**Dependencies:** Task 1.2

**Description:**
Ensure all apps accessible via Tailscale domain.

**Subtasks:**
- [ ] Verify tailscale serve config
- [ ] Test all apps via https://homelab.taild32764.ts.net/<app>
- [ ] Test from external device (phone/laptop)
- [ ] Document Tailscale configuration

**Acceptance Criteria:**
- All apps accessible via Tailscale domain
- No 502 errors
- Tested from external device
- Tailscale config documented

---

#### Task 1.4: Fix CasaOS App Metadata
**Priority:** Medium  
**Estimate:** 3-4 hours  
**Status:** Ready  
**Dependencies:** Task 1.2, 1.3

**Description:**
Ensure all apps show correctly in CasaOS dashboard with proper icons and routing.

**Subtasks:**
- [ ] Audit all apps for metadata (hostname, scheme, index, icon)
- [ ] Fix missing metadata fields
- [ ] Verify all icon URLs are accessible (HTTP 200)
- [ ] Test "Open" button routing for each app
- [ ] Document metadata schema

**Acceptance Criteria:**
- All apps have complete metadata
- All icons load correctly
- All "Open" buttons route to correct URL
- Metadata schema documented

---

#### Task 1.5: Ensure All Containers on web_proxy Network
**Priority:** Medium  
**Estimate:** 1-2 hours  
**Status:** Ready  
**Dependencies:** Task 1.2

**Description:**
Verify all web-accessible apps are on web_proxy network for Caddy routing.

**Subtasks:**
- [ ] Audit current network membership
- [ ] Add missing containers to web_proxy network
- [ ] Test connectivity from Caddy to each container
- [ ] Document network membership

**Acceptance Criteria:**
- All web-accessible apps on web_proxy network
- Caddy can reach all containers
- Network membership documented

---

#### Task 1.6: Final Testing & Documentation
**Priority:** High  
**Estimate:** 2-3 hours  
**Status:** Ready  
**Dependencies:** Tasks 1.2-1.5

**Description:**
Comprehensive end-to-end testing and documentation.

**Subtasks:**
- [ ] End-to-end testing of all apps
- [ ] Update architecture documentation
- [ ] Create operational runbook
- [ ] Document common issues and solutions
- [ ] Commit all documentation to GitHub

**Acceptance Criteria:**
- All apps tested end-to-end
- Documentation complete and accurate
- Runbook created
- All docs committed to GitHub

---

### Epic 2: DNS & Ad Blocking

#### Task 2.1: Deploy Pi-hole
**Priority:** High  
**Estimate:** 2-3 hours  
**Status:** Not Started

**Description:**
Deploy Pi-hole container for network-wide ad blocking.

**Subtasks:**
- [ ] Create Pi-hole Docker container
- [ ] Configure web interface
- [ ] Set up blocklists
- [ ] Test ad blocking
- [ ] Add Caddy route for /pihole

**Acceptance Criteria:**
- Pi-hole running and accessible
- Ad blocking working
- Web UI accessible via Tailscale

---

#### Task 2.2: Deploy Unbound
**Priority:** High  
**Estimate:** 2-3 hours  
**Status:** Not Started

**Description:**
Deploy Unbound as recursive DNS resolver.

**Subtasks:**
- [ ] Create Unbound Docker container
- [ ] Configure as recursive resolver
- [ ] Integrate with Pi-hole as upstream
- [ ] Test DNS resolution
- [ ] Benchmark performance

**Acceptance Criteria:**
- Unbound running
- Recursive DNS working
- Integrated with Pi-hole
- Performance acceptable

---

#### Task 2.3: Network Integration
**Priority:** High  
**Estimate:** 1-2 hours  
**Status:** Not Started

**Description:**
Integrate Pi-hole with router DHCP for network-wide DNS.

**Subtasks:**
- [ ] Update router DNS settings to point to Pi-hole
- [ ] Configure DHCP DNS option
- [ ] Test all devices on network
- [ ] Monitor performance
- [ ] Document router configuration

**Acceptance Criteria:**
- All devices using Pi-hole for DNS
- Ad blocking working network-wide
- DNS resolution working
- Router config documented

---

### Epic 3: Optimization

#### Task 3.1: Performance Tuning
**Priority:** Medium  
**Estimate:** 2-3 hours  
**Status:** Not Started

**Description:**
Optimize Caddy, Pi-hole, and Unbound performance.

**Subtasks:**
- [ ] Optimize Caddy configuration
- [ ] Tune Pi-hole settings
- [ ] Tune Unbound cache settings
- [ ] Monitor resource usage
- [ ] Document performance baselines

**Acceptance Criteria:**
- Performance optimized
- Resource usage acceptable
- Baselines documented

---

#### Task 3.2: Monitoring & Alerts
**Priority:** Medium  
**Estimate:** 3-4 hours  
**Status:** Not Started

**Description:**
Set up monitoring and alerting for infrastructure.

**Subtasks:**
- [ ] Choose monitoring tool (Grafana/Netdata/Uptime Kuma)
- [ ] Deploy monitoring stack
- [ ] Configure health checks
- [ ] Set up alerts for downtime
- [ ] Create dashboards

**Acceptance Criteria:**
- Monitoring deployed
- Health checks configured
- Alerts working
- Dashboards created

---

#### Task 3.3: Backup & Recovery
**Priority:** High  
**Estimate:** 3-4 hours  
**Status:** Not Started

**Description:**
Implement automated backup and recovery procedures.

**Subtasks:**
- [ ] Choose backup tool (Restic/BorgBackup)
- [ ] Configure backup schedules
- [ ] Set up offsite storage
- [ ] Test restore procedures
- [ ] Create recovery runbook

**Acceptance Criteria:**
- Automated backups running
- Offsite storage configured
- Restore tested and working
- Recovery runbook complete

---

### Epic 4: Documentation & Handoff

#### Task 4.1: Architecture Documentation
**Priority:** High  
**Estimate:** 2-3 hours  
**Status:** Not Started

**Description:**
Finalize all architecture documentation.

**Subtasks:**
- [ ] Complete 02-NETWORK.md
- [ ] Complete 03-ROUTING.md
- [ ] Complete 04-SECURITY.md
- [ ] Create network diagrams
- [ ] Create routing diagrams
- [ ] Publish to Obsidian/GitHub

**Acceptance Criteria:**
- All architecture docs complete
- Diagrams created
- Published and accessible

---

#### Task 4.2: Operational Documentation
**Priority:** High  
**Estimate:** 3-4 hours  
**Status:** Not Started

**Description:**
Create operational runbooks and maintenance guides.

**Subtasks:**
- [ ] Create 06-OPERATIONS.md
- [ ] Document common procedures
- [ ] Document troubleshooting steps
- [ ] Create maintenance schedules
- [ ] Document backup/restore procedures

**Acceptance Criteria:**
- Operations runbook complete
- All procedures documented
- Troubleshooting guide complete

---

#### Task 4.3: Knowledge Transfer
**Priority:** High  
**Estimate:** 2-3 hours  
**Status:** Not Started

**Description:**
Review documentation with Jon and transfer knowledge.

**Subtasks:**
- [ ] Review all docs with Jon
- [ ] Answer questions
- [ ] Update based on feedback
- [ ] Final sign-off
- [ ] Archive project

**Acceptance Criteria:**
- Jon reviewed all docs
- All questions answered
- Feedback incorporated
- Project archived

---

## Import Instructions

### Option 1: Manual Import to Linear
1. Go to Linear → Projects → New Project
2. Create project "Homelab Infrastructure"
3. Create 4 epics (one per phase)
4. Copy tasks from this document into each epic
5. Set priorities and estimates

### Option 2: CSV Import
1. Convert this document to CSV format
2. Import into Linear using CSV import
3. Map fields appropriately

### Option 3: Use as-is
Keep this document as the source of truth and manually track progress in Linear.

---

## Progress Tracking

**Phase 1 Progress:** 0/6 tasks complete  
**Phase 2 Progress:** 0/3 tasks complete  
**Phase 3 Progress:** 0/3 tasks complete  
**Phase 4 Progress:** 0/3 tasks complete  

**Overall Progress:** 0/15 tasks complete (0%)

---

**Last Updated:** 2026-01-XX  
**Next Update:** After Task 1.1 completion
