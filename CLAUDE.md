# KittoTV — Whitelabel OTT Platform

## 1. Project Overview

KittoTV is a multi-tenant, whitelabel OTT (over-the-top streaming) platform. Tenants sign up as operators, upload their own catalog of raw movies, customize the look of their viewer-facing site, and serve a Netflix-style streaming experience to their end users — all under their own brand and (optionally) custom domain.

The system has five cooperating components:

1. **CMS Dashboard** — operator UI. Tenants manage their catalog, upload raw video, edit metadata, and customize player branding/theme.
2. **CMS Backend API** (`apps/cms-api`) — multi-tenant Node/Express service. Backs the dashboard, issues presigned raw-upload URLs, enqueues transcode jobs, owns tenant + catalog + theme data.
3. **Transcoding Pipeline** (`apps/transcoder-worker`) — queue-driven. On-prem FFmpeg workers poll for pending jobs, download raw from the `raw` S3 bucket, transcode into HLS + DASH adaptive ladders, and push outputs to the `delivery` S3 bucket.
4. **OTT Web (Whitelabel)** (`apps/ott-web`) — Netflix-style React SPA. End users sign up and stream. The same deploy serves every tenant; theme + catalog are resolved per hostname at runtime.
5. **OTT API** (`apps/ott-api`) — viewer-facing Node/Express service. Separate deployable from `cms-api`. Handles viewer auth, public catalog browse, watchlist, theme resolution by host, and short-TTL signed playback URLs.

**Audiences:**
- *Tenant operators* — use the CMS Dashboard to manage their service.
- *End viewers* — use their tenant's OTT Web site to sign up and watch.

**Portability constraint:** the architecture must stay portable to AWS Lambda. Express is the current transport, but business logic must not depend on it (see §9).

## 2. Architecture

```
            ┌───────────────────┐         ┌──────────────────┐
  Operator ─▶  CMS Dashboard    │──HTTPS──▶  CMS API (Node)  │──▶ MongoDB
            │  (React + Vite)   │         │  Express + JWT   │──▶ Redis
            └───────────────────┘         └────────┬─────────┘
                                                   │
                                   presigned PUT   │  enqueue transcode_job
                                                   ▼
                                          ┌──────────────────┐
                       raw upload  ──────▶│  S3 raw bucket   │
                                          └────────┬─────────┘
                                                   │ download
                                                   ▼
                                          ┌──────────────────┐
                                          │ Transcoder Worker │
                                          │ (on-prem, FFmpeg) │
                                          └────────┬──────────┘
                                                   │ aws s3 cp
                                                   ▼
                                       ┌──────────────────────┐
                                       │ S3 delivery bucket   │
                                       │   (HLS + DASH)       │
                                       └──────────┬───────────┘
                                                  │
                                              CloudFront
                                                  │
            ┌───────────────────┐                 │
  Viewer ──▶│   OTT Web SPA     │◀────────────────┘
            │  (React + Vite)   │  HLS/DASH manifests + segments
            └─────────┬─────────┘
                      │ tenant theme + catalog + signed playback URL
                      ▼
            ┌───────────────────┐
            │     OTT API       │──▶ MongoDB (viewers, watchlists,
            │  Express + JWT    │     read-only on CMS catalog/theme)
            └───────────────────┘
```

## 3. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Language | TypeScript (strict) | Shared types across CMS/player/worker |
| CMS API | Node.js + Express | Familiar; Lambda-portable if handlers stay thin |
| CMS Dashboard | React + Vite | Fast DX, SPA, no SSR needed |
| OTT API | Node.js + Express | Separate deployable from CMS API; viewer-facing only |
| OTT Web | React + Vite SPA | Static, CloudFront-cacheable, cheap to scale |
| DB | MongoDB | Flexible schema for catalogs/themes |
| Cache / Queue | Redis | Sessions, refresh tokens, job queue coordination |
| Auth | JWT (access + refresh) + bcrypt | Self-managed, no vendor lock-in |
| Raw storage | S3 (private `raw` bucket; MinIO locally) | Same path as delivery storage; portable to cloud-only deployments |
| Delivery storage | S3 + CloudFront | Standard OTT delivery |
| Transcoding | FFmpeg on on-prem worker | Cheapest path; managed services can come later |
| Repo | pnpm + Turborepo monorepo | Shared types, single CI graph |

## 4. Monorepo Layout

```
kittotv/
├── apps/
│   ├── cms-api/              # Express + TS. Operator API. JWT, MongoDB, Redis. Lambda-portable.
│   ├── cms-dashboard/        # React + Vite. Operator UI.
│   ├── ott-api/              # Express + TS. Viewer-facing API. Separate deployable from cms-api.
│   ├── ott-web/              # React + Vite SPA. Whitelabel viewer site.
│   └── transcoder-worker/    # Node daemon. Polls jobs, runs FFmpeg, uploads to S3.
├── packages/
│   ├── shared-types/         # TS types: Tenant, Asset, TranscodeJob, TranscodeSettings, Theme, etc.
│   ├── ffmpeg-scripts/       # Pure FFmpeg argv builders (HLS ladder, DASH, posters, sprites).
│   └── sdk/                  # Typed clients (cms + ott). Used by dashboards + ott-web.
├── infra/                    # IaC stubs (S3 buckets, CloudFront), env templates.
├── turbo.json
├── pnpm-workspace.yaml
└── CLAUDE.md
```

## 5. Data Model (MongoDB)

- **`tenants`** — `_id`, `slug`, `customDomain?`, `name`, `plan`, `createdAt`.
- **`themes`** — `tenantId`, colors, fonts, `logoS3Key`, hero layout config, button styles.
- **`cms_users`** — `tenantId`, `email`, `passwordHash`, `role` (`owner` | `editor`).
- **`viewers`** — `tenantId`, `email`, `passwordHash`, `watchlistAssetIds[]`.
- **`assets`** — `tenantId`, `title`, `description`, `rawS3Key` (key in the private raw bucket), `status` (`uploaded` | `queued` | `transcoding` | `ready` | `failed`), `hlsManifestUrl?`, `dashManifestUrl?`, `posterUrls[]`, `durationSec?`, `createdAt`.
- **`transcode_jobs`** — `assetId`, `tenantId`, `status` (`pending` | `processing` | `done` | `failed`), `attempts`, `ffmpegProfile`, `workerId?`, `lastHeartbeatAt?`, `errorLog?` (truncated stderr tail), timestamps.

**Hard rule:** every collection above (except `tenants` itself) carries `tenantId`. See §10.

## 6. Transcoding Pipeline

**Upload flow.**
1. Dashboard requests an upload URL from CMS API.
2. CMS API returns a presigned PUT/multipart for the **raw S3 bucket** (`s3://kittotv-raw-<env>/<tenantId>/<assetId>/source.<ext>`).
3. Dashboard uploads the raw file directly to S3.
4. CMS API creates an `assets` doc (`status: uploaded`, `rawS3Key`) and enqueues a `transcode_jobs` doc (`status: pending`, with `TranscodeSettings`).

**Worker daemon.**
- Long-polls Redis (`BRPOPLPUSH pending processing`) **or** atomically claims jobs via Mongo `findOneAndUpdate({status:'pending'}, {$set:{status:'processing', workerId, lastHeartbeatAt: now}})`.
- Heartbeats every 15s. A supervisor re-queues jobs whose heartbeat is stale (> 60s).
- Downloads raw from the S3 raw bucket → local temp dir.

**FFmpeg.**
- HLS adaptive ladder: 240p / 360p / 480p / 720p / 1080p with per-rung bitrate caps.
- DASH variant set generated from the same renditions.
- Poster frames + thumbnail sprite sheet.
- Commands are built by **pure functions** in `packages/ffmpeg-scripts` that return `string[]` argv arrays — never shell-concatenated strings in the worker (see §10).

**Output.**
- Write segments + manifests to local temp.
- `aws s3 sync` to `s3://<delivery-bucket>/<tenantId>/<assetId>/hls/...` and `.../dash/...`.
- Update `assets` doc with `hlsManifestUrl`, `dashManifestUrl`, `durationSec`, posters.
- Mark `transcode_jobs.status = done`, `assets.status = ready`.

**Failure handling.**
- Capture last ~4KB of FFmpeg stderr into `transcode_jobs.errorLog`.
- Exponential backoff retries up to N (default 3). After N, `status: failed` and surface in dashboard.

## 7. Whitelabel / Theming

- **One OTT Web deploy** serves all tenants.
- On load, the SPA resolves the current tenant:
  1. Match `window.location.hostname` against `tenants.customDomain`.
  2. Else parse subdomain slug (`<slug>.kittotv.app`).
- Then it fetches `GET /public/theme/by-host/:host` from **OTT API** and applies the response as CSS variables + asset URLs.
- Catalog is fetched via `GET /public/catalog` from OTT API (tenant inferred from host).
- Viewer JWT carries `tenantId`; the API middleware enforces tenant scope on every authenticated request.

## 8. Auth Conventions

- **JWT**: short-lived access token (~15m) + refresh token (~30d).
- **bcrypt** cost 12 for all password hashes.
- **Separate signing secrets** per audience: `cms_user` and `viewer`. Tokens are not interchangeable.
- **Refresh tokens** stored in Redis under `refresh:<userId>:<sessionId>` — enables server-side revocation (logout-all, password change).
- Tokens include `{ sub, tenantId, aud, sessionId, iat, exp }`.
- Never log raw JWTs, password hashes, or presigned S3 URLs.

## 9. Lambda-Portability Rules

Both `cms-api` and `ott-api` are Express today but should be trivially portable to Lambda later.

- `apps/{cms-api,ott-api}/src/services/**` — pure business logic. **Must not** import `express`, `req`, or `res` types.
- `apps/{cms-api,ott-api}/src/controllers/**` — thin adapters. Read inputs from `req`, call a service with a `Context` object, write `res`.
- `Context` shape: `{ userId, tenantId, role, db, redis, logger }`. Built once per request by middleware.
- For Lambda, swap the controller layer for handlers that build the same `Context` from API Gateway events. Services stay byte-for-byte the same.
- No global mutable state, no `app.locals`. Dependencies (db, redis, S3 client) are injected.

## 10. Conventions for Claude

- **TypeScript strict mode everywhere.** No `any` without a written reason.
- **Multi-tenancy isolation is non-negotiable.** Every query that touches tenant data MUST filter by `tenantId`. Add a repository layer where `tenantId` is the **first** argument. **Never** read `tenantId` from a request body — it comes from the verified JWT, period.
- **FFmpeg commands** live in `packages/ffmpeg-scripts` as argv-array builders. Do not shell-concat in the worker. Do not `exec(string)` — use `spawn(cmd, argv)`.
- **Presigned S3 URLs** for both upload and download. Never proxy video bytes through the API.
- **OTT Web must remain pure SPA** (no SSR). It has to be fully CloudFront-cacheable for the shell; only API calls hit origin.
- **Never log** raw JWTs, password hashes, presigned URLs, or full S3 object contents.
- **Don't add dependencies casually.** Prefer the standard library and what's already in the workspace.

## 11. Local Dev

- `docker-compose.yml` brings up MongoDB, Redis, and MinIO (S3-compatible).
- `pnpm install` at repo root.
- `pnpm dev` runs all apps via Turbo (`cms-api`, `cms-dashboard`, `ott-api`, `ott-web`, `transcoder-worker`).
- MinIO is initialized with two buckets: `kittotv-raw-dev` and `kittotv-delivery-dev`.
- Worker is configured against MinIO locally; same code path as production S3.
- Seed script provisions one demo tenant + a sample asset.

## 12. Verification

For any change before declaring done:

- `pnpm -w typecheck` passes.
- `pnpm -w test` passes.
- **Transcoding changes:** run worker against a sample MP4. Confirm `master.m3u8` and `manifest.mpd` are produced, uploaded to S3 (or MinIO), and play in `hls.js` + dash.js reference players.
- **Player theme changes:** load the player against at least two distinct tenant slugs and confirm visual swap (colors, logo, hero) without rebuild.
- **API changes:** test multi-tenant isolation explicitly — a token for tenant A must not read/write tenant B's data.
