# Phase 0 — Shared Foundations

## Goal

Stand up the monorepo skeleton and shared infrastructure that every other component depends on. "Done" = `pnpm install && pnpm -w typecheck` succeeds on an empty workspace graph, and `docker compose up` brings up MongoDB + Redis + MinIO with the two buckets created.

## Inputs / Outputs

- **Produces:**
  - Monorepo skeleton with `apps/*` and `packages/*` stubs.
  - `packages/shared-types` with foundational types.
  - `packages/sdk` with audience-aware fetch wrapper.
  - `packages/ffmpeg-scripts` skeleton.
  - `docker-compose.yml` + bucket-init script.
  - `infra/` directory placeholder for future IaC.

## Tech & Dependencies

- pnpm workspaces + Turborepo
- TypeScript (strict)
- ESLint + Prettier
- Vitest for unit testing across packages
- docker-compose: `mongo:7`, `redis:7`, `minio/minio:latest`, `minio/mc` for bucket init

## Data Model Touch Points

Defines TS types only. No DB writes yet.

## Milestones

- [ ] `pnpm-workspace.yaml`, `turbo.json`, root `package.json`, root `tsconfig.base.json`.
- [ ] `apps/cms-api`, `apps/cms-dashboard`, `apps/ott-api`, `apps/ott-web`, `apps/transcoder-worker` as empty TS packages with their own `tsconfig.json` extending the base.
- [ ] `packages/shared-types` with `Tenant`, `Theme`, `CmsUser`, `Viewer`, `Asset`, `TranscodeJob`, `TranscodeSettings`, `Context` types from CLAUDE.md §5 + Phase 1 settings schema.
- [ ] `packages/sdk` with `createCmsClient()` and `createOttClient()` factories. Token storage is injectable (memory by default).
- [ ] `packages/ffmpeg-scripts` skeleton exporting `buildHlsLadder()`, `buildDash()`, `buildPosterSprite()` placeholders returning `string[]`.
- [ ] `docker-compose.yml` with mongo, redis, minio, and an `mc` one-shot service that creates `kittotv-raw-dev` and `kittotv-delivery-dev`.
- [ ] Shared ESLint + Prettier config.
- [ ] Root scripts: `pnpm dev`, `pnpm -w typecheck`, `pnpm -w test`, `pnpm -w lint`.

## Open Questions

- Node version? (Recommend Node 22 LTS in `.nvmrc` and `engines`.)
- Use `tsx` or `tsup` for dev/build of Node services? (Recommend `tsx` for dev, `tsup` for build.)

## Verification

- `pnpm install` succeeds.
- `pnpm -w typecheck` passes with all stub packages.
- `docker compose up -d` succeeds; `mc ls local/` shows both buckets.
- `pnpm -w test` runs (even if zero tests yet) without error.
