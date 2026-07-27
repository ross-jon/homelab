# Homelab Architecture Documentation

This directory contains the complete architecture and implementation plan for the homelab infrastructure project.

## Quick Start

1. **Read the overview:** Start with [00-INDEX.md](./00-INDEX.md) for goals, current state, and high-level plan
2. **Understand the architecture:** Review [01-ARCHITECTURE.md](./01-ARCHITECTURE.md) for detailed technical design
3. **Execute Phase 1:** Follow [05-IMPLEMENTATION-PHASE1.md](./05-IMPLEMENTATION-PHASE1.md) for step-by-step implementation

## Document Structure

```
homelab-architecture/
├── 00-INDEX.md              # Project overview, goals, phases
├── 01-ARCHITECTURE.md       # Detailed technical architecture
├── 02-NETWORK.md            # Network topology (to be created)
├── 03-ROUTING.md            # Routing rules (to be created)
├── 04-SECURITY.md           # Security model (to be created)
├── 05-IMPLEMENTATION-PHASE1.md  # Phase 1 implementation plan
├── 06-OPERATIONS.md         # Runbooks, troubleshooting (to be created)
├── diagrams/                # Architecture diagrams (to be created)
└── decisions/               # Architecture Decision Records (to be created)
```

## Project Phases

### Phase 1: Foundation (Current)
- Restore Caddy configuration
- Verify Tailscale integration
- Fix CasaOS app metadata
- Ensure all apps accessible via Tailscale domain

**Status:** Planning complete, ready to execute  
**Estimated Duration:** 3-5 days  
**See:** [05-IMPLEMENTATION-PHASE1.md](./05-IMPLEMENTATION-PHASE1.md)

### Phase 2: DNS & Ad Blocking
- Deploy Pi-hole for network-wide ad blocking
- Deploy Unbound for recursive DNS
- Integrate with router DHCP

**Status:** Not started  
**Estimated Duration:** 2-3 days

### Phase 3: Optimization
- Performance tuning
- Monitoring & alerts
- Backup & recovery

**Status:** Not started  
**Estimated Duration:** 2-3 days

### Phase 4: Documentation & Handoff
- Finalize all documentation
- Create operational runbooks
- Knowledge transfer

**Status:** Not started  
**Estimated Duration:** 2-3 days

## Key Components

### Current Stack
- **Tailscale:** External access, TLS termination, authentication
- **Caddy:** Reverse proxy, path-based routing
- **Docker:** Container orchestration
- **CasaOS:** Dashboard, container management
- **25+ containers:** ARR stack, media apps, utilities

### Planned Additions
- **Pi-hole:** Network-wide ad blocking
- **Unbound:** Recursive DNS resolver

## Access Patterns

### External Access (via Tailscale)
```
Browser → Tailscale (TLS) → Caddy (HTTP) → Container
```
- URL: `https://homelab.taild32764.ts.net/<app>`
- Auth: Tailscale handles authentication
- TLS: Terminated at Tailscale

### Local Access (LAN)
```
Browser → CasaOS (Port 80) → Container
```
- URL: `http://192.168.86.29` (CasaOS dashboard)
- Auth: None (trusted network)
- Direct app access: `http://192.168.86.29:<port>`

## Decision Records

Architecture decisions are documented in ADR format:

- **ADR-001:** Use Caddy as reverse proxy (not Traefik)
  - Rationale: Simpler config, good Docker integration
  - Trade-off: Less features than Traefik

- **ADR-002:** Use Tailscale serve for external access
  - Rationale: Secure, no public exposure, built-in auth
  - Trade-off: Requires Tailscale for external access

- **ADR-003:** Use Pi-hole + Unbound for DNS
  - Rationale: Network-wide ad blocking, privacy
  - Trade-off: Additional complexity

## Troubleshooting

### Common Issues

**502 Bad Gateway**
- Cause: Caddy can't reach container
- Fix: Check container is running and on web_proxy network
- See: [06-OPERATIONS.md](./06-OPERATIONS.md) (to be created)

**Blank White Page**
- Cause: Assets blocked (auth, CORS)
- Fix: Check app authentication settings
- See: [06-OPERATIONS.md](./06-OPERATIONS.md) (to be created)

**CasaOS "Legacy" Apps**
- Cause: Missing Docker Compose labels
- Fix: Recreate containers with proper labels
- See: [06-OPERATIONS.md](./06-OPERATIONS.md) (to be created)

## Maintenance

### Regular Tasks
- **Daily:** Monitor container health
- **Weekly:** Review logs, check for updates
- **Monthly:** Update blocklists, review security

### Backup Strategy
- **Configs:** Git repository (this directory)
- **Data:** Regular backups (TBD in Phase 3)
- **Recovery:** Documented procedures (TBD in Phase 3)

## Contributing

When making changes:
1. Update relevant documentation
2. Create ADR for significant decisions
3. Test thoroughly before deploying
4. Update this README if structure changes

## Contact

**Owner:** Jon Ross  
**Created:** 2026-01-XX  
**Last Updated:** 2026-01-XX

---

**Status:** Active development  
**Next Milestone:** Phase 1 completion
