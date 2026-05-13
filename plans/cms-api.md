# Phase 2 — CMS API

## Goal

Build the operator-facing HTTP API behind the CMS Dashboard. "Done" = a tenant can sign up, log in, request a presigned raw-upload URL, create an asset + transcode job (consumed by the Phase 1 worker), monitor job status, and CRUD theme + catalog metadata — all under strict multi-tenant isolation.

## Inputs / Outputs

- **Consumes:** HTTPS requests from `cms-dashboard`; JWTs minted by itself.
- **Produces:**
  - `tenants`, `cms_users`, `themes`, `assets`, `transcode_jobs` documents.
  - Presigned S3 PUT/multipart URLs for the raw bucket.
  - JWT access + refresh tokens (audience: `cms_user`).

## Tech & Dependencies

- Express + TypeScript (`apps/cms-api`)
- Lambda-portable layering (controllers thin, services pure; CLAUDE.md §9)
- `mongodb` driver
- `ioredis` (refresh-token store, rate-limit counters)
- `bcrypt` (cost 12)
- `jsonwebtoken`
- `@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`
- `zod` for input validation at controller boundary
- `pino` for logging
- `packages/shared-types`, `packages/sdk` (consumed by dashboard)

## Data Model Touch Points

Owns and writes:
- `tenants`, `themes`, `cms_users`, `assets`, `transcode_jobs`.

Does not touch:
- `viewers`, `watchlists`, `playback_sessions` (owned by ott-api).

## Endpoint Surface (v1)

- `POST /auth/signup` — creates `tenants` + first `cms_users` owner.
- `POST /auth/login` — returns access + refresh tokens.
- `POST /auth/refresh`, `POST /auth/logout`.
- `GET /me`, `GET /tenant`, `PATCH /tenant`.
- `GET /theme`, `PUT /theme`.
- `POST /assets` → returns `{ assetId, uploadUrl }` (presigned PUT to raw bucket).
- `POST /assets/:id/complete` — call after upload finishes; creates `transcode_jobs` (with `TranscodeSettings` from request body, validated by zod).
- `GET /assets`, `GET /assets/:id`, `PATCH /assets/:id`, `DELETE /assets/:id`.
- `GET /jobs`, `GET /jobs/:id`, `POST /jobs/:id/retry`.

## Layering

```
src/
├── index.ts              # boots express, mounts routers
├── controllers/          # Express adapters. Validate with zod, build Context, call service.
├── services/             # Pure business logic. Imports no express.
├── repositories/         # Mongo access. Every method takes (tenantId, ...).
├── middleware/           # auth, requestContext, errorHandler, rateLimit
├── lib/                  # s3, jwt, bcrypt wrappers
└── types/                # request/response schemas (zod) re-exported as TS types
```

`Context` object built per request: `{ userId, tenantId, role, db, redis, s3, logger }`.

## Milestones

- [ ] **M1** Express boot + health endpoint + structured logging + error handler.
- [ ] **M2** Mongo + Redis + S3 clients wired and injected via `Context`.
- [ ] **M3** Signup + login + JWT (access + refresh) with Redis-backed refresh store.
- [ ] **M4** `requireAuth` middleware extracting `tenantId` from token; **never** from body.
- [ ] **M5** Repository layer with `tenantId` as first arg; integration test that asserts tenant A's token cannot read tenant B's documents.
- [ ] **M6** Tenant + theme CRUD.
- [ ] **M7** Asset create + presigned PUT URL issuance to raw bucket.
- [ ] **M8** `POST /assets/:id/complete` enqueues `transcode_jobs` with validated `TranscodeSettings`.
- [ ] **M9** Job list + detail + retry endpoints.
- [ ] **M10** Per-IP and per-user rate limiting on auth endpoints (Redis token bucket).
- [ ] **M11** OpenAPI doc generated from zod schemas (e.g. via `zod-to-openapi`) so dashboard + sdk stay in sync.

## Open Questions

- Multipart upload threshold (single PUT for <100MB, multipart above)?
- Theme schema versioning (add `themeVersion` field for future migrations)?
- Soft delete vs hard delete for assets (recommend soft delete + S3 lifecycle rules).

## Verification

- `pnpm --filter cms-api test` — unit tests on services + integration tests against a docker-compose Mongo.
- Manual E2E using `curl` or the `sdk`:
  1. Signup → login → get token.
  2. Create asset → `curl -T sample.mp4 <presigned URL>`.
  3. Complete asset → confirm a `transcode_jobs` doc with `status: pending` exists.
  4. Phase 1 worker picks it up and transitions it to `ready` (validates the cross-component contract).
- Tenant isolation test: with two tenants A and B, A's token returns 404 on B's `assetId`.
