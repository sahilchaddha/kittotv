# Phase 1 — Transcoding Pipeline (PRIORITY)

## Goal

Build the on-prem FFmpeg-based transcoder worker plus the `packages/ffmpeg-scripts` argv builders that drive it. "Done" = given a `transcode_jobs` document in Mongo pointing at a raw MP4 in the `kittotv-raw-dev` MinIO bucket and a `TranscodeSettings` payload, the worker downloads the raw, produces HLS + DASH outputs per settings, uploads them to `kittotv-delivery-dev`, and marks the job + asset `ready` — with the manifests actually playing in reference players.

This phase ships **before** any UI. A minimal CLI / HTTP stub is enough to enqueue test jobs.

## Inputs / Outputs

- **Consumes:**
  - `transcode_jobs` documents (`status: pending`) with embedded `TranscodeSettings`.
  - Raw video files at `s3://kittotv-raw-dev/<tenantId>/<assetId>/source.<ext>`.
  - Optional sidecar audio + subtitle objects in the same prefix (per `TranscodeSettings.audioTracks`, `.subtitles`).
- **Produces:**
  - HLS output at `s3://kittotv-delivery-dev/<tenantId>/<assetId>/hls/master.m3u8` + variant playlists + segments.
  - DASH output at `.../dash/manifest.mpd` + segments.
  - Thumbnail sprite + poster frames at `.../thumbs/`.
  - Updated `assets` doc with `hlsManifestUrl`, `dashManifestUrl`, `posterUrls[]`, `durationSec`, `status: ready`.
  - Updated `transcode_jobs` doc with terminal status, attempts, heartbeat history, and (on failure) `errorLog`.

## Tech & Dependencies

- Node.js + TypeScript (`apps/transcoder-worker`)
- `packages/ffmpeg-scripts` (argv builders, pure functions, unit-tested)
- `packages/shared-types` for `TranscodeJob`, `TranscodeSettings`, `Asset`
- `mongodb` driver
- `@aws-sdk/client-s3` + `@aws-sdk/lib-storage` (works against MinIO via endpoint override)
- Local FFmpeg binary (assumed installed on worker host; `engines.ffmpeg: ">=6"` documented)
- `pino` logger

## Data Model Touch Points

- **Reads / writes `transcode_jobs`** (claim, heartbeat, terminal status, errorLog).
- **Writes `assets`** (manifest URLs, durationSec, posterUrls, status).
- Does **not** read tenant/user collections.

## TranscodeSettings (canonical schema, defined in `packages/shared-types`)

```ts
export type LadderPreset = 'SD' | 'HD' | 'FHD' | '4K' | 'custom';

export interface TranscodeSettings {
  ladder: LadderPreset;
  customRungs?: Array<{ height: number; bitrateKbps: number; maxrateKbps?: number }>;
  formats: { hls: boolean; dash: boolean };
  audioTracks: Array<{ s3Key: string; language: string; default?: boolean }>;
  subtitles: Array<{ s3Key: string; language: string; format: 'srt' | 'vtt' }>;
  watermark?: { s3Key: string; position: 'tl' | 'tr' | 'bl' | 'br'; opacity: number };
  drm?: { provider: 'none' | 'widevine-stub' | 'fairplay-stub'; keyHint?: string };
  thumbnails?: { spriteIntervalSec: number };
}
```

Ladder presets resolve to:
- **SD**: 360p, 480p
- **HD**: 480p, 720p
- **FHD**: 480p, 720p, 1080p
- **4K**: 720p, 1080p, 1440p, 2160p

## Worker Loop

1. Boot: connect Mongo + S3, register `workerId`, start heartbeat.
2. Claim: `findOneAndUpdate({ status: 'pending' }, { $set: { status: 'processing', workerId, lastHeartbeatAt: now, startedAt: now } })`.
3. If no job: backoff 5s, loop.
4. Resolve `assets` doc → `rawS3Key` → download to `/tmp/<jobId>/source`.
5. Download any sidecar audio + subtitle objects.
6. Build argv via `packages/ffmpeg-scripts` per settings; `spawn(ffmpeg, argv)`.
7. Heartbeat every 15s while ffmpeg runs.
8. On success: upload `/tmp/<jobId>/out/**` to delivery bucket; update asset + job; cleanup temp.
9. On failure: capture last 4KB of stderr → `transcode_jobs.errorLog`; increment `attempts`; either re-queue with backoff or mark `failed` after N attempts.

A separate **supervisor loop** scans for `processing` jobs whose `lastHeartbeatAt < now - 60s` and resets them to `pending`.

## Milestones

- [ ] **M1** Worker boots, connects Mongo + S3 (MinIO), heartbeats.
- [ ] **M2** Job claim (atomic findOneAndUpdate) works under concurrency (simulate with two worker processes).
- [ ] **M3** Download raw from S3 to local temp.
- [ ] **M4** `buildHlsLadder()` returns valid argv for an HD ladder; worker `spawn`s ffmpeg; `master.m3u8` lands in temp.
- [ ] **M5** Add DASH packaging path (`buildDash()`).
- [ ] **M6** Multi-audio mux (additional `-i` inputs + `-map` flags per `audioTracks`).
- [ ] **M7** Subtitle ingest: SRT → VTT conversion, packaged into HLS (`#EXT-X-MEDIA TYPE=SUBTITLES`) and DASH (`<AdaptationSet contentType="text">`).
- [ ] **M8** Watermark overlay (`overlay` filter, position + opacity).
- [ ] **M9** Thumbnail sprite generation (`buildPosterSprite()`).
- [ ] **M10** DRM packaging **stub**: when `drm.provider !== 'none'`, emit `ContentProtection` placeholders in the manifest and document where real key-server integration will hook in. No actual encryption keys in v1.
- [ ] **M11** Upload outputs to delivery bucket with tenant-prefixed keys; update `assets` doc.
- [ ] **M12** Retry policy + supervisor for stale `processing` jobs.
- [ ] **M13** A tiny CLI (`pnpm --filter transcoder-worker run enqueue -- --raw-key=... --settings=path.json`) for manual end-to-end testing before CMS API exists.

## Open Questions

- Will any tenants need per-rendition HDR / 10-bit? (Defer; document as out of v1.)
- Segment duration default — 6s? (Recommend 6s for HLS, matching CMAF-friendly settings for DASH.)
- DRM real integration target (Widevine via Shaka Packager, FairPlay via separate flow) — out of v1, but `drm` field must already exist in settings so it can be added without a schema change.

## Verification

- Run worker against a sample 1080p MP4 with `{ ladder: 'FHD', formats: { hls: true, dash: true }, audioTracks: [stereo], subtitles: [en-vtt], watermark: { ... }, thumbnails: { spriteIntervalSec: 10 } }`.
- Confirm in MinIO browser: `kittotv-delivery-dev/<tenantId>/<assetId>/hls/master.m3u8`, `/dash/manifest.mpd`, `/thumbs/sprite.jpg`.
- Load `master.m3u8` in the [hls.js demo player](https://hls-js.netlify.app/demo/) — playback works, subtitle track selectable, watermark visible.
- Load `manifest.mpd` in the [dash.js reference player](https://reference.dashif.org/dash.js/) — same checks.
- Pull a "fail" case (corrupt input) and confirm `transcode_jobs.errorLog` captures the FFmpeg stderr tail and `status` becomes `failed` after N attempts.
- Kill a worker mid-job; confirm supervisor re-queues within 60–90s.
- `pnpm --filter ffmpeg-scripts test` passes (unit tests on argv builders — deterministic strings in, deterministic argv out).
