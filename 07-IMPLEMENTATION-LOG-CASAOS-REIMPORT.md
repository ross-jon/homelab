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
