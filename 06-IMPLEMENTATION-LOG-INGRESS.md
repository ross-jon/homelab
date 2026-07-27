# Phase 1 Implementation Log — Ingress (Tasks 1.2 + 1.3)

**Date:** 2026-07-27  
**Status:** ✅ Working  
**Decision:** Option B (Caddy terminates TLS with Tailscale-issued cert; `tailscale serve` removed)

---

## What was broken

`tailscale serve` (the original ingress) was **non-functional** because the containerized
Tailscale cannot initialize iptables (kernel modules blocked by secureboot/lockdown —
"Operation not permitted" even with NET_ADMIN). `tailscale serve` relies on iptables,
so its TLS proxy silently failed (intermittent NO-RESPONSE / 502 to all apps).

## What we implemented (Option B)

Caddy now terminates TLS directly using a Tailscale-issued Let's Encrypt cert and listens
on `:443` (published on host, so the Tailscale IP `100.103.106.49:443` reaches it).
`tailscale serve` was removed entirely. Tailscale still provides device auth + the tunnel;
it just no longer does the HTTP proxy.

### Architecture
```
Browser (Tailscale device)
   │  https://homelab.taild32764.ts.net/<app>/
   ▼
Tailscale tunnel → host 100.103.106.49:443
   ▼
Caddy :443  (TLS terminated with /etc/caddy/certs/homelab.taild32764.ts.net.{crt,key})
   │  reverse_proxy <container>:<port>  (+ X-Forwarded-Proto so apps emit https://)
   ▼
App container (on web_proxy network, or host.docker.internal for host-net apps)
```

### Caddyfile (two entrypoints, shared routes)
- `homelab.taild32764.ts.net:443` — TLS via Tailscale cert; external/Tailscale entry
- `:8080` — plain HTTP; LAN/CasaOS dashboard entry
- Both `import routes` (a snippet with all the `handle` blocks)
- `(fwd)` snippet sets `X-Forwarded-Proto {http.request.scheme}` so apps generate
  correct https:// URLs whether hit via :443 or :8080

### Key gotchas (learned the hard way)
1. **`tailscale cert` must generate cert+key in ONE command** — running it twice with
   different `--key-file` paths produced a cert whose key did NOT match (TLS
   "internal error"). Fix: delete old files, run once with both `--cert-file` and
   `--key-file`, verify with openssl pubkey comparison.
2. **`host.docker.internal` is NOT auto-resolved on Linux user-defined networks.**
   Must pass `--add-host host.docker.internal:host-gateway` to the caddy container.
   (Apps using it: qbittorrent/gluetun:8181, homeassistant:8123, esphome:6052,
   syncthing:8384.)
3. **SNI matters for testing** — connecting to the IP with SNI=IP fails (no cert for IP).
   Always test with SNI = `homelab.taild32764.ts.net` (what a real browser does).
4. **Caddy snippet order** — snippets (`(routes)`, `(fwd)`) must be DEFINED before
   they are `import`ed, or Caddy errors "File to import not found".
5. **Recreating caddy** — it was started via `docker run` (not CasaOS compose). Run cmd:
   `docker run -d --name caddy --network web_proxy --add-host host.docker.internal:host-gateway
   --restart unless-stopped -p 8080:8080 -p 443:443 -p 10380:10380
   -v /DATA/AppData/caddy/Caddyfile:/etc/caddy/Caddyfile
   -v /DATA/AppData/caddy/data:/data
   -v /DATA/AppData/tailscale/certs:/etc/caddy/certs:ro caddy:alpine
   caddy run --config /etc/caddy/Caddyfile --adapter caddyfile`

### Verification (all 18 apps return 200/30x via the domain)
vaultwarden, radarr, sonarr, lidarr, readarr, prowlarr, bazarr, jellyfin, overseerr,
kavita, navidrome, mylar3, seerr, qbittorrent, homeassistant, esphome, syncthing, hermes.

---

## TODO before declaring done
- [ ] **Cert renewal** — Let's Encrypt cert expires ~Oct 25 2026. Set up auto-renew
      (script + cron on host). See `renew-tailscale-cert.sh`.
- [ ] CasaOS one-click metadata (Task 1.4) — icons + web URLs pointing to the domain.
- [ ] Network membership (Task 1.5) — verify all web-facing apps on web_proxy.
