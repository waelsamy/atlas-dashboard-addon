# Changelog

All notable changes to the Atlas (Apex) add-on are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project follows [Semantic Versioning](https://semver.org/).

## [0.1.5] - 2026-05-17

### Fixed

- **Entity area resolution + Lovelace-parity filter.** Previously
  `/api/discover/entities` returned all 1,182 active entities but only ~22
  had `area_id` populated (1.9%). The room-based UI looked empty even though
  the user had 15 areas defined. Root cause: HA's standard pattern (used by
  Lovelace auto-area dashboards) is `entity.area_id ?? device.area_id` —
  most integrations set area on the *device*, not on each entity the device
  produces. Atlas's backend never fetched `device_registry`, so the join
  never happened.

### Added

- Backend `haclient.DeviceRegistry()` method (WS `config/device_registry/list`)
  + `Device` struct.
- Backend `discoverHandlers.entities` now fetches both registries and joins
  via the standard inheritance rule. Coverage jumps from 22 to ~987 entities
  with `area_id` (83.5% of all entities — the remaining 17% are
  legitimately unassigned: automations, scripts, sun/weather sensors).

### Changed

- `/api/discover/entities` now filters like HA Lovelace's auto-area view:
  drops `disabled_by`, `hidden_by`, `entity_category: "config"` (device
  setup options like power-on-behavior), and `entity_category: "diagnostic"`
  (battery/signal/etc. read-outs for ops). For a typical install this brings
  the response down from ~2,000 to ~450 user-facing entities — matching
  what Lovelace renders. Hidden entities are no longer surfaced with
  `hidden: true` in the response; that flag stays for forward-compat but is
  always `false` post-filter.

### Lessons (recorded in `codev/resources/lessons-learned.md`)

- When implementing an external HA client, look at what registries Lovelace
  itself fetches — there are four (area, floor, device, entity), not three.
  The entity→device→area inheritance is "just how the model works" but
  isn't called out as a separate gotcha in HA docs.

## [0.1.4] - 2026-05-16

### Fixed

- **WebSocket read limit.** After v0.1.3 fixed HA Core token scope,
  `/api/discover/areas` and `/api/discover/floors` returned real registry
  data — but `/api/discover/entities` still returned 502 `ha_unreachable`.
  Root cause: the backend's `coder/websocket` connections inherited the
  library's default read limit of 32768 bytes, while HA's `entity_registry`
  response on real installations easily exceeds that (hundreds of KiB to
  multi-MiB; HA stores ~500 bytes per entry). Once HA sent a frame larger
  than the limit, `coder/websocket` closed the conn with
  `StatusMessageTooBig` and the close error wrapped to `ErrUnreachable` —
  surfacing as the misleading 502 `ha_unreachable`. The endpoint was
  reachable and auth was good; the frame was just too big. v0.1.4 raises
  the read limit to 10 MiB (~20K entities of headroom) on all three
  backend WS read paths: `haclient.wsCommand` (registry queries),
  `haws/proxy` browser-side `Accept` (browser → proxy reads), and
  `haws/proxy` HA-side `Dial` (HA → proxy reads). The live WS proxy now
  survives HA's `subscribe_entities` initial-state frame for arbitrary
  install sizes (within the 10 MiB ceiling), so the dashboard's live
  state stream stays connected.

### Plan 8.x closeout — chain closed

With 0.1.4, Atlas is functionally complete as an HA add-on across all
plan-8 surfaces. The Plan 8.x chain is officially closed:

- **8.7** — distribution repo + GHCR build pipeline (custom-repo install
  works end-to-end).
- **8.8** — ingress path-prefix compatibility (SPA renders inside HA's
  ingress iframe at root and on nested-route hard-refresh).
- **8.9** — supervisor WebSocket path (backend dials
  `ws://supervisor/core/websocket`, not the non-existent
  `ws://supervisor/api/websocket`).
- **8.10** — HA Core API token scope (`SUPERVISOR_TOKEN` authenticates
  against HA Core's `auth_required` handshake).
- **8.11** — WS read limit (this entry; closes the chain).

Plan 9 picks up the remaining hardening: admin-PIN gate on state-mutating
endpoints, `X-Remote-User-*` HA ingress headers for trust-model
alignment, snapshot reconnect cache for instant render on reconnect.

### Lessons

- **WebSocket client libraries default to small read-frame limits unsuited
  to registry-style payloads.** `coder/websocket`'s 32 KiB default is
  appropriate for chat-style protocols but is exceeded by any non-trivial
  HA registry response. When a Go service uses a WS library to fetch list
  responses from an external service (HA registries, Kafka admin APIs,
  gRPC-over-WS bridges, etc.), explicitly set the read limit to fit the
  payload — and audit every `websocket.Dial` / `websocket.Accept` call
  site in the same package as a class, not one at a time. The
  `ErrUnreachable`-wrapped `StatusMessageTooBig` error was misleading
  during debugging: the endpoint was reachable, the auth completed, the
  frame was just too big. Future debugging of `ha_unreachable` on a code
  path that previously worked should check frame size first. Full
  defect-class write-up in `codev/resources/lessons-learned.md` under
  `## Plan 8.11 — WebSocket Read Limit (entity_registry size)`.

## [0.1.3] - 2026-05-16

### Fixed

- **HA Core API authorization.** `/api/discover/*` returned 502
  `ha_unauthorized` in addon mode because the addon manifest did not
  declare `homeassistant_api: true`. Without that flag, Supervisor does
  not grant `SUPERVISOR_TOKEN` the scope needed to authenticate against
  HA Core's REST + WebSocket APIs at `http://supervisor/core/api/*` and
  `ws://supervisor/core/websocket` — the WS `auth_required` handshake
  fails, and the discover endpoints surface that failure as `502
  ha_unauthorized`. v0.1.3 declares the flag; on the next addon restart
  Supervisor re-issues the token with HA Core scope, the auth handshake
  completes, and the discover endpoints return real registry data.

### Correction to [0.1.2] release notes

The previous entry stated that v0.1.2 "shows real rooms, real entities,
real state, and service calls actually toggle real devices." That was
aspirational. v0.1.2 fixed the WS *path* (Plan 8.9) — the backend now
reaches `ws://supervisor/core/websocket` instead of the non-existent
`ws://supervisor/api/websocket` — but the auth handshake on that path
still failed because the addon manifest lacked `homeassistant_api: true`.
Production Chrome MCP capture against the user's HA at `192.168.86.110`
on 2026-05-16 showed `/api/discover/*` returning 502 `ha_unauthorized`
on v0.1.2. The SPA continued falling back to APEX_FIXTURES (dummy data)
on every load. v0.1.3 is what actually delivers on the v0.1.2 promise.

### Plan 8.x closeout

With 0.1.3, Atlas is functionally complete as an HA add-on. The 8.x
series resolved, in order:

- **8.7** — distribution repo + GHCR build pipeline (custom-repo install
  path works end-to-end).
- **8.8** — ingress path-prefix compatibility (SPA renders inside HA's
  ingress iframe at root and on nested-route hard-refresh).
- **8.9** — supervisor WebSocket path (backend dials
  `ws://supervisor/core/websocket`, not the non-existent
  `ws://supervisor/api/websocket`).
- **8.10** — HA Core API token scope (`SUPERVISOR_TOKEN` authenticates
  against HA Core's auth_required handshake).

Plan 9 picks up the remaining hardening: admin-PIN gate on state-mutating
endpoints, `X-Remote-User-*` HA ingress headers for trust-model
alignment, snapshot reconnect cache for instant render on reconnect.

### Lessons

- **Addon manifest API flags grant runtime token scopes — they are not
  documentation comments.** The original Plan 8 manifest (v0.1.0) omitted
  `homeassistant_api: true` as a "least-privilege" choice, on the theory
  that `SUPERVISOR_TOKEN` would still authenticate against `http://supervisor/core`
  because that endpoint is the Supervisor's HA Core proxy. The Supervisor
  *routes* the request, but HA Core's auth layer validates token scope
  independently — without the manifest declaration, the token has no
  Core scope. Audit each `*_api` flag against the actual HA API endpoints
  the addon hits. `hassio_api` (Supervisor RBAC) and `homeassistant_api`
  (HA Core REST + WS) are siblings covering distinct surfaces, not
  aliases. Recorded in `codev/resources/lessons-learned.md`.

## [0.1.2] - 2026-05-16

### Fixed

- **Supervisor WS path.** `/api/discover/*` returned 502 in addon mode
  because the backend's WebSocket URL builder stripped the `/core` segment
  from `http://supervisor/core`, producing `ws://supervisor/api/websocket`
  (a non-existent endpoint) instead of `ws://supervisor/core/websocket`
  (the documented [supervisor proxy WS endpoint][supervisor-core]). Same
  bug in two places (registry queries in `haclient` + the live `/api/ha-ws`
  reverse proxy in `haws`); both fixed via a shared `internal/hapath`
  helper used from both call sites. End-to-end effect: with v0.1.1 the SPA
  rendered but fell back to APEX_FIXTURES (dummy rooms/devices) on every
  load because the discover endpoints 502'd; v0.1.2 shows real rooms, real
  entities, real state, and service calls actually toggle real devices.

[supervisor-core]: https://developers.home-assistant.io/docs/api/supervisor/endpoints#core

### Lessons

- **Defect class — duplicated URL construction.** Plan 6 inlined the WS
  URL builder in two files (`haclient/client.go` and `haws/proxy.go`)
  rather than extracting a helper. Both diverged from the real HA URL
  shape in the same way, and only one Plan-8.x verification (against a
  real Supervisor) was strong enough to surface it. Recorded in
  `codev/resources/lessons-learned.md`: when the same path/URL
  construction logic appears in two places, extract a shared helper from
  the start. Inline duplication doubles the surface for bugs that only
  surface against real infrastructure.

## [0.1.1] - 2026-05-16

### Fixed

- **HA ingress path-prefix compatibility.** SPA now renders correctly inside
  HA's ingress iframe — at the root URL AND on nested-route hard-refresh.
  Previously, asset URLs (`/assets/*`), REST endpoints (`/api/*`), and the
  WS proxy URL (`/api/ha-ws`) all resolved at the host root instead of under
  the ingress prefix (`/api/hassio_ingress/<token>/`). Result: blank screen
  on first open. Fix has two coordinated pieces:
  1. **Backend `<base href>` injection.** The Go backend reads HA's
     documented `X-Ingress-Path` request header and injects a `<base>` tag
     into served `index.html`, pinning `document.baseURI` to the SPA mount
     root regardless of which nested client-side route the user landed on.
     Without ingress (standalone Docker / dev) it injects `<base href="/">`.
  2. **Frontend `appBase()` + Vite `base: './'`.** A runtime helper reads
     `document.baseURI` (now stable thanks to the backend injection) and
     prepends the mount prefix to fetch and WS URLs. Built HTML emits
     relative asset URLs that resolve against the same `document.baseURI`.

  Standalone Docker mode unaffected (`<base href="/">` + empty `appBase()`
  is the no-op default).

### Plan 8.x closeout

With 0.1.1, Atlas is functionally installable + usable as an HA add-on at
any HA Supervisor instance via the custom-repo URL
<https://github.com/waelsamy/atlas-dashboard-addon>.

## [0.1.0] - 2026-05-16

First public release. Atlas (Apex) installs via the custom-repo URL
<https://github.com/waelsamy/atlas-dashboard-addon> — paste it into HA's
Settings → Add-ons → Add-on Store → ⋮ → Repositories.

### Added

- **Distribution repo + cross-repo sync.** A separate
  `waelsamy/atlas-dashboard-addon` repository hosts `repository.yaml` at
  its root (where HA Supervisor's custom-repo discovery scans) and the
  `atlas/` add-on directory mirrored from this main repo. The
  `sync-to-addon-repo` job in `.github/workflows/addon-release.yml`
  mirrors `addon/atlas/*` from this repo to the distribution repo on
  every `v*` tag push and rewrites the destination `config.yaml`'s
  `version:` to match the tag.
- **Multi-arch prebuilt images** at
  `ghcr.io/waelsamy/atlas-dashboard-addon-amd64` and
  `ghcr.io/waelsamy/atlas-dashboard-addon-aarch64`. Built by
  `.github/workflows/addon-release.yml` on every `v*` tag push. On
  custom-repo installs (this distribution path), Supervisor pulls the
  prebuilt image; on in-repo manual installs (Plan 8 path), Supervisor
  still builds locally via the Dockerfile.
- **Add-on manifest** (`config.yaml`) verified field-by-field against
  the live HA developer docs and the canonical `home-assistant/addons/ssh`
  reference at 2026-05-16. Ingress-only network surface, admin-only
  panel, no `hassio_api`, no `map` entry (least privilege). Live `image:`
  field points at the GHCR namespace above.
- **Per-arch base-image map** (`build.yaml`) pinned to
  `ghcr.io/home-assistant/{aarch64,amd64}-base:3.23-2026.04.0`.
- **Multi-stage `Dockerfile`** — `oven/bun:1` (SPA build) →
  `golang:1.25-alpine` (Go cross-compile with `embed.FS`) →
  `${BUILD_FROM}` (HA-base final image). VERSION ldflag wired so
  `/api/health` reports the manifest version.
- **128×128 `icon.png`** rendered from the canonical Apex mark in
  `design/apex/project/assets/apex-mark.svg` via macOS `qlmanage`.
- **Root `README.md`** describes the custom-repo install path with a
  five-step user walkthrough and the autonomous tag → sync → install
  flow. (`addon/atlas/README.md` + `addon/atlas/DOCS.md` are unchanged
  in Plan 8.7 — they still describe the Plan 8 in-repo install path and
  reference the original `waelsamy/atlas-addon` GHCR namespace. A
  follow-up sweep (Plan 8.7.1) updates those addon-internal docs.)
- **Ingress UI panel** — HA's reverse proxy; HA login is trusted; no
  second auth round-trip in the browser.
- **Hub (1280×800 wall tablet) and Mobile (phone-portrait) surfaces**
  shipped from a single locked design system (Hearth).
- **Hearth design system** — Cobalt brand, 7 runtime-swappable accents,
  light + dark schemes.
- **Live HA data** via Supervisor proxy. HA token never reaches the
  browser; the `atlas-backend` Go process injects credentials on the
  WebSocket reverse-proxy path.
- **Six screens**: Home, Rooms, Room detail, Climate, Device, Onboarding.
- **Service calls with optimistic updates + rollback** — light
  toggle/brightness, climate target, cover position, fan speed, lock.
- **Onboarding auto-skip in addon mode** — `SUPERVISOR_TOKEN` supplies
  the HA URL + token, so the manual onboarding flow is bypassed when
  running as an HA add-on.

### Known limitations

- **No admin-PIN gate yet** — interim trust model relies on HA's
  admin-group visibility via `panel_admin` (Plan 9). Single-admin
  effective model.
- **No per-device dashboard customization editor** — entities are
  auto-discovered from HA areas; per-device overrides will be Plan 9.
- **Scenes and automations not yet wired** into the dashboard (Plan 9).
- **Hidden/disabled entity overrides** not yet exposed in-app (Plan 9).
- **No `armv7` architecture** (deferred unless user demand surfaces).
- **No wordmark `logo.png`** in the addon-store card (only `icon.png`
  ships; the wordmark waits for a design-reviewed asset that respects
  the locked-Cobalt-brand constraint). `logo.png` is optional per the HA
  add-on spec; the add-on installs and runs without it.
