# Phase 3 — CMS Dashboard

## Goal

Build the operator UI on top of the Phase 2 CMS API. "Done" = an operator can sign up, log in, configure their theme with live preview, upload a raw video with chosen transcode settings, monitor the job to completion, and preview the resulting HLS playback inside the dashboard.

## Inputs / Outputs

- **Consumes:** CMS API (via `packages/sdk`'s `createCmsClient()`).
- **Produces:** No persistent backend data of its own; pure React SPA bundled to static assets.

## Tech & Dependencies

- React + Vite + TypeScript (`apps/cms-dashboard`)
- `packages/sdk` (cms client)
- `packages/shared-types`
- Router: `react-router-dom`
- State: TanStack Query for server state; local component state otherwise (no Redux).
- Form: `react-hook-form` + `zod` (reuse zod schemas from cms-api via shared package or re-export).
- Styling: Tailwind CSS (or CSS Modules — decide in Phase 0).
- Playback preview: `hls.js` for previewing transcoded output inside the dashboard.

## Data Model Touch Points

None directly — talks only to CMS API.

## Screens

- **Auth** — Signup, Login, Forgot Password (stub for v1).
- **Tenant Settings** — name, custom domain field, plan.
- **Theme Editor** — color pickers, logo upload (to delivery bucket via a separate "asset" subtype), hero layout, button styles. **Live preview pane** rendering a mini OTT-Web shell with the in-progress theme applied.
- **Asset List** — table with title, duration, status, last updated. Filters: status, search.
- **Asset Upload** — form: title, description, raw file picker (multipart upload to presigned URL with progress), `TranscodeSettings` form (ladder preset radio, formats checkboxes, audio + subtitle sidecar uploaders, watermark uploader + position picker, DRM dropdown (disabled for v1 except `none`), thumbnail interval).
- **Asset Detail** — metadata, transcode job status timeline, retry button, embedded `hls.js` preview of `master.m3u8` once `ready`.
- **Job Monitor** — global table of recent `transcode_jobs` with status, attempts, errorLog tail.

## Milestones

- [ ] **M1** Vite + React + Tailwind boot. Router with placeholder pages.
- [ ] **M2** Auth flow (signup / login / logout) using sdk; token persisted in memory + refresh on mount.
- [ ] **M3** Tenant Settings page.
- [ ] **M4** Theme Editor with live preview (preview iframe loads the OTT-Web shell pointed at a draft theme).
- [ ] **M5** Asset List + Asset Detail.
- [ ] **M6** Asset Upload with multipart progress + transcode settings form.
- [ ] **M7** Job Monitor.
- [ ] **M8** Polling / SSE for job status updates on Asset Detail.

## Open Questions

- Component library — build from scratch with Tailwind, or use `shadcn/ui`? (Recommend `shadcn/ui` for speed.)
- How to render the live theme preview without a network round-trip per keystroke (debounce + postMessage to iframe).

## Verification

- Run `pnpm --filter cms-api dev` and `pnpm --filter cms-dashboard dev` together.
- Full operator journey in browser: signup → set theme → upload a sample MP4 with FHD + HLS+DASH → watch job go `pending → processing → ready` → play it in the embedded preview.
- Multi-tenant smoke: create two tenants in two browsers; confirm they cannot see each other's assets.
