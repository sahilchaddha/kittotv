# Phase 4 — OTT API

## Goal

Build the viewer-facing HTTP API that backs the whitelabel OTT Web SPA. Separate deployable from `cms-api`. "Done" = a viewer on a tenant's custom domain can sign up, browse the catalog, manage a watchlist, and receive short-TTL signed playback URLs that play in `hls.js` / `dash.js`.

## Inputs / Outputs

- **Consumes:** HTTPS requests from `ott-web`. JWTs minted by itself (audience: `viewer`).
- **Produces:**
  - `viewers`, `watchlists`, `playback_sessions` documents.
  - JWT access + refresh tokens (audience: `viewer`).
  - Short-TTL CloudFront signed URLs (in production) / time-bound signed S3 URLs (locally against MinIO).

## Tech & Dependencies

- Express + TypeScript (`apps/ott-api`)
- Same Lambda-portable layering as `cms-api`.
- `mongodb` driver (read-only on CMS-owned collections: `tenants`, `themes`, `assets`).
- `ioredis` (refresh tokens, tenant-by-host cache).
- `bcrypt`, `jsonwebtoken`, `zod`, `pino`, `@aws-sdk/cloudfront-signer`.
- `packages/shared-types`, `packages/sdk` (ott client).

## Data Model Touch Points

- **Owns / writes:** `viewers`, `watchlists`, `playback_sessions`.
- **Reads (read-only):** `tenants` (host resolution), `themes`, `assets`.

## Endpoint Surface (v1)

- `GET /public/theme/by-host/:host` — resolves tenant by hostname, returns theme + tenant display fields. Cached aggressively (Redis 60s).
- `GET /public/catalog` — tenant inferred from `Host` header; returns featured/categories/list of `ready` assets.
- `GET /public/assets/:id` — public metadata only.
- `POST /auth/signup` — creates `viewers` row scoped to tenant inferred from `Host`.
- `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`.
- `GET /me`, `GET /watchlist`, `POST /watchlist/:assetId`, `DELETE /watchlist/:assetId`.
- `POST /playback/:assetId/ticket` — authenticated viewer; verifies asset belongs to host's tenant and is `ready`; returns `{ hlsUrl, dashUrl, expiresAt }` as short-TTL signed URLs (default 4h), plus creates a `playback_sessions` row for analytics.

## Tenant Scoping

Two enforcement points (both required, AND-ed):
1. **Host resolution.** Middleware resolves `Host` → `tenantId` via cached `tenants` lookup. If no match → 404.
2. **JWT audience + tenantId match.** Viewer JWT carries `tenantId`; if it disagrees with host-resolved `tenantId` → 401 (prevents stolen tokens from being replayed against another tenant's domain).

## Layering

Same shape as cms-api (`controllers / services / repositories / middleware / lib`).

## Milestones

- [ ] **M1** Express boot + health endpoint.
- [ ] **M2** Host-resolution middleware with Redis cache.
- [ ] **M3** Public theme + catalog endpoints (no auth required).
- [ ] **M4** Viewer signup + login + JWT (viewer audience). Verify token-vs-host mismatch is rejected.
- [ ] **M5** Watchlist CRUD.
- [ ] **M6** `playback/ticket` issuing time-bound signed URLs. Local mode uses S3 presigned GET; prod mode uses CloudFront signed URLs with a configured key pair.
- [ ] **M7** Cross-tenant isolation tests (token for tenant A on tenant B's domain → 401; asset from tenant A requested under tenant B's host → 404).
- [ ] **M8** Rate limiting on auth + ticket endpoints.

## Open Questions

- DRM license endpoint surface (left as a v2 hook; ticket response can grow a `drm` block later).
- Geo restrictions per asset — defer; document the `playback_sessions` row should record `country` for future enforcement.
- Anonymous browse (catalog visible without login)? Recommend yes, login required only for watchlist + playback ticket.

## Verification

- Boot `ott-api` against the dev Mongo + MinIO.
- Seed one tenant with `customDomain: localhost` and a `ready` asset (produced by Phase 1).
- `curl -H 'Host: localhost' /public/theme/by-host/localhost` returns the theme.
- Signup + login as a viewer; `curl /playback/<assetId>/ticket` returns playable URLs; load `hlsUrl` in hls.js → playback works.
- Cross-tenant: token from tenant A presented with `Host: tenantb.example` → 401.
