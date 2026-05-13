# Phase 5 — OTT Web (Whitelabel SPA)

## Goal

Build the Netflix-style whitelabel viewer SPA. One deploy, many tenants resolved by hostname. "Done" = a viewer visits a tenant's custom domain, sees the tenant's branding applied instantly, signs up, browses the catalog, adds to watchlist, and streams a title via HLS (with DASH fallback) using a signed playback ticket from OTT API.

## Inputs / Outputs

- **Consumes:** OTT API via `packages/sdk`'s `createOttClient()`.
- **Produces:** Static SPA bundle deployed to S3 + CloudFront. CloudFront's default behavior maps the same bundle to every tenant hostname; the SPA does runtime tenant resolution.

## Tech & Dependencies

- React + Vite + TypeScript (`apps/ott-web`)
- `packages/sdk` (ott client)
- `packages/shared-types`
- Router: `react-router-dom`
- State: TanStack Query
- Playback: `hls.js` primary, `dash.js` fallback (or `shaka-player` if both formats are desired in one engine — decide in M3).
- Styling: CSS variables driven by theme response (so per-tenant restyle is purely runtime CSS, no rebuild).

## Data Model Touch Points

None directly — talks only to OTT API.

## Boot Sequence

1. SPA mounts; reads `window.location.host`.
2. Fetches `GET /public/theme/by-host/:host`.
3. Applies returned theme as `:root` CSS variables (`--brand-primary`, `--font-display`, …) and sets `<link rel="icon">`, `<title>`, hero assets.
4. Fetches `GET /public/catalog` to render the browse view.
5. Anonymous browse allowed. Login modal triggered on watchlist/play actions.

## Screens

- **Browse / Home** — hero, rows of categories (Netflix-style horizontal scrollers).
- **Title Detail** — poster, description, episode list (if series — out of v1, document as v2), Play + Add to Watchlist buttons.
- **Player** — fullscreen playback using `hls.js`. Subtitle + audio track selection from manifest. Watermark is baked in by the transcoder, not the player.
- **Auth Modal** — signup / login (viewer JWT).
- **My List** — watchlist.
- **Account** — email, change password, logout.

## Whitelabel Mechanics

- All visual choices come from the theme response (colors, fonts, logo, hero treatment, button radius).
- The SPA shell is **fully CloudFront-cacheable**; only API calls hit origin.
- No SSR. Initial paint shows a neutral splash; brand swaps in as soon as theme resolves (target < 200ms on warm cache).

## Milestones

- [ ] **M1** Vite + React boot. Tenant resolution + theme application before first paint.
- [ ] **M2** Browse view with rows from `/public/catalog`.
- [ ] **M3** Player route with `hls.js` integration; pick DASH engine.
- [ ] **M4** Viewer auth modal + token persistence.
- [ ] **M5** Watchlist + My List.
- [ ] **M6** Signed-ticket flow: on Play, request `/playback/:assetId/ticket`, hand `hlsUrl` to `hls.js`.
- [ ] **M7** Loading + error states (asset not ready, ticket expired, network).
- [ ] **M8** Two-tenant visual smoke: same bundle, two hosts, two completely different brands.

## Open Questions

- Series / seasons support — defer to v2; document the catalog response to leave room for it.
- Offline downloads — out of v1.
- Resume-position tracking — recommend a `playback_sessions` heartbeat from the player every 15s; out of v1 unless trivial.

## Verification

- Bring up the full stack (`cms-api`, `ott-api`, `cms-dashboard`, `ott-web`, `transcoder-worker`, docker-compose deps).
- In CMS Dashboard: create tenant T1 with theme A and asset X (transcoded to `ready`); create tenant T2 with theme B and asset Y.
- In OTT Web: visit `t1.localhost:5173` and `t2.localhost:5173` (or equivalent host-based routing). Confirm each shows its own brand and its own catalog only.
- Play asset X as a logged-in viewer on T1 — HLS playback works, subtitles selectable.
- Confirm a viewer logged in on T1 cannot fetch T2's playback ticket.
