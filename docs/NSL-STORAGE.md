# Newborg Spatial Labs shared-storage convention (v1)

All NSL apps are served from the same origin https://knybbe.github.io (different paths), so they share
localStorage, IndexedDB, Cache API, OPFS and ONE storage quota. Rules:

1. localStorage private keys: `nsl:<app>:<key>` with app ids hub, pts (Photo to Splat), gsv (Godot viewer),
   p2 (Potree), pg (photogrammetry), 3dgen. Migrate old keys once on load (read old -> write new -> remove old),
   keep reading old key as fallback for one release.
2. Shared theme: key `nsl-theme` (values light|dark|system). Every app reads it FIRST on load and writes it on
   change. Stop writing other apps' private theme keys (gsv-theme, p2.theme, pg.theme, pts-settings.theme);
   listen to the `storage` event for `nsl-theme` so open tabs follow.
3. IndexedDB names: `nsl-<app>-<purpose>` (e.g. `nsl-pts-history`). Migrate existing data, then delete only the
   app's OWN old DB.
4. OPFS private dirs: `nsl-<app>/...`. Never touch other roots.
5. Shared large assets (AI model weights, big WASM runtimes meant for reuse): Cache API cache `nsl-models-v1`,
   keyed by the canonical absolute https URL (no query-string cache busters). Any app may add/read entries.
   NO app ever deletes `nsl-models-*` caches or entries except via an explicit user "Clear shared models" action.
6. Service workers: register only with scope `/<repo-path>/`, never `/`. Cleanup deletes only caches whose name
   contains the app's own prefix/scope. Never `caches.keys()`-then-delete-all. Never precache another app's files.
7. Eviction / "clear cache" features may only remove the app's own prefixed storage.
8. Call `navigator.storage.persist()` once (hub on first visit, or the first app that stores >50 MB), ignore refusal.
9. Never call localStorage.clear(), indexedDB.deleteDatabase on foreign names, or clear site data programmatically.
10. Secrets (API tokens) on this shared origin: keep them under `nsl:<app>:` and never log them; flag for later
    hardening (any page on knybbe.github.io can read them).

Reference helper (copy inline, no dependency, ~40 lines): `nslGet(app,key,legacyKeys[])`, `nslSet(app,key,val)`,
`getTheme()/setTheme()` on `nsl-theme`, `openModelCache()` -> caches.open('nsl-models-v1'),
`cachedFetch(url)` -> match in nsl-models-v1 else fetch + put (only for ok, non-opaque responses).
