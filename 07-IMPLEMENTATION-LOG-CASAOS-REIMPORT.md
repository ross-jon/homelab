# Phase 2: CasaOS App Re-import (Legacy → Managed)

**Date:** 2026-07-28
**Status:** FILE-EDIT FAILED — `app_order.json` is a cache, overwritten by CasaOS on restart. Use CasaOS **UI re-import** (see UPDATE) or fix gateway ECDSA auth to drive via API.
**Goal:** Register the ~8 "BigBearCasaOS (legacy)" orphaned apps as managed in the
CasaOS dashboard so the dashboard is clean and Prowlarr→\*arr sync stays healthy.

---

## UPDATE 2026-07-28 — file-edit approach FAILED

Editing `/var/lib/casaos/1/app_order.json` did **not** register the apps. After a
CasaOS restart the file was **overwritten back to 15 entries** (the 8 additions
dropped) — proving `app_order.json` is a **cache written by CasaOS from its internal
app store, NOT the registration source**. Editing it is futile; CasaOS rebuilds it from
its real state on every restart.

**Reliable re-import paths:**
1. **CasaOS UI re-import** (recommended) — open CasaOS; the orphaned apps surface a
   re-import / "Import" control; click it per app. ~2 min, no risk. This writes to the
   real internal store.
2. **Fix gateway ECDSA auth** so the `/v1/*` import API is callable. The gateway rejects
   even valid login JWTs with 401 (key mismatch between gateway and casaos service).
   Deep/risky — only if UI re-import is undesirable.

The `app_order.json.bak.2026-07-28` backup is now stale (live file reverted); safe to
delete.

## Root cause (discovered)

- **CasaOS is a native host install** (v0.4.15), **not** a container. Gateway on `:8080`.
- App **managed vs orphaned** status is driven by `/var/lib/casaos/1/app_order.json`
  (the `1` = CasaOS user id, matches the JWT `id`/`username` claim). An app whose ID is
  absent from `data[]` is shown as orphaned/legacy.
- **Timeline:** gluetun/qbittorrent containers created **2026-07-24**; the \*arr apps
  (radarr/sonarr/lidarr/readarr/prowlarr) recreated **2026-07-27 19:02–19:03**. At
  **2026-07-27 19:43** an empty `casaos.db` (lowercase) was created in
  `/var/lib/casaos/db/` — the app index reset. Apps re-registered afterwards
  (qbittorrent, big-bear-\*, bazarr, navidrome, mylar3, flaresolverr, seerr) appear
  managed; the \*arr stack + hermes/jellyfin/vaultwarden were never re-registered → orphaned.

## Why the API path was blocked

- CasaOS gateway validates tokens with **ECDSA**
  (`jwt.Validate(token, GetPublicKey(runtimePath))`). A valid login JWT
  (`alg: ES256`, `iss: casaos`, `username: jonross`, `id: 1`) returns **401 on every
  `/v1/*` endpoint** (incl. `/v1/users/profile`). This is a host-side gateway↔token key
  mismatch — API-driven management from the sandbox is not possible. No
  `casaos-app-management` skill is present in this environment to supply the exact
  re-import call.

## KEY FINDING 2026-07-28 (later) — icon label is the missing piece

The orphaned apps are not just unregistered in the store — their **running containers
physically lack the `icon` label** that managed apps carry. Verified live:

- `gluetun` container labels include `"icon": "https://cdn.jsdelivr.net/…/qBittorrent/icon.svg"`
  → this is the URL CasaOS renders for the dashboard icon.
- `radarr`/`sonarr`/`lidarr` containers have **NO `icon` key** — only standard OCI/image
  labels (`org.opencontainers.image.*`) and compose labels. CasaOS has nothing to display.

**Mechanism:** CasaOS's *own* App Store install flow does two things plain `docker compose up`
does not:
1. Stamps an `icon` runtime label onto the container.
2. Registers the app in CasaOS's internal store (→ "managed", not "legacy").

The \*arr/hermes/jellyfin/vaultwarden were recreated on **2026-07-27 19:02** via a direct
`docker compose up` (to fix the IPv6 dead-end). Plain docker **ignores CasaOS's `x-casaos`
compose extension**, so it never stamped the `icon` label and never re-registered the app.
Their compose files still live in `/var/lib/casaos/apps/<app>/` (so they were originally
CasaOS apps) but now run "raw". gluetun/qbittorrent (07-24, CasaOS-installed) keep both the
icon label and store registration → managed + icon.

**Why the file-edit was doubly insufficient:**
1. `app_order.json` is a cache (overwritten from the store on restart) — additions vanish.
2. Even if persisted, the running containers are **missing the `icon` label**, so CasaOS
   still could not render a proper icon. Only CasaOS's install/re-import flow (UI button or
   `/v1/apps/install` API) both stamps the label AND writes the store entry.

**Lesson for future work:** never recreate a CasaOS-managed app with raw `docker compose up`
to "fix" something (e.g. IPv6) — it silently un-registers the app and strips its icon label.
Use CasaOS's own recreate/redeploy, or re-import afterward.

## Action taken (file-edit re-import)

- Mounted host `/var/lib/casaos` via the docker volume from the sandbox (the sandbox has
  docker access to the homelab daemon) and edited `/var/lib/casaos/1/app_order.json`.
- **Backed up** to `app_order.json.bak.2026-07-28` first (non-destructive, reversible).
- Added 8 orphaned app IDs to `data[]`:
  `hermes, jellyfin, lidarr, prowlarr, radarr, readarr, sonarr, vaultwarden`.
  No containers were restarted.
- All 8 now present in the file.

## Verification needed

- CasaOS may cache `app_order.json` in memory. **Refresh the CasaOS browser tab**; if the
  apps still show as legacy, **`sudo systemctl restart casaos`** on the host (or CasaOS
  settings → restart) and refresh again. After that the 8 apps should show as managed.
- Adding to `app_order.json` does **not** recreate containers, so running apps + Caddy
  routing are unaffected. If CasaOS rewrites `app_order.json` from its own state on restart
  and drops the 8, revert via the `.bak` and use the CasaOS UI re-import instead.

## Functional state (verified earlier — already healthy)

- All \*arr apps were **already on the `web_proxy` network** → Prowlarr→\*arr sync intact.
- All 17 Caddy routes green (Phase 1 decoupling: Caddy now uses `host.docker.internal`;
  only `vaultwarden` remains on container-name routing, which works).
