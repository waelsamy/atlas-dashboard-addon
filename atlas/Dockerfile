# Plan 8 addon Dockerfile — multi-stage build (iter3 per architect direction).
#
# Architect rejected iter2's Path A (pre-built binary + minimal Dockerfile +
# scripts/build-addon-rootfs.sh) because Supervisor builds the image from the
# fetched repo contents — the pre-built binary would not exist there, COPY
# would fail. iter3 restores the multi-stage Dockerfile so the binary is
# produced inside docker build itself.
#
# Build context MUST be the repo root (so `COPY web/` and `COPY backend/`
# resolve). Plan 8 ships this as "developer install only" — see addon/
# README.md, root README.md, and addon/atlas/DOCS.md for the install path.
# Plan 8.5 shipped the build pipeline (.github/workflows/addon-release.yml
# produces multi-arch images at ghcr.io/waelsamy/atlas-addon-{aarch64,amd64}
# on every v* tag push). Real custom-repo distribution — separate
# waelsamy/atlas-addon repo with repository.yaml at the root level + atlas/
# at the root level + end-to-end Supervisor install verification — is Plan 8.7.
#
# BUILD_FROM is supplied per-arch by Supervisor's builder (via build.yaml's
# build_from map). Supervisor 2026.04.0+ does NOT auto-provide BUILD_FROM,
# so a sensible amd64 default is set here for direct `docker build` usage.

ARG BUILD_FROM=ghcr.io/home-assistant/amd64-base:3.23-2026.04.0

# Stage 1: SPA build (mirrors backend/Dockerfile:7-16 exactly — root bun.lock,
# not web/bun.lock; bun workspaces resolve from repo root).
FROM oven/bun:1 AS web-build
WORKDIR /app
COPY package.json bun.lock ./
COPY web/package.json ./web/
RUN bun install --frozen-lockfile
COPY web/ ./web/
RUN bun --filter=atlas-web run build

# Stage 2: Go cross-compile (mirrors backend/Dockerfile:18-34 + TARGETARCH).
# CGO_ENABLED=0 because modernc.org/sqlite is pure Go — no cgo dependency.
FROM golang:1.25-alpine AS go-build
ARG TARGETARCH
WORKDIR /src
COPY backend/go.mod backend/go.sum ./
RUN go mod download
COPY backend/ ./
# Bring in the freshly-built SPA so //go:embed all:web/dist works.
COPY --from=web-build /app/web/dist ./web/dist
ARG VERSION=0.1.5
RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build \
    -trimpath \
    -ldflags="-s -w -X main.Version=${VERSION}" \
    -o /atlas-backend \
    .

# Stage 3: HA addon final image. DIFFERS from backend/Dockerfile (which uses
# `FROM scratch` for the standalone Docker user); the addon variant ships
# HA's base image so Supervisor's expected runtime is present.
FROM ${BUILD_FROM}

LABEL org.opencontainers.image.title="Atlas (Apex)" \
      org.opencontainers.image.source="https://github.com/waelsamy/home-assistant-dash" \
      org.opencontainers.image.licenses="MIT"

COPY --from=go-build /atlas-backend /usr/bin/atlas-backend

# Plan 6's store.go writes to /data/atlas.db; this is the addon's
# persistent volume per HA convention. /data is auto-provisioned by
# Supervisor for every addon — no `map:` entry needed in config.yaml.
VOLUME ["/data"]

# Plan 6's main.go binds 0.0.0.0:8080 by default. Ingress proxies to it.
EXPOSE 8080

ENV PORT=8080 \
    DATA_DIR=/data \
    LOG_LEVEL=info

ENTRYPOINT ["/usr/bin/atlas-backend"]
