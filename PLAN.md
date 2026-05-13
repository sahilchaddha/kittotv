# KittoTV — Build Plan

This is the master plan for building the KittoTV whitelabel OTT platform described in [`CLAUDE.md`](./CLAUDE.md). It declares the build order and links to focused per-component plans in [`plans/`](./plans/).

## Goal (v1)

Ship an end-to-end pipeline where:
1. A tenant operator signs up via the CMS Dashboard.
2. Uploads a raw MP4 with chosen transcode settings.
3. The transcoder produces playable HLS + DASH on S3 (or MinIO locally).
4. An end viewer browses the tenant's whitelabel OTT site and streams it.

## Build Order

Each phase delivers a vertical slice that can be demoed before the next phase starts. **Do not start a phase before the previous one is "done" per its plan's Verification section.**

| Phase | Component | Plan | Why this order |
|---|---|---|---|
| 0 | Shared foundations | [`plans/shared-foundations.md`](./plans/shared-foundations.md) | Monorepo, shared types, docker-compose. Everything else depends on these. |
| 1 | Transcoding pipeline (PRIORITY) | [`plans/transcoding-pipeline.md`](./plans/transcoding-pipeline.md) | Validates the storage + queue + FFmpeg chain end-to-end before any UI is written. Risk lives here. |
| 2 | CMS API | [`plans/cms-api.md`](./plans/cms-api.md) | Replaces the stub job-enqueue used in Phase 1 with full tenant/asset/job CRUD. |
| 3 | CMS Dashboard | [`plans/cms-dashboard.md`](./plans/cms-dashboard.md) | Operator UI on top of CMS API. |
| 4 | OTT API | [`plans/ott-api.md`](./plans/ott-api.md) | Viewer-facing API: auth, catalog, signed playback URLs. |
| 5 | OTT Web | [`plans/ott-web.md`](./plans/ott-web.md) | Whitelabel SPA, theme resolution, HLS/DASH playback. |

## Cross-cutting Principles

These apply to every phase — see [`CLAUDE.md`](./CLAUDE.md) §9, §10 for full text:

- TypeScript strict everywhere.
- Multi-tenant isolation enforced at the repository layer; `tenantId` always comes from the verified JWT or host resolution, never the request body.
- Business logic in `src/services/**` stays Express-free for Lambda portability.
- FFmpeg commands live only in `packages/ffmpeg-scripts` as argv-array builders.
- Presigned S3 URLs for raw upload and playback delivery. No byte-proxying.

## Status Tracking

Each plan has a Milestones checklist. Mark items in-place as work lands. Once a plan's Verification section passes, mark the phase "done" in the table above.
