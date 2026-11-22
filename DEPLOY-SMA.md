# Homebox Deployment (tester-env)

Homebox v0.27.1 (`1fd28822634a3d5935c002d569559ef46530d4a0`), built from this
editable checkout with the upstream `Dockerfile`. Single container: the Nuxt SPA
is embedded in the Go binary, SQLite lives at `/data/homebox.db`, and attachments
live under `/data/data`.

## Quick Start

```bash
./tester-env deploy   # build image, start container, register the owner account
./tester-env seed     # import tester-env-seed.csv + fixed maintenance entries
./tester-env verify   # health, embedded SPA, auth, seed counts, seed digest
./tester-env reset    # drop container + /data volume
```

## Credentials

Non-demo mode with a fixed first user (registration makes it the collection
owner). Demo mode is not used: it blocks wipe-inventory, collection import,
user self-delete, password change and password reset.

- URL: `http://localhost:7745/` (browser container: `http://host.docker.internal:7745/`)
- Owner: `owner@example.com` / `homebox2026`
- Override with `HBOX_OWNER_EMAIL`, `HBOX_OWNER_PASSWORD`, `HBOX_OWNER_NAME`; the
  pepper is `HBOX_AUTH_API_KEY_PEPPER` (fixed default, >= 32 bytes).

## Environment and ports

`deploy` sets:

| Variable | Value | Why |
|---|---|---|
| `HBOX_AUTH_API_KEY_PEPPER` | fixed 39-byte value | startup fails fast without a >=32-byte pepper |
| `HBOX_DEMO` | `false` | keep wipe/import/user-delete/password-change available |
| `HBOX_OPTIONS_GITHUB_RELEASE_CHECK` | `false` | no hourly outbound release check |
| `HBOX_LOG_LEVEL` | `-1` | keep container logs small |

Image defaults are kept: `HBOX_MODE=production`,
`HBOX_DATABASE_SQLITE_PATH=/data/homebox.db?...&_pragma=journal_mode=WAL&_fk=1`,
`HBOX_STORAGE_PREFIX_PATH=data`, port 7745, healthcheck `/api/v1/status`.
Port mapping is `-p ${PORT:-7745}:7745`; `RUN_ID` scopes container
`tester-env-homebox[-<run-id>]` and volume `homebox-data[-<run-id>]`. `IMAGE_TAG`
(from `scripts/rl-env`) replaces the image name; `reset` keeps the image.

## Deterministic baseline

Registration creates one collection owned by the new user with 8 default
locations (Garage, Living Room, Kitchen, Bedroom, Bathroom, Office, Attic,
Basement) and 6 default tags (Appliances, IOT, Electronics, Servers, General,
Important).

`seed` then imports `tester-env-seed.csv` through the app's own CSV importer
(`POST /api/v1/entities/import`, `HB.*` headers), which creates missing nested
locations, reuses the default tags and fills every item field. The import is
keyed on `HB.import_ref` (`maple-01`..`maple-24`), so re-running `seed` updates
the same rows instead of duplicating them. Five fixed maintenance entries are
then ensured through `POST /api/v1/entities/{id}/maintenance` (matched by name).

Stable baseline (verified by `./tester-env verify`):

| Fact | Value |
|---|---|
| Active items | 23 (24 fixture rows, 1 archived: `Old Cordless Phone`) |
| Locations | 15 (8 roots + Garage/Workbench, Kitchen/Pantry, Living Room/Media Console, Office/Server Rack, Bathroom/Medicine Cabinet, Bedroom/Closet, Basement/Storage Shelves) |
| Tags | the 6 defaults only; filter counts General 13, Electronics 4, Servers 3, Appliances 3, IOT 2, Important 2 |
| Dashboard total value | `$7,291.82` (`sum(purchasePrice * quantity)` over active items) |
| Maintenance entries | 5 (Espresso Machine 2 completed; Rack Server 1 completed 2025-08-19 + 1 scheduled 2026-12-15; Wi-Fi Router 1 completed) |
| Search oracles | `Makita` -> 2, `Herman Miller` -> 1, `skillet` -> 1 |
| Quantity examples | Storage Bins 8, Winter Blankets 4, Water Filter Cartridges 4, Extension Cord 25 ft 3, LED Work Light 2 |

Item names sort alphabetically from `55-inch OLED TV` to `Winter Blankets`; the
home "recently added" table (pageSize 5, createdAt desc) ends with `Holiday
Lights`, `Storage Bins`, `Travel Suitcase`, `Winter Blankets`, `First Aid Kit`,
but `createdAt` is seed time, so do not assert relative dates.

`verify` also builds an id-free canonical snapshot (location paths, item fields
and tags, maintenance entries), sorts it and compares its SHA-256 with the pinned
`SEED_DIGEST` in `tester-env`; any extra, missing or edited record fails the
check. The same digest was reproduced across repeated `reset -> deploy -> seed`
cycles.

`seed`/`verify` authenticate with `POST /api/v1/users/login` (the returned token
already carries the `Bearer ` prefix). Both are plain curl + jq against
`/api/v1`, so no helper file is needed inside the container. The requirement
manifest lives next to the fixture in `tester-env-seed.json`.

## Browser proof for the seed

Log in as the owner and check:

1. `/home` quick stats: 23 total items, 15 locations, 6 tags, total value
   `$7,291.82`; storage locations show the 8 root cards, with nested children
   (e.g. Basement -> Storage Shelves, Office -> Server Rack).
2. `/items`: 23 rows, `Old Cordless Phone` absent; search `Makita` -> 2 rows
   (Cordless Drill, Impact Driver); search `Herman Miller` -> Office Chair;
   tag filter Electronics -> 4 rows.
3. Espresso Machine detail: breadcrumb `Kitchen > Pantry`, Breville `BES870XL`,
   serial `ESP-44102`, price `$649.00`, purchased 2023-11-24 from Kitchen & Co,
   tags Appliances + Important, maintenance log with 2 completed entries
   (2026-02-14, 2025-12-06).
4. `/maintenance`: 5 entries, including the scheduled `Firmware update and BIOS
   check` on Rack Server (2026-12-15).

## Build cost (measured 2026-11-22, 28 cores, Docker 28.5.1 / buildx 0.29.1)

| Path | Time |
|---|---|
| Cold build (fresh builder cache, base images pulled) | 172.9 s |
| Frontend mutation (`frontend/pages/index.vue` string change) | ~57 s (`pnpm build` 33 s + `go build`/copy/export) |
| Comment-only change in `frontend/nuxt.config.ts` | ~61 s (`pnpm build` re-runs, Go layer reused) |
| Go-only mutation (`backend/app/api/routes.go`) | ~50 s (`go build` itself ~6 s; apk + module-copy revalidation dominates) |
| Redeploy with a cached image / reset cycle | ~5 s / ~7 s |
| No-change relayer | ~0.1 s |

Upstream already separates `pnpm install` (package.json + lockfile layer), the
Nuxt build and the Go build, and mounts `/root/.cache/go-build`, so small
mutations never repeat dependency installs.

## Caveats

- The PWA service worker (`/sw.js`, `@vite-pwa/nuxt`, autoUpdate) caches API
  responses NetworkFirst and can mask a frontend mutant until reload; the fork
  may need PWA disabled for frontend mutations.
- Auth rate limiting is upstream default (5 failed attempts/minute with
  backoff, reset on success); a wrong-password scenario can see 429s for
  minutes. Set `HBOX_AUTH_RATE_LIMIT_ENABLED=false` for such a scenario rather
  than changing the baseline.
- `createdAt` is seed time and drives "recently added" and relative dates;
  assert on fixed purchase/warranty dates or counts instead.
- Base images (`node:22-alpine`, `golang:alpine`, `alpine:latest`,
  `mirror.gcr.io`) are unpinned upstream, so a rebuild after base-image drift
  can invalidate otherwise cached layers. Pin digests in the fork if needed.
- Registration stays open (`HBOX_OPTIONS_ALLOW_REGISTRATION` default true), so a
  second user can register and join via invitations for permission scenarios.

## Reset

`./tester-env reset` stops and removes the container and the `/data` volume, so
the SQLite DB, WAL files and attachments are gone; `deploy` re-runs the goose
migrations and `seed` rebuilds the same baseline. The image is intentionally not
removed (content-addressed tags are shared across runs).

## Evidence (2026-11-22)

- Build: upstream `Dockerfile` unchanged from tag `v0.27.1`; cold build 172.9 s,
  Go-only mutation ~50 s, frontend mutation ~57 s, cached redeploy ~5 s.
- Reset: `reset -> deploy -> seed -> verify` ran three times; the SQLite DB,
  WAL and attachments were dropped each time and the id-free seed digest came
  back identical (`sha256 6a6eb33070b21607df0ee88e1b0cf45774213886d4af854fc4daaa6d47541f04`).
- Browser smoke: create + delete of an item verified in Chromium (state-changing
  assertion, gone after reset); login, `/home`, `/items` and item detail render.
- Seed: browser-verified dashboard 23 items / 15 locations / 6 tags /
  `$7,291.82`, search `Makita` -> 2, tag Electronics -> 4, Espresso Machine
  detail (breadcrumb, Breville BES870XL, serial ESP-44102, `$649.00`,
  2023-11-24, Kitchen & Co, Appliances + Important, 2 maintenance entries) and
  `/maintenance` -> 5 entries incl. the scheduled Rack Server 2026-12-15 row.
- `verify` reproduces the same assertions from the API, so the CLI gate matches
  the browser proof without a browser.
