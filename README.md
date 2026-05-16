# Atlas (Apex) — Home Assistant Add-on Distribution

Atlas is a modern, design-forward dashboard for Home Assistant. Hub (1280×800 wall tablet) and Mobile surfaces from a single locked design system.

**Source code + development:** <https://github.com/waelsamy/home-assistant-dash>

## Install (HA Supervisor)

1. In Home Assistant: **Settings → Add-ons → Add-on Store**
2. Open the **⋮** menu (top-right) → **Repositories**
3. Paste this URL:

   ```
   https://github.com/waelsamy/atlas-dashboard-addon
   ```

4. Click **Add**. The store refreshes and **Atlas (Apex)** appears.
5. Click **Install**. Wait for the addon image to pull (~30s for prebuilt images).
6. Click **Start**. Once status is green, click **Open Web UI** to launch.

The addon uses **ingress** — your HA login is trusted. No separate Atlas account.

## What's in this repo

This repo is a **distribution-only mirror**. The `atlas/` directory holds the add-on manifest, build recipe, and metadata that Home Assistant Supervisor consumes. Each release is auto-synced from the source repo via GitHub Actions.

- `repository.yaml` — repo-level identifier for HA
- `atlas/config.yaml` — addon manifest (ports, ingress, options, schema)
- `atlas/build.yaml` — multi-arch base-image mapping
- `atlas/Dockerfile` — minimal pull-from-GHCR shim
- `atlas/README.md` / `atlas/DOCS.md` / `atlas/CHANGELOG.md` — addon-store docs
- `atlas/icon.png` — addon-store icon

Prebuilt multi-arch images live at `ghcr.io/waelsamy/atlas-dashboard-addon-{amd64,aarch64}` and are pulled automatically by Supervisor on install.

## Development / issues

Code, issues, and discussion happen in the source repo: <https://github.com/waelsamy/home-assistant-dash>
