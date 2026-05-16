# Changelog

All notable changes to the Atlas (Apex) add-on are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project follows [Semantic Versioning](https://semver.org/).

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
