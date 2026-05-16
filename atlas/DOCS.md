# Atlas (Apex) — Add-on Reference Documentation

This document describes the Atlas add-on's reference artifacts: what the manifest declares, what the Dockerfile produces, how the security boundary is shaped, and what configuration options will exist when the add-on is installable end-to-end. The actual user-facing install + first-run + troubleshooting docs land with **Plan 8.7** when the distribution repo (`waelsamy/atlas-addon`) ships.

## Status: build pipeline shipped (Plan 8.5), install path deferred (Plan 8.7)

Plan 8 shipped the packaging metadata as reference artifacts. Plan 8.5 shipped the build pipeline (`.github/workflows/addon-release.yml` produces multi-arch images at `ghcr.io/waelsamy/atlas-addon-{aarch64,amd64}` on every `v*` tag push). Plan 8.6 (this iteration's docs cleanup) uncommented the `image:` field in `config.yaml` to point at those images. The final missing link — a separate `waelsamy/atlas-addon` distribution repo with `repository.yaml` at the root level so HA's custom-repo discovery resolves correctly — is **Plan 8.7** work. See [`../README.md`](../README.md) for the full scope breakdown.

This document is structured as reference now so it carries forward into Plan 8.7 with minimal rewriting.

## What Atlas is

Atlas is a custom React app that talks to your Home Assistant instance over WebSocket and REST. It replaces the Lovelace dashboard for two surfaces: a wall-mounted tablet at 1280×800 (Hub) and a phone in portrait (Mobile). Both ship from the same Hearth design system — Cobalt brand, light and dark schemes, seven swappable accents.

The add-on bundles two pieces of software into one container:

- A Go backend (`atlas-backend`) that proxies the browser's WebSocket connection to Supervisor's HA endpoint, injecting the `SUPERVISOR_TOKEN` server-side so the browser never sees an HA token.
- A React single-page app, embedded in the Go binary via `embed.FS`. The backend serves the SPA from the same listener as the API; Supervisor's ingress provides the public URL.

The add-on runs as a single process inside the container. Supervisor's ingress is the only network surface; no host port is mapped.

## Manifest fields (config.yaml)

The `config.yaml` declares the runtime contract. Field-by-field:

- `name: Atlas (Apex)`, `slug: atlas`, `version: "0.1.0"` — identity.
- `arch: [aarch64, amd64]` — Pi 4/5 + NUC/generic x86 only; `armv7` deferred.
- `image: "ghcr.io/waelsamy/atlas-addon-{arch}"` — points at the multi-arch images published by `.github/workflows/addon-release.yml` (Plan 8.5). Supervisor honors this field only on custom-repo installs (Plan 8.7); for the current in-repo manual install path, Supervisor still builds locally.
- `init: false` — atlas-backend handles its own signal/shutdown; HA's s6-overlay would conflict.
- `startup: application` — wait for HA Core + integrations before binding. The Plan 6 backend's `discover/*` endpoints need the area/floor/entity registries populated.
- `watchdog: "http://[HOST]:[PORT:8080]/api/health"` — Supervisor pings Plan 6's health endpoint; restarts on hang.
- `ingress: true`, `ingress_port: 8080`, `ingress_entry: "/"` — Supervisor's reverse proxy is the only network surface.
- `panel_icon: mdi:view-dashboard-variant`, `panel_title: Atlas`, `panel_admin: true` — sidebar panel, admin-only visibility.
- `ports: {8080/tcp: null}` — no host port mapped. Ingress is the only path.
- **No `hassio_api` / `homeassistant_api` flags.** Plan 8 ships least-privilege; the runtime uses `SUPERVISOR_TOKEN` (auto-injected) to talk to `http://supervisor/core` and needs no explicit API flag. A future Plan 9 adds the smallest privilege when device-pairing actually needs Supervisor or Core REST access.
- **No `map` entry.** Atlas writes to `/data/atlas.db`, which Supervisor auto-provisions for every add-on as the persistent volume. `map: [addon_config:rw]` (which exposes `/addon_configs/<slug>/`) is unused by the backend and intentionally omitted to shrink the permission surface.
- `options: {}`, `schema: {}` — Plan 8 ships zero user-facing options. Settings live in `/api/config`; the admin-PIN flow lands in Plan 9.

## Build recipe (Dockerfile)

The `Dockerfile` is multi-stage:

- **Stage 1 (`web-build`):** `FROM oven/bun:1`. Installs deps from the repo-root `bun.lock` (NOT a non-existent `web/bun.lock`), builds the React SPA via `bun --filter=atlas-web run build`. Output at `/app/web/dist`.
- **Stage 2 (`go-build`):** `FROM golang:1.25-alpine` (matches `backend/go.mod 1.25.6`). Cross-compiles with `CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH}` (Plan 6's `modernc.org/sqlite` is pure-Go; no cgo). Pulls the stage-1 SPA into `backend/web/dist/` so `//go:embed all:web/dist` resolves. Links with `-trimpath -ldflags="-s -w -X main.Version=${VERSION}"` so `/api/health` reports the manifest version.
- **Stage 3 (final):** `FROM ${BUILD_FROM}` (default `ghcr.io/home-assistant/amd64-base:3.23-2026.04.0`). LABELs (org.opencontainers.image.title/source/licenses). Copies the cross-compiled binary into `/usr/bin/atlas-backend`. `VOLUME ["/data"]`. `EXPOSE 8080`. `ENV PORT=8080 DATA_DIR=/data LOG_LEVEL=info`. `ENTRYPOINT ["/usr/bin/atlas-backend"]`.

**Build context = repo root.** The Dockerfile's `COPY web/` and `COPY backend/` need the repo root present in the docker context. A reference build runs as:

```bash
docker build \
  --build-arg BUILD_FROM=ghcr.io/home-assistant/amd64-base:3.23-2026.04.0 \
  --build-arg TARGETARCH=amd64 \
  -f addon/atlas/Dockerfile \
  -t atlas-addon:dev \
  .
```

Replace `amd64` with `aarch64` (and `TARGETARCH=arm64` — Go's GOARCH naming differs from HA's arch naming) for ARM builds. Expected image size ~60–80 MB (HA base ~40 MB + binary ~12 MB + bun/go intermediate layers).

This reference build does NOT install the result into a running Supervisor — that's the Plan 8.7 work (custom-repo install + end-to-end verification). The build verifies the recipe composes correctly and the resulting image is structurally sound. Plan 8.5's `workflow_dispatch` dry-run additionally verifies that the recipe builds successfully under GitHub Actions Buildx on both `linux/amd64` and `linux/arm64`.

## Configuration options

None in Plan 8 — `options: {}` and `schema: {}` in `config.yaml`. Future surfaces:

- **Admin PIN** — argon2id-hashed in SQLite, gates the admin panel. Lands in Plan 9.
- **Per-device dashboard customization** — JSON dashboards stored in `/data/atlas.db`. Plan 9.
- **Hidden/disabled entity overrides** — user-controlled entity visibility. Plan 9.

Until those land, Atlas inherits HA's own access control: `panel_admin: true` restricts panel visibility to HA admins, and ingress terminates the auth boundary before any request reaches the backend.

## Security model

- **`ports: 8080/tcp: null`** — no host port mapped. Atlas is unreachable from the LAN except through Supervisor's ingress.
- **`ingress: true`** — Supervisor's reverse proxy is the only public surface. HA authentication terminates at ingress before forwarding to the backend.
- **`panel_admin: true`** — sidebar panel + "Open Web UI" button visible only to HA admins. Interim trust model until the Plan 9 admin-PIN gate ships.
- **No `hassio_api` / `homeassistant_api`** — least privilege. The backend's HA access flows through `SUPERVISOR_TOKEN`, auto-injected by Supervisor regardless of API flags.
- **No `map` entry** — smallest permission surface. Atlas writes only to `/data/atlas.db` (auto-provisioned per-add-on volume).
- **`SUPERVISOR_TOKEN` server-side only** — never reaches the browser. The WS reverse-proxy injects authentication during the handshake.

## Troubleshooting (future reference)

When Plan 8.7 ships the install path, the most common failure modes will be:

- **Add-on doesn't appear in the store after adding the custom repo:** force the store to refresh via the ⋮ menu → Check for updates.
- **Install fails with a build error:** check Supervisor logs (Settings → System → Logs → Supervisor). Usually a transient network issue while pulling the base image; re-run install.
- **Can't reach HA from the dashboard:** check the Atlas add-on's own log tab for `auth_invalid` or `connection refused`. Verify HA Core is healthy; restart Atlas if Supervisor recently restarted Core.
- **Panel doesn't load:** confirm the add-on is started (green status). Confirm your HA user has admin privileges. Try **Open Web UI** from the detail page directly.
- **Updating the add-on:** Supervisor auto-detects new versions in the distribution repo and offers an Update button. SQLite state in `/data/atlas.db` persists across updates.
- **Uninstalling:** Stop → Uninstall. Supervisor removes the `/data/` volume automatically.

## Local development

For contributors iterating on Atlas itself (not just installing it), the source repo ships a Home Assistant Apps devcontainer that runs a local Supervisor + builds Atlas in-place. See [DEVELOPMENT.md](https://github.com/waelsamy/home-assistant-dash/blob/main/DEVELOPMENT.md) for the full walkthrough. The short version: VS Code + Dev Containers extension → **Reopen in Container** → `ha apps rebuild --force "local_atlas"` per iteration.

## Reporting issues

File issues against the upstream Atlas repository:

- GitHub Issues: `https://github.com/waelsamy/home-assistant-dash/issues`

When reporting (post-Plan-8.7):

- Atlas add-on version (visible on the add-on detail page)
- HA Core version + Supervisor version
- Architecture (aarch64 or amd64)
- A snippet of the relevant Supervisor or add-on log

The maintainer is Wael Ewida (`waelsamy@gmail.com`).
